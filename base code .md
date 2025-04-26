#include "security.h" 
#include <BlynkSimpleEsp8266.h>
#include <DHT.h>
#include <AESLib.h>

// Blynk Template Settings
#define BLYNK_TEMPLATE_ID "TMPL22DhPZES"
#define BLYNK_TEMPLATE_NAME "FULL HOME AUTOMATION"
#define BLYNK_PRINT Serial

// Hardware Definitions
#define BUZZER_PIN D0
#define MQ2_PIN A0
#define PIR_PIN D4
#define TRIG_PIN D5
#define ECHO_PIN D6
#define RELAY1_PIN D7
#define RELAY2_PIN D8

// Initialize Components
LiquidCrystal_I2C lcd(0x27, 16, 2);  // I2C Address 0x27, 16x2 LCD
DHT dht(D3, DHT11);                  // DHT11 on pin D3
WiFiClientSecure client;              // Secure client for HTTPS
BlynkTimer timer;                    // Timer for periodic tasks
AESLib aesLib;                       // AES Encryption Library

// AES Encryption Settings
byte aesKey[16] = {0x2b,0x7e,0x15,0x16,0x28,0xae,0xd2,0xa6,0xab,0xf7,0x97,0x75,0x46,0x32,0x84,0x9b};
byte iv[16] = {0};  // Initialization Vector

// State Variables
bool pirEnabled = false;
bool relay1State = false;
bool relay2State = false;

// Blynk Virtual Pin Handlers
BLYNK_WRITE(V0) { pirEnabled = param.asInt(); }

BLYNK_WRITE(V5) {
  relay1State = param.asInt();
  digitalWrite(RELAY1_PIN, !relay1State); // Active LOW
}

BLYNK_WRITE(V6) {
  relay2State = param.asInt();
  digitalWrite(RELAY2_PIN, !relay2State); // Active LOW
}

void setup() {
  Serial.begin(115200);
  
  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.print("Initializing...");
  
  // Configure Pins
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(PIR_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);
  
  // Set initial relay states (HIGH = OFF)
  digitalWrite(RELAY1_PIN, HIGH);
  digitalWrite(RELAY2_PIN, HIGH);

  // Initialize AES Library
  aesLib.set_paddingmode((paddingMode)0);
  randomSeed(analogRead(0));
  for (int i=0; i<16; i++) {
    iv[i] = random(256);  // Generate random IV
  }

  // Connect to WiFi
  WiFi.begin(ssid, pass);
  lcd.clear();
  lcd.print("Connecting WiFi");
  
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
    if (attempts % 4 == 0) lcd.print(".");
  }

  // Connection status
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi Connected");
    lcd.clear();
    lcd.print("WiFi Connected");
    
    // Configure secure client
    client.setInsecure(); // Bypass certificate validation (for testing)
    // For production: client.setFingerprint(fingerprint);
    
    // Connect to Blynk via HTTPS
    Blynk.begin(client, auth, ssid, pass);
  } else {
    Serial.println("\nWiFi Failed");
    lcd.clear();
    lcd.print("WiFi Failed");
  }

  // Start DHT sensor
  dht.begin();
  
  // Setup periodic tasks
  timer.setInterval(1000L, readGasSensor);
  timer.setInterval(2000L, readDHT11Sensor);
  timer.setInterval(500L, readPIRSensor);
  timer.setInterval(1000L, readUltrasonic);
}

// Encrypt data before transmission
String encryptData(String plaintext) {
  byte ciphertext[plaintext.length() + 16];
  int cipherLength = aesLib.encrypt((byte*)plaintext.c_str(), plaintext.length(), ciphertext, aesKey, sizeof(aesKey), iv);
  
  // Convert to hex string
  String result;
  for (int i=0; i<cipherLength; i++) {
    if (ciphertext[i] < 16) result += "0";
    result += String(ciphertext[i], HEX);
  }
  return result;
}

void readGasSensor() {
  int value = analogRead(MQ2_PIN);
  int percentage = map(value, 0, 1024, 0, 100);
  
  // Gas alarm (35% threshold)
  bool gasAlert = (percentage > 35);
  digitalWrite(BUZZER_PIN, gasAlert ? HIGH : LOW);
  
  // Encrypt and send data
  String encrypted = encryptData(String(percentage));
  Blynk.virtualWrite(V1, encrypted);
  
  // Display on LCD
  lcd.setCursor(9, 0);
  lcd.print("G:");
  lcd.print(percentage);
  lcd.print("% ");
  
  if (gasAlert) {
    Blynk.logEvent("gas_alert", "High gas detected!");
  }
}

void readDHT11Sensor() {
  float h = dht.readHumidity();
  float t = dht.readTemperature();

  if (isnan(h) || isnan(t)) {
    Serial.println("DHT Read Error");
    return;
  }

  // Encrypt and send data
  String tempStr = encryptData(String(t, 1));
  String humStr = encryptData(String(h, 1));
  Blynk.virtualWrite(V2, tempStr);
  Blynk.virtualWrite(V3, humStr);

  // Display on LCD
  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.print(t, 1);
  lcd.print("C");
  
  lcd.setCursor(0, 1);
  lcd.print("H:");
  lcd.print(h, 1);
  lcd.print("%");
}

void readPIRSensor() {
  if (pirEnabled) {
    bool motion = digitalRead(PIR_PIN);
    digitalWrite(BUZZER_PIN, motion ? HIGH : LOW);
    
    if (motion) {
      Blynk.virtualWrite(V7, encryptData("1"));
      Blynk.logEvent("motion_alert", "Motion detected!");
    }
  }
}

void readUltrasonic() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  long distance = duration * 0.034 / 2; // cm
  
  // Encrypt and send data
  Blynk.virtualWrite(V4, encryptData(String(distance)));
  
  // Display on LCD
  lcd.setCursor(9, 1);
  lcd.print("D:");
  lcd.print(distance);
  lcd.print("cm");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    Blynk.run();
  }
  timer.run();
}
