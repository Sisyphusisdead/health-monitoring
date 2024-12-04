#include <SoftwareSerial.h>  // For ESP8266
#include <dht11.h>           // For DHT11 sensor

// DHT11 Sensor Pin
#define dht_apin 11 // DHT11 data pin
dht11 dhtObject;

// Pulse Sensor Pin
#define pulsePin A0

// ESP8266 Pin Configuration
#define RX 2
#define TX 3
SoftwareSerial esp8266(RX, TX);

// Wi-Fi and ThingSpeak Configuration
String AP = "Redmi";       // Wi-Fi SSID
String PASS = "11111111";  // Wi-Fi Password
String API = "WI0YUACC1A6L377D";   // ThingSpeak Write API Key
String HOST = "api.thingspeak.com";
String PORT = "80";

// Variables for Wi-Fi communication
int countTrueCommand;
int countTimeCommand;
boolean found = false;

// Variables for Pulse Sensor
int signal;                // To store pulse signal value
int threshold = 550;       // Threshold for detecting heartbeat (adjustable)
unsigned long lastBeat = 0;
int bpm = 0;               // Beats per minute
unsigned long previousMillis = 0;

// Setup function
void setup() {
  // Initialize Serial Communication
  Serial.begin(9600);
  esp8266.begin(115200);

  // Connect to Wi-Fi
  connectToWiFi();
}

// Main loop function
void loop() {
  // Read DHT11 Sensor Data
  dhtObject.read(dht_apin);
  int temp = dhtObject.temperature;
  int hum = dhtObject.humidity;

  // Pulse Sensor Reading
  signal = analogRead(pulsePin); // Read pulse sensor value
  if (signal > threshold) {
    unsigned long currentTime = millis();
    if (currentTime - lastBeat > 600) { // Debounce to avoid multiple counts
      bpm = 60000 / (currentTime - lastBeat);
      lastBeat = currentTime;
    }
  }

  // Print Data to Serial Monitor
  Serial.print("Temperature: ");
  Serial.print(temp);
  Serial.println(" °C");
  Serial.print("Humidity: ");
  Serial.print(hum);
  Serial.println(" %");
  Serial.print("Pulse Signal: ");
  Serial.print(signal);
  Serial.print(" BPM: ");
  Serial.println(bpm);

  // Send Data to ThingSpeak
  sendDataToThingSpeak(temp, hum, bpm);

  delay(10000); // Wait 10 seconds before the next loop
}

// Function to Send Data to ThingSpeak
void sendDataToThingSpeak(int temperature, int humidity, int bpm) {
  String getData = "GET /update?api_key=" + API + "&field1=" + String(temperature) + "&field2=" + String(humidity) + "&field3=" + String(bpm);
  Serial.println("Sending data: " + getData);

  // Establish TCP Connection
  if (!sendCommand("AT+CIPMUX=1", 5, "OK")) return;
  if (!sendCommand("AT+CIPSTART=0,\"TCP\",\"" + HOST + "\"," + PORT, 15, "OK")) return;
  
  // Send Data Length
  if (!sendCommand("AT+CIPSEND=0," + String(getData.length() + 4), 5, ">")) return;
  
  // Send the Data
  esp8266.println(getData);
  delay(1500); // Wait for the data to send
  
  // Close Connection
  sendCommand("AT+CIPCLOSE=0", 5, "OK");
}

// Function to Connect to Wi-Fi
void connectToWiFi() {
  if (sendCommand("AT", 5, "OK")) {
    Serial.println("ESP8266 Ready");
  } else {
    Serial.println("ESP8266 Not Responding");
    return;
  }

  if (sendCommand("AT+CWMODE=1", 5, "OK")) {
    Serial.println("Wi-Fi Mode Set to Station");
  } else {
    Serial.println("Failed to Set Wi-Fi Mode");
    return;
  }

  if (sendCommand("AT+CWJAP=\"" + AP + "\",\"" + PASS + "\"", 20, "OK")) {
    Serial.println("Wi-Fi Connected!");
  } else {
    Serial.println("Failed to Connect to Wi-Fi");
  }
}

// Function to Send AT Commands to ESP8266
bool sendCommand(String command, int maxTime, char readReplay[]) {
  Serial.print("Sending: ");
  Serial.println(command);

  int attempt = 0;
  found = false;

  while (attempt < maxTime) {
    esp8266.println(command); // Send command
    if (esp8266.find(readReplay)) { // Look for expected response
      found = true;
      Serial.println("Command Successful!");
      break;
    }
    attempt++;
    delay(500); // Delay between attempts
  }

  if (!found) {
    Serial.println("Command Failed!");
    Serial.println("ESP Response:");
    while (esp8266.available()) {
      Serial.write(esp8266.read()); // Print the ESP8266 response
    }
  }

  return found;
}
