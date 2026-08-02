
#include <SPI.h>
#include <LoRa.h>
#include "BluetoothSerial.h" // Built-in ESP32 Bluetooth Classic library

// --- ESP32 HARDWARE HEADERS TO DISABLE BROWNOUT ---
#include "soc/soc.h"
#include "soc/rtc_cntl_reg.h"

// Ground Station Default VSPI Pins for LoRa
#define ss 5
#define rst 14
#define dio0 26 

#define BAND 433E6 // Must match your transmitter band

// Watchdog Timer Variables
unsigned long lastPacketTime = 0;
const unsigned long watchdogTimeout = 30000; // 30 seconds
bool signalLostPrinted = false;

// Initialize Bluetooth
BluetoothSerial SerialBT;

// Helper function to split the received CSV string by commas
String getValue(String data, char separator, int index) {
  int found = 0;
  int strIndex[] = {0, -1};
  int maxIndex = data.length() - 1;

  for (int i = 0; i <= maxIndex && found <= index; i++) {
    if (data.charAt(i) == separator || i == maxIndex) {
      found++;
      strIndex[0] = strIndex[1] + 1;
      strIndex[1] = (i == maxIndex) ? i + 1 : i;
    }
  }
  return found > index ? data.substring(strIndex[0], strIndex[1]) : "";
}

// Function to print the beautiful dashboard to BOTH Serial and Bluetooth
void printDashboard(String receivedData, int rssiVal) {
  // Parse CSV variables
  String uptimeClock = getValue(receivedData, ',', 1);
  String gX           = getValue(receivedData, ',', 2);
  String gY           = getValue(receivedData, ',', 3);
  String gZ           = getValue(receivedData, ',', 4);
  String degX         = getValue(receivedData, ',', 5);
  String degY         = getValue(receivedData, ',', 6);
  String degZ         = getValue(receivedData, ',', 7);
  String latStr       = getValue(receivedData, ',', 8);
  String lngStr       = getValue(receivedData, ',', 9);
  String altStr       = getValue(receivedData, ',', 10);
  String satsStr      = getValue(receivedData, ',', 11);
  String tempStr      = getValue(receivedData, ',', 12);
  String humStr       = getValue(receivedData, ',', 13);

  // Format the dashboard
  String dash = "\n====================================================\n"
                "  TELEMETRY RECEIVED  |  Signal: " + String(rssiVal) + " dBm\n"
                "====================================================\n"
                "  [SYSTEM]  Uptime Clock: " + uptimeClock + "\n"
                "  [IMU]     Accel Gs:     (" + gX + ", " + gY + ", " + gZ + ")\n"
                "            Gyro dps:     (" + degX + ", " + degY + ", " + degZ + ")\n"
                "  [GPS]     Coordinates:  " + latStr + ", " + lngStr + "\n"
                "            Altitude:     " + altStr + "m  |  Sats: " + satsStr + "\n"
                "  [ENV]     Temperature:  " + tempStr + " C  |  Humidity: " + humStr + "%\n"
                "====================================================\n";

  // Print to Computer Serial Monitor
  Serial.print(dash);

  // Print to Phone via Bluetooth (if a phone is connected)
  if (SerialBT.hasClient()) {
    SerialBT.print(dash);
  }
}

void setup() {
  // Disable brownout detector immediately
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0); 
  
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n===============================================");
  Serial.println("     GROUND STATION TELEMETRY RECEIVER ACTIVE  ");
  Serial.println("===============================================");

  // Start Bluetooth Classic (Broadcasting as "Rocket_Ground_Station")
  SerialBT.begin("Rocket_Ground_Station");
  Serial.println("[+] SUCCESS: Bluetooth SPP online.");

  // Initialize LoRa
  LoRa.setPins(ss, rst, dio0);
  Serial.print("[*] Initializing LoRa Radio at "); Serial.print(BAND / 1000000); Serial.println(" MHz...");
  if (!LoRa.begin(BAND)) {
    Serial.println("[-] ERROR: Starting LoRa failed! Check wiring.");
    while (1) { delay(10); } 
  }

  // Match transmitter parameters
  LoRa.setSpreadingFactor(8);      
  LoRa.setSignalBandwidth(125E3);  
  LoRa.setCodingRate4(5);          

  Serial.println("[+] SUCCESS: LoRa Radio online.");
  Serial.println("\n---> WAITING FOR INCOMING TELEMETRY PACKETS <---\n");
  
  lastPacketTime = millis();
}

void loop() {
  // Check for incoming LoRa packets
  int packetSize = LoRa.parsePacket();
  
  if (packetSize) {
    String receivedData = "";
    while (LoRa.available()) {
      receivedData += (char)LoRa.read();
    }

    int rssiVal = LoRa.packetRssi();

    // Print dashboard to both Serial and Bluetooth
    printDashboard(receivedData, rssiVal);

    // Feed the watchdog
    lastPacketTime = millis();
    signalLostPrinted = false; // Reset the warning flag
  }

  // PASSIVE WATCHDOG: Just warning you instead of rebooting!
  if (millis() - lastPacketTime > watchdogTimeout) {
    if (!signalLostPrinted) {
      Serial.println("\n[!] WARNING: Signal Lost. No telemetry received for 30 seconds.");
      Serial.println("[!] Ground Station remains online, listening...\n");
      signalLostPrinted = true; // Print only once to prevent spamming
    }
  }
}