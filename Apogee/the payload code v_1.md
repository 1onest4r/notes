#include <Wire.h>

#include <SPI.h>

#include <SD.h>

#include <TinyGPS++.h>

  

// Pin definitions

const int CS_PIN = 5;

const int MPU_ADDR = 0x68;

  

// Non-blocking timer variables

unsigned long lastLogTime = 0;

const unsigned long logInterval = 250; // Changed to 250ms (4 times a second)

  

// GPS definitions (using HardwareSerial port 2)

HardwareSerial gpsSerial(2);

TinyGPSPlus gps;

  

// Raw IMU variables

int16_t raw_ax, raw_ay, raw_az;

int16_t raw_temp;

int16_t raw_gx, raw_gy, raw_gz;

  

// Global string to store the unique filename

String fileName;

  

void setup() {

Serial.begin(115200);

delay(1000);

Serial.println("\n===============================================");

Serial.println(" ROCKET PAYLOAD FLIGHT COMPUTER ");

Serial.println("===============================================");

  

// 1. Initialize SD Card

if (!SD.begin(CS_PIN)) {

Serial.println("[-] ERROR: SD card initialization failed!");

while(1) { delay(10); }

}

Serial.println("[+] SUCCESS: SD card online.");

  

// Find a unique filename so we never overwrite older flight logs

int fileCount = 1;

while (true) {

fileName = "/flight_" + String(fileCount) + ".csv";

if (!SD.exists(fileName.c_str())) {

break; // Found an unused name!

}

fileCount++;

}

Serial.print("[+] Target log file created: ");

Serial.println(fileName);

  

// Write CSV header to the new file

File dataFile = SD.open(fileName.c_str(), FILE_APPEND);

if (dataFile) {

dataFile.println("Time(ms),Accel_X(G),Accel_Y(G),Accel_Z(G),Gyro_X(deg/s),Gyro_Y(deg/s),Gyro_Z(deg/s),GPS_Lat,GPS_Lng,GPS_Alt(m),Sats");

dataFile.close();

}

  

// 2. Initialize and Configure MPU6500 (Raw I2C)

Wire.begin();

Wire.beginTransmission(MPU_ADDR);

Wire.write(0x6B);

Wire.write(0x00); // Wake up

if (Wire.endTransmission() != 0) {

Serial.println("[-] ERROR: MPU6500 not responding! Check I2C wires.");

while(1) { delay(10); }

}

// Set Accel +/-16G and Gyro +/-2000 deg/s

Wire.beginTransmission(MPU_ADDR); Wire.write(0x1C); Wire.write(0x18); Wire.endTransmission();

Wire.beginTransmission(MPU_ADDR); Wire.write(0x1B); Wire.write(0x18); Wire.endTransmission();

Serial.println("[+] SUCCESS: MPU6500 configured (16G, 2000 DPS).");

  

// 3. Initialize GPS Serial (Baud rate is 9600 for BN-180, on Pins 16 & 17)

gpsSerial.begin(9600, SERIAL_8N1, 16, 17);

Serial.println("[+] SUCCESS: GPS Hardware Serial online.");

Serial.println("\n---> SYSTEM ACTIVE. STARTING LOGGING LOOP <---\n");

}

  

void loop() {

// Feed GPS data to the parser constantly in the loop

while (gpsSerial.available() > 0) {

gps.encode(gpsSerial.read());

}

  

// Check if 250ms has passed since our last log

if (millis() - lastLogTime >= logInterval) {

lastLogTime = millis();

logData();

}

}

  

void logData() {

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

  

// Convert raw IMU to physical units

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

  

// 3. Create the CSV formatted line for the SD card

String csvString = String(millis()) + "," +

String(gX, 2) + "," + String(gY, 2) + "," + String(gZ, 2) + "," +

String(degX, 1) + "," + String(degY, 1) + "," + String(degZ, 1) + "," +

latStr + "," + lngStr + "," + altStr + "," + satsStr;

  

// 4. PRINT DESCRIPTIVE, HUMAN-READABLE OUTPUT TO SERIAL MONITOR

Serial.print("[T: "); Serial.print(millis()); Serial.print("ms] ");

Serial.print("IMU Gs: (");

Serial.print(gX, 2); Serial.print(", ");

Serial.print(gY, 2); Serial.print(", ");

Serial.print(gZ, 2); Serial.print(") | ");

Serial.print("Gyro dps: (");

Serial.print(degX, 0); Serial.print(", ");

Serial.print(degY, 0); Serial.print(", ");

Serial.print(degZ, 0); Serial.print(") | ");

Serial.print("GPS: "); Serial.print(latStr); Serial.print(", "); Serial.print(lngStr);

Serial.print(" | Alt: "); Serial.print(altStr); Serial.print("m");

Serial.print(" | Sats: "); Serial.println(satsStr);

  

// 5. WRITE THE RAW CSV STRING TO SD CARD (and close it immediately to save!)

File dataFile = SD.open(fileName.c_str(), FILE_APPEND);

if (dataFile) {

dataFile.println(csvString);

dataFile.close(); // Saves the file data securely to the SD card

} else {

Serial.println("[-] WARNING: Failed to write to SD card!");

}

  

// Diagnostic warning if GPS isn't sending *any* serial characters

if (millis() > 5000 && gps.charsProcessed() < 10) {

Serial.println("[-] WARNING: No raw data from GPS module. Check TX/RX wires.");

}

}