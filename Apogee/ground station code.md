#include <SPI.h>
#include <LoRa.h>

// --- ESP32 HARDWARE HEADERS TO DISABLE BROWNOUT ---
#include "soc/soc.h"
#include "soc/rtc_cntl_reg.h"

// Ground Station Default VSPI Pins for LoRa (Kept exactly as you had them)
#define ss 5
#define rst 14
#define dio0 26 

#define BAND 433E6 // Must match your transmitter band (433E6, 868E6, or 915E6)

// Watchdog Timer Variables
unsigned long lastPacketTime = 0;
const unsigned long watchdogTimeout = 30000; // 30 seconds

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

void setup() {
  // 1. Disable brownout detector immediately
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0); 
  
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n===============================================");
  Serial.println("     GROUND STATION TELEMETRY RECEIVER ACTIVE  ");
  Serial.println("===============================================");

  // 2. Initialize LoRa
  LoRa.setPins(ss, rst, dio0);
  Serial.print("[*] Initializing LoRa Radio at "); Serial.print(BAND / 1000000); Serial.println(" MHz...");
  if (!LoRa.begin(BAND)) {
    Serial.println("[-] ERROR: Starting LoRa failed! Check wiring.");
    while (1) { delay(10); } 
  }

  // Match transmitter telemetry parameters
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
    // Read packet
    String receivedData = "";
    while (LoRa.available()) {
      receivedData += (char)LoRa.read();
    }

    int rssiVal = LoRa.packetRssi();

    // PARSE AND DISPLAY A BEAUTIFUL DESCRIPTIVE TELEMETRY DASHBOARD
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

    // Print parsed dashboard to Serial Monitor
    Serial.println("\n====================================================");
    Serial.print("  TELEMETRY RECEIVED  |  Signal Strength: "); Serial.print(rssiVal); Serial.println(" dBm");
    Serial.println("====================================================");
    Serial.print("  [SYSTEM]  Uptime Clock: "); Serial.println(uptimeClock);
    Serial.print("  [IMU]     Accel Gs:     ("); Serial.print(gX); Serial.print(", "); Serial.print(gY); Serial.print(", "); Serial.print(gZ); Serial.println(")");
    Serial.print("            Gyro dps:     ("); Serial.print(degX); Serial.print(", "); Serial.print(degY); Serial.print(", "); Serial.print(degZ); Serial.println(")");
    Serial.print("  [GPS]     Coordinates:  "); Serial.print(latStr); Serial.print(", "); Serial.println(lngStr);
    Serial.print("            Altitude:     "); Serial.print(altStr); Serial.print("m  |  Satellites: "); Serial.println(satsStr);
    Serial.print("  [ENV]     Temperature:  "); Serial.print(tempStr); Serial.print(" C  |  Humidity: "); Serial.print(humStr); Serial.println("%");
    Serial.println("====================================================");

    // Feed the watchdog
    lastPacketTime = millis();
  }

  // Watchdog reset if no connection for 30s
  if (millis() - lastPacketTime > watchdogTimeout) {
    Serial.println("\n[!] WATCHDOG TRIGGERED: No telemetry received for 30 seconds!");
    Serial.println("[!] Resetting Ground Station...\n");
    delay(1000);
    ESP.restart(); 
  }
}