/* -----------------------------------------------------------------------------
 * Project:      Smart LPG Gas Monitoring System (IoT)
 * Author:       Lokesh Pandranki
 * Environment:  Arduino IDE / ESP8266 Wi-Fi Module
 * Description:  Reads analog telemetry from an MQ-2 Gas Sensor. If the LPG 
 *               concentration exceeds the safety threshold, it triggers a local 
 *               hardware interrupt (Buzzer) and pushes a low-latency alert 
 *               to the Cadio IoT Cloud platform.
 * ----------------------------------------------------------------------------- */

#include <ESP8266WiFi.h>

// --- Hardware Pin Definitions ---
#define MQ2_SENSOR_PIN A0     // Analog input for MQ-2 gas sensor
#define BUZZER_PIN     D1     // Digital output for the local piezo buzzer
#define STATUS_LED     D2     // Digital output for system status

// --- System Constants & Thresholds ---
const int GAS_THRESHOLD = 450;       // Safe limit for LPG concentration (adjust based on calibration)
const int POLL_INTERVAL = 1000;      // Telemetry polling interval in milliseconds (1 second)

// --- Wi-Fi & IoT Configuration ---
const char* WIFI_SSID     = "YOUR_WIFI_NETWORK_NAME";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
const char* IOT_HOST      = "api.cadio.io"; // Example IoT endpoint

void setup() {
  // Initialize Serial Monitor for debugging
  Serial.begin(115200);
  delay(100);

  // Configure Hardware Pins
  pinMode(MQ2_SENSOR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(STATUS_LED, OUTPUT);
  
  // Ensure alarms are off on boot
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(STATUS_LED, LOW);

  Serial.println("\n[SYSTEM] Initializing Smart LPG Monitor...");
  connectToWiFi();
}

void loop() {
  // 1. Read real-time telemetry from the MQ-2 sensor
  int gasConcentration = analogRead(MQ2_SENSOR_PIN);
  
  // 2. Output current levels to serial monitor for debugging
  Serial.print("[DATA] Current Gas Concentration: ");
  Serial.println(gasConcentration);

  // 3. Evaluate concentration against the safety threshold
  if (gasConcentration > GAS_THRESHOLD) {
    triggerAlarmCondition(gasConcentration);
  } else {
    normalizeSystem();
  }

  // 4. Wait for the next polling cycle
  delay(POLL_INTERVAL);
}

/* ================================================================= */
/*                     HELPER FUNCTIONS                              */
/* ================================================================= */

void connectToWiFi() {
  Serial.print("[WIFI] Connecting to ");
  Serial.println(WIFI_SSID);

  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  // Flash LED while connecting
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    digitalWrite(STATUS_LED, !digitalRead(STATUS_LED)); 
  }

  digitalWrite(STATUS_LED, HIGH); // Solid light indicates successful connection
  Serial.println("\n[WIFI] Successfully connected!");
  Serial.print("[WIFI] Assigned IP Address: ");
  Serial.println(WiFi.localIP());
}

void triggerAlarmCondition(int concentrationLevel) {
  Serial.println("[WARNING] LPG LEAK DETECTED! Triggering protocols...");
  
  // Activate Local Hardware Alarms
  digitalWrite(BUZZER_PIN, HIGH);
  
  // Rapidly flash status LED to indicate emergency
  digitalWrite(STATUS_LED, LOW);
  delay(100);
  digitalWrite(STATUS_LED, HIGH);
  delay(100);

  // Transmit payload to Cloud IoT Platform (5.3s latency architecture)
  transmitCloudAlert(concentrationLevel);
}

void normalizeSystem() {
  // Deactivate alarms during safe conditions
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(STATUS_LED, HIGH); // Keep status LED solid to show system is online
}

void transmitCloudAlert(int concentrationLevel) {
  WiFiClient client;
  
  if (client.connect(IOT_HOST, 80)) {
    Serial.println("[IOT] Pushing alert to Cadio platform...");
    
    // Construct the HTTP GET request payload
    String url = "/triggerAlert?sensor=LPG&value=" + String(concentrationLevel);
    
    client.print(String("GET ") + url + " HTTP/1.1\r\n" +
                 "Host: " + IOT_HOST + "\r\n" +
                 "Connection: close\r\n\r\n");
                 
    Serial.println("[IOT] Alert transmitted successfully.");
  } else {
    Serial.println("[ERROR] Failed to connect to IoT Cloud.");
  }
}
