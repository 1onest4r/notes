#include <Wire.h>
#include <SPI.h>
#include <SD.h>
#include <TinyGPS++.h>
#include <LoRa.h>
#include "DHT.h"

// --- ESP32 HARDWARE HEADERS TO DISABLE BROWNOUT ---
#include "soc/soc.h"
#include "soc/rtc_cntl_reg.h"

// Custom HSPI Bus Pins for SD Card (No shared wires with LoRa!)
#define SD_SCK  14
#define SD_MISO 25
#define SD_MOSI 13
#define SD_CS   15

// Default VSPI Pins for LoRa
#define LORA_CS   27
#define LORA_RST  32 
#define LORA_DIO0 26

const int MPU_ADDR = 0x68;
#define DHTPIN 4     
#define DHTTYPE DHT11 

// --- MATCH THIS BAND WITH YOUR GROUND STATION ---
// 433E6 (433 MHz) | 868E6 (868 MHz) | 915E6 (915 MHz)
#define BAND 433E6 

// Non-blocking timer variables
unsigned long lastLogTime = 0;
const unsigned long logInterval = 250; // Log/Transmit every 250ms

// Sensor & Radio definitions
HardwareSerial gpsSerial(2); 
TinyGPSPlus gps;
DHT dht(DHTPIN, DHTTYPE);

// Create a custom SPI interface specifically for the SD card (HSPI)
SPIClass hspi(HSPI);

// Raw IMU variables
int16_t raw_ax, raw_ay, raw_az;
int16_t raw_temp;
int16_t raw_gx, raw_gy, raw_gz;

String fileName; // Unique CSV file

// Convert milliseconds to readable HH:MM:SS
String getReadableTime(unsigned long ms) {
  unsigned long seconds = ms / 1000;
  unsigned long minutes = seconds / 60;
  unsigned long hours = minutes / 60;
  
  seconds %= 60;
  minutes %= 60;
  
  char buffer[15];
  sprintf(buffer, "%02lu:%02lu:%02lu", hours, minutes, seconds);
  return String(buffer);
}

void setup() {
  // 1. DISABLE BROWNOUT DETECTOR ON BOOT
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0); 
  
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n===============================================");
  Serial.println("  ROCKET PAYLOAD TRANSMITTER ACTIVE (DUAL SPI) ");
  Serial.println("===============================================");

  // 2. Initialize Custom HSPI Bus for the SD Card
  hspi.begin(SD_SCK, SD_MISO, SD_MOSI, SD_CS);
  
  if (!SD.begin(SD_CS, hspi)) {
    Serial.println("[-] ERROR: SD card initialization failed!");
    while(1) { delay(10); }
  }
  Serial.println("[+] SUCCESS: SD card online (HSPI).");

  // Create unique filename
  int testCount = 1;
  while (true) {
    fileName = "/flight_log_" + String(testCount) + ".csv";
    if (!SD.exists(fileName.c_str())) {
      break; 
    }
    testCount++;
  }
  Serial.print("[+] Target flight log created: "); Serial.println(fileName);

  // Write CSV header
  File dataFile = SD.open(fileName.c_str(), FILE_APPEND); 
  if (dataFile) {
    dataFile.println("Uptime(ms),Uptime_Clock,Accel_X(G),Accel_Y(G),Accel_Z(G),Gyro_X(deg/s),Gyro_Y(deg/s),Gyro_Z(deg/s),GPS_Lat,GPS_Lng,GPS_Alt(m),Sats,Temp_C,Humidity(%)");
    dataFile.close();
  }

  // 3. Initialize LoRa (Uses default VSPI bus automatically)
  LoRa.setPins(LORA_CS, LORA_RST, LORA_DIO0);
  Serial.print("[*] Initializing LoRa Radio at "); Serial.print(BAND / 1000000); Serial.println(" MHz...");
  if (!LoRa.begin(BAND)) {
    Serial.println("[-] ERROR: Starting LoRa failed!");
    while(1) { delay(10); }
  }
  
  // Telemetry optimization settings (Must match ground station!)
  LoRa.setSpreadingFactor(8); 
  LoRa.setSignalBandwidth(125E3);
  LoRa.setCodingRate4(5);
  Serial.println("[+] SUCCESS: LoRa Radio online (VSPI).");

  // 4. Initialize and Configure MPU6500 (Raw I2C)
  Wire.begin();
  Wire.beginTransmission(MPU_ADDR); Wire.write(0x6B); Wire.write(0x00);
  if (Wire.endTransmission() != 0) {
    Serial.println("[-] ERROR: MPU6500 not responding!");
    while(1) { delay(10); }
  }
  Wire.beginTransmission(MPU_ADDR); Wire.write(0x1C); Wire.write(0x18); Wire.endTransmission();
  Wire.beginTransmission(MPU_ADDR); Wire.write(0x1B); Wire.write(0x18); Wire.endTransmission();
  Serial.println("[+] SUCCESS: MPU6500 configured (16G, 2000 DPS).");

  // 5. Initialize Sensors (Power BN-180 with 3.3V to keep it cool!)
  gpsSerial.begin(9600, SERIAL_8N1, 16, 17);
  dht.begin();
  Serial.println("[+] SUCCESS: GPS and DHT11 online.");
  Serial.println("\n---> TRANSMITTER ACTIVE. BROADCASTING... <---\n");
}

void loop() {
  // Feed GPS continuously
  while (gpsSerial.available() > 0) {
    gps.encode(gpsSerial.read());
  }

  // Check if 250ms has passed
  if (millis() - lastLogTime >= logInterval) {
    lastLogTime = millis();
    logAndTransmitData();
  }
}

void logAndTransmitData() {
  unsigned long currentTime = millis();
  String uptimeClock = getReadableTime(currentTime);

  // 1. Read Raw IMU Data
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B);
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_ADDR, 14, true);

  raw_ax = Wire.read() << 8 | Wire.read();
  raw_ay = Wire.read() << 8 | Wire.read();
  raw_az = Wire.read() << 8 | Wire.read();
  raw_temp = Wire.read() << 8 | Wire.read();
  raw_gx = Wire.read() << 8 | Wire.read();
  raw_gy = Wire.read() << 8 | Wire.read();
  raw_gz = Wire.read() << 8 | Wire.read();

  // Convert raw IMU to G-forces and DPS
  float gX = (float)raw_ax / 2048.0;
  float gY = (float)raw_ay / 2048.0;
  float gZ = (float)raw_az / 2048.0;
  float degX = (float)raw_gx / 16.4;
  float degY = (float)raw_gy / 16.4;
  float degZ = (float)raw_gz / 16.4;

  // 2. Fetch GPS Variables
  String latStr = "0.000000";
  String lngStr = "0.000000";
  String altStr = "0.0";
  String satsStr = "0";

  if (gps.location.isValid()) {
    latStr = String(gps.location.lat(), 6);
    lngStr = String(gps.location.lng(), 6);
  }
  if (gps.altitude.isValid()) {
    altStr = String(gps.altitude.meters(), 1);
  }
  satsStr = String(gps.satellites.value());

  // 3. Read DHT11
  float h = dht.readHumidity();
  float t = dht.readTemperature();
  if (isnan(h) || isnan(t)) { h = 0.0; t = 0.0; }

  // 4. Create the CSV Telemetry String
  String csvString = String(currentTime) + "," +
                     uptimeClock + "," +
                     String(gX, 2) + "," + String(gY, 2) + "," + String(gZ, 2) + "," +
                     String(degX, 1) + "," + String(degY, 1) + "," + String(degZ, 1) + "," +
                     latStr + "," + lngStr + "," + altStr + "," + satsStr + "," +
                     String(t, 1) + "," + String(h, 0);

  // 5. Save locally to SD Card
  File dataFile = SD.open(fileName.c_str(), FILE_APPEND);
  if (dataFile) {
    dataFile.println(csvString);
    dataFile.close(); 
  } else {
    Serial.println("[-] WARNING: SD write failed.");
  }

  // 6. Broadcast over LoRa!
  LoRa.beginPacket();
  LoRa.print(csvString);
  LoRa.endPacket();

  // Print local dashboard so you can monitor transmission
  Serial.print("[TX] "); Serial.print(uptimeClock);
  Serial.print(" | GPS Sats: "); Serial.print(satsStr);
  Serial.print(" | Temp: "); Serial.print(t, 1); Serial.print("C");
  Serial.println(" | Telemetry Packet Broadcasted!");
}