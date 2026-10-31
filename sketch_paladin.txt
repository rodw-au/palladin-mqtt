#include <WiFi.h>
#include <PubSubClient.h>
#include <time.h>


#define LED_BUILTIN 2

// Serial2 Pins for Paladin Interface
#define RXD2 16  // Connect to Paladin TX
#define TXD2 17  // Connect to Paladin RX

// Wi-Fi Configuration.
const char *ssid = "xxxxx";
const char *password = "yyyyy";

// MQTT Broker Configuration
const char *mqtt_broker = "192.168.1.XX";
const int mqtt_port = 1883;
const char *mqtt_username = "homeassistant";
const char *mqtt_password = "xxxxxxx";

// MQTT Topics
const char *topic_cmd = "homeassistant/paladin/command";
const char *topic_state = "homeassistant/paladin/telemetry";

// Telemetry Polling Timer (Request 'A' dataset every 5 seconds)
unsigned long lastPollTime = 0;
const unsigned long POLL_INTERVAL = 5000; 

// Time Server
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 36000;   // GMT +10 in seconds)
const int   daylightOffset_sec = 0;  // No Daylight Saving here


WiFiClient espClient;
PubSubClient client(espClient);

String serialBuffer = "";

/*
 ======================================================================================
  PALADIN API SINGLE-CHARACTER COMMAND REFERENCE (57600 Baud 8N1)
 ======================================================================================
  [ DATA REQUESTS ]
  'A' : Full DataSet for Paladin4-6 (Watts, Temps, Accumulators, Health, Flags, Timestamp)
  'R' : Restricted DataSet for PaladinJB
  'T' : Time Hack coming (Format: T + timestamp + \n)

  [ HOT WATER BOOST CONTROL ]
  'B' : Trigger One-Time Boost (Heats to 60°C)
  'b' : Cancel One-Time / Health Boost

  [ SYSTEM RESET & DEFAULTS ]
  'X' : Cancel all override commands & revert to defaults
  'x' : Reset counters
  'Z' : Perform soft reset (AsoftReset)

  [ MINIMUM WATER TEMP SETTINGS ]
  '1' : Set Min Temp to 10°C
  '2' : Set Min Temp to 20°C
  '3' : Set Min Temp to 30°C
  '4' : Set Min Temp to 40°C
  '5' : Set Min Temp to 50°C
  '6' : Set Min Temp to 60°C

  [ MAXIMUM WATER TEMP SETTINGS ]
  '9' : Set Max Temp to 10°C (Effectively halts power transfer)
  'm' : Reset Max Temp to 73°C (Default)
  'M' : Set Max Temp to 78°C
  'N' : Set Max Temp to 83°C
  'n' : Set Max Temp to 88°C

  [ HARDWARE & RELAY CONTROLS ]
  'P' : Stop ALL Solid-State Relay (SSR) activity
  'p' : Enable / Resume SSR activity
  'C' : Enable Battery Charger (GWCharging = true)
  'c' : Disable Battery Charger (GWCharging = false)
  'I' : Inverter ON (GWActiveFromAI = true)
  'i' : Inverter OFF (GWActiveFromAI = false)
  'E' : Force Solar Export ON
  'e' : Force Solar Export OFF

  [ DISPLAY & AUXILIARY CONTROLS ]
  'S' : Backlight Screen ON
  's' : Backlight Screen OFF
  'J' : Stop Ramp-Up on TopUp (GWRampOff = false)
  'j' : Enable Ramp-Up on TopUp (GWRampOff = true)
  'H' : Enable Health Checking
  'h' : Disable Health Checking
  'W' : Enable 5400W Power Limit
  'w' : Disable 5400W Power Limit

  [ CT OVERFLOW & EXPORT LIMITS ]
  'F' : Force higher OverFlow value (3) for battery charging
  'f' : Revert OverFlow to normal (0)
  '~' : Enable Export Limit
  '!' : Disable Export Limit
  '@'..'&' : Set Export OverFlow limits (1kW to 6kW)
 ======================================================================================
*/

void setupWifi() {
    delay(10);
    Serial.println();
    Serial.print("Connecting to WiFi: ");
    Serial.println(ssid);

    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500);
        Serial.print(".");
    }

    Serial.println("\nWiFi connected. IP address: ");
    Serial.println(WiFi.localIP());
}


void parseAndPublishJson(String csvData) {
    csvData.trim();
    
    int commaCount = 0;
    for (int i = 0; i < csvData.length(); i++) {
        if (csvData.charAt(i) == ',') commaCount++;
    }

    if (commaCount >= 15) {
        String fields[20];
        int fieldCount = 0;
        int lastIdx = 0;

        for (int i = 0; i < csvData.length() && fieldCount < 20; i++) {
            if (csvData.charAt(i) == ',') {
                fields[fieldCount++] = csvData.substring(lastIdx, i);
                lastIdx = i + 1;
            }
        }
        if (lastIdx < csvData.length() && fieldCount < 20) {
            fields[fieldCount++] = csvData.substring(lastIdx);
        }

        // Clean up any NAN or empty fields so they output valid JSON (null or 0)
        for (int i = 0; i < fieldCount; i++) {
            fields[i].trim();
            if (fields[i].equalsIgnoreCase("NAN") || fields[i].length() == 0) {
                fields[i] = "null";
            }
        }

        String jsonBuffer = "{";
        jsonBuffer += "\"mains_watts\":" + fields[0] + ",";
        jsonBuffer += "\"trans_watts\":" + fields[1] + ",";
        jsonBuffer += "\"solar_watts\":" + fields[2] + ",";
        jsonBuffer += "\"water_temp\":" + fields[3] + ",";
        jsonBuffer += "\"delta_temp\":" + fields[4] + ",";
        jsonBuffer += "\"min_temp\":" + fields[5] + ",";
        jsonBuffer += "\"max_temp\":" + fields[6] + ",";
        jsonBuffer += "\"mains_watts_in\":" + fields[7] + ",";
        jsonBuffer += "\"mains_watts_out\":" + fields[8] + ",";
        jsonBuffer += "\"trans_watts_in\":" + fields[9] + ",";
        //jsonBuffer += "\"trans_watts_out\":" + fields[10] + ",";
        jsonBuffer += "\"trans_watts_out\":";
        if (fields[10].length() == 0 || fields[10] == "null") {
            jsonBuffer += "0.0,";
        } else {
            jsonBuffer += fields[10] + ",";
        }
        jsonBuffer += "\"solar_watts_out\":" + fields[11] + ",";
        jsonBuffer += "\"health_hours\":" + fields[12] + ",";
        jsonBuffer += "\"gw_active\":" + fields[13] + ",";
        jsonBuffer += "\"gw_charging\":" + fields[14] + ",";
        jsonBuffer += "\"solar_export\":" + fields[15];
        
        if (fieldCount > 16) {
            jsonBuffer += ",\"timestamp\":" + fields[16];
        }
        jsonBuffer += "}";

        Serial.printf("JSON Telemetry: %s\n", jsonBuffer.c_str());
        bool published = client.publish(topic_state, jsonBuffer.c_str());
        if (published) {
            Serial.println("MQTT publish success!");
        } else {
            Serial.println("MQTT publish failed!");
        }
        
    } else {
        Serial.printf("Raw Serial: %s\n", csvData.c_str());
        bool published = client.publish(topic_state, csvData.c_str());
        if (published) {
            Serial.println("MQTT RAW publish success!");
        } else {
            Serial.println("MQTT RAW publish failed!");
        }        
    }
}

//Autopublish device to Home Assistant
void publishDiscoveryConfig() {
    String deviceInfo = "\"device\":{"
                        "\"identifiers\":[\"paladin_solar_diverter\"],"
                        "\"name\":\"Paladin Solar Diverter\","
                        "\"model\":\"ESP32 Controller\","
                        "\"manufacturer\":\"MrRodW\"}";

    // Helper macro or lambda for cleaner publishing
    auto publishSensor = [&](String id, String name, String unit, String devClass, String stateClass) {
        String topic = "homeassistant/sensor/paladin/" + id + "/config";
        String payload = "{"
                         "\"name\":\"" + name + "\","
                         "\"unique_id\":\"paladin_" + id + "\","
                         "\"state_topic\":\"homeassistant/paladin/telemetry\","
                         "\"value_template\":\"{{ value_json." + id + " }}\"";
        
        if (unit.length() > 0) payload += ",\"unit_of_measurement\":\"" + unit + "\"";
        if (devClass.length() > 0) payload += ",\"device_class\":\"" + devClass + "\"";
        if (stateClass.length() > 0) payload += ",\"state_class\":\"" + stateClass + "\"";
        
        payload += "," + deviceInfo + "}";
        client.publish(topic.c_str(), payload.c_str(), true); // Retained = true
    };

    // Power Sensors
    publishSensor("mains_watts", "Mains Power", "W", "power", "measurement");
    publishSensor("trans_watts", "Transfer Power", "W", "power", "measurement");
    publishSensor("solar_watts", "Solar Power", "W", "power", "measurement");
    publishSensor("mains_watts_in", "Mains Power In", "W", "power", "measurement");
    publishSensor("mains_watts_out", "Mains Power Out", "W", "power", "measurement");
    publishSensor("trans_watts_in", "Transfer Power In", "W", "power", "measurement");
    publishSensor("trans_watts_out", "Transfer Power Out", "W", "power", "measurement");
    publishSensor("solar_watts_out", "Solar Power Out", "W", "power", "measurement");

    // Temperature Sensors
    publishSensor("water_temp", "Water Temperature", "°C", "temperature", "measurement");
    publishSensor("delta_temp", "Delta Temperature", "°C", "temperature", "measurement");
    publishSensor("min_temp", "Minimum Temperature", "°C", "temperature", "measurement");
    publishSensor("max_temp", "Maximum Temperature", "°C", "temperature", "measurement");

    // System & Status Metrics
    publishSensor("health_hours", "System Health Hours", "h", "", "total_increasing");
    publishSensor("gw_active", "Gateway Active", "", "", "measurement");
    publishSensor("gw_charging", "Gateway Charging", "", "", "measurement");
    publishSensor("solar_export", "Solar Export", "", "", "measurement");
}

void callback(char *topic, byte *payload, unsigned int length) {
    String message = "";
    for (int i = 0; i < length; i++) {
        message += (char)payload[i];
    }
    message.trim();

    Serial.printf("MQTT received [%s]: %s\n", topic, message.c_str());

    // Home assistant payload to set time to GMT + 10 (36000 seconds): {{ 'T' ~ (now().timestamp() | int + 36000) }}
    if (message.startsWith("T") && message.length() > 0){
        String timeStr = message.substring(1);
        struct timeval epoch_time;
        epoch_time.tv_sec = timeStr.toInt();
        settimeofday(&epoch_time, NULL);
        time_t nowEpoch = time(NULL);
        struct tm timeinfo;
        if (localtime_r(&nowEpoch, &timeinfo)) {
            char timeString[64];
            strftime(timeString, sizeof(timeString), "%Y-%m-%d %H:%M:%S", &timeinfo);
            Serial.printf("ESP32 clock synced to: %s\n", timeString);
        }
    }
    char cmdToSend = 0;
    // Map common Home Assistant string aliases to Paladin single-char commands
    if (message.equalsIgnoreCase("BOOST") || message.equalsIgnoreCase("ON") || message.equalsIgnoreCase("BOOST_ON")) {
        cmdToSend = 'B';
    } else if (message.equalsIgnoreCase("STOP") || message.equalsIgnoreCase("OFF") || message.equalsIgnoreCase("BOOST_OFF")) {
        cmdToSend = 'b';
    } else if (message.equalsIgnoreCase("POLL") || message.equalsIgnoreCase("STATUS") || message.equalsIgnoreCase("GET")) {
        cmdToSend = 'A';
    } else if (message.equalsIgnoreCase("RESET") || message.equalsIgnoreCase("CANCEL")) {
        cmdToSend = 'X';
    } else if (message.length() == 1) {
        // If a single character was published directly (e.g. 'B', 'b', '1', '6', 'P', 'p', 'S', 's', 'X')
        cmdToSend = message.charAt(0);
    }

    if (cmdToSend != 0) {
        Serial2.write(cmdToSend);
        Serial.printf("Sent single char command to Paladin: '%c' (ASCII %d)\n", cmdToSend, cmdToSend);
    } else {
        // Fallback: send raw string
        Serial2.print(message);
        Serial.printf("Sent raw string to Paladin: %s\n", message.c_str());
    }
}

void printLocalTime(){
  struct tm timeinfo;
  if(!getLocalTime(&timeinfo)){
    Serial.println("Failed to obtain time");
    return;
  }
  Serial.println(&timeinfo, "%A, %B %d %Y %H:%M:%S");
  Serial.println();
}

void reconnect() {
    // Loop until connected to the MQTT broker
    while (!client.connected()) {
        // First, ensure Wi-Fi is actually connected
        if (WiFi.status() != WL_CONNECTED) {
            Serial.println("Wi-Fi disconnected. Reconnecting Wi-Fi...");
            setupWifi();
        }

        String client_id = "esp32-paladin-";
        client_id += String(WiFi.macAddress());
        Serial.printf("Connecting to MQTT broker as %s...\n", client_id.c_str());

        if (client.connect(client_id.c_str(), mqtt_username, mqtt_password)) {
            Serial.println("MQTT connected.");
            publishDiscoveryConfig();
            client.subscribe(topic_cmd);
            Serial.printf("Subscribed to %s\n", topic_cmd);
            break; // Exit the while once successfully connected
        } else {
            Serial.print("Failed MQTT connection, rc=");
            Serial.print(client.state());
            Serial.println(" retrying in 5 seconds...");
            delay(5000);
        }
    }
    Serial.println("Waiting for NTP time sync...");
    configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
    printLocalTime();
    struct tm timeinfo;
    if(getLocalTime(&timeinfo)){    
      time_t epochTime = mktime(&timeinfo);
      char paladinCommand[32];
      snprintf(paladinCommand, sizeof(paladinCommand), "T%ld", (long)(epochTime+gmtOffset_sec+daylightOffset_sec));
      Serial.print("Sending Time to Paladin: ");
      Serial.println(paladinCommand); 
      Serial2.println(paladinCommand);  
    } 
    else{
       Serial.println("Failed to obtain time");
    }  
}


void setup() {
    pinMode(LED_BUILTIN, OUTPUT);
    digitalWrite(LED_BUILTIN, LOW);

    // USB Hardware Serial for debugging
    Serial.begin(115200);

    // Paladin Hardware Serial (Serial2: 57600 baud, 8N1)
    Serial2.begin(57600, SERIAL_8N1, RXD2, TXD2);

    client.setServer(mqtt_broker, mqtt_port); 
    client.setCallback(callback);
    client.setBufferSize(512); 
    
}

void loop() {
    if (!client.connected()) {
        reconnect();
    }
    client.loop();

    // 1. Read incoming telemetry data from Paladin and publish to MQTT
    while (Serial2.available() > 0) {
        char c = Serial2.read();
        if (c == '\n') {
          if (serialBuffer.length() > 0) {
           // Call parser instead of publishing raw serialBuffer directly
            parseAndPublishJson(serialBuffer);
            digitalWrite(LED_BUILTIN, LOW);
            delay(10);
            digitalWrite(LED_BUILTIN, HIGH);
            } 
            serialBuffer = "";
        } else if (c != '\r') {
            serialBuffer += c;
        }
    } 
    // 2. Periodic poll request ('A') if Paladin does not auto-stream
    unsigned long now = millis();
    if (now - lastPollTime >= POLL_INTERVAL) {
        lastPollTime = now;
        Serial2.write('A'); // Uncomment if your Paladin revision requires polling to output data
    }


}
