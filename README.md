#include "secrets.h"//Security file recall 
#define BLYNK_TEMPLATE_ID "TMPL22DhPZES"
#define BLYNK_TEMPLATE_NAME "FULL HOME AUTOMATION"
#define BLYNK_PRINT Serial

#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
DHT dht(D3, DHT11);
BlynkTimer timer;

#define Buzzer D0
#define MQ2 A0
#define PIR D4
#define trig D5
#define echo D6
#define relay1 D7
#define relay2 D8

bool pirbutton = 0;
bool connected = false;
unsigned long lastRelayTrigger = 0;
const unsigned long relayTimeout = 60000;

void checkConnection() {
if (!Blynk.connected()) {
if (connected) {
Serial.println("Blynk disconnected");
connected = false;
Blynk.connect();
}
} else {
if (!connected) {
Serial.println("Reconnected to Blynk");
connected = true;
}
}
}

void sendSensorData() {
float temp = dht.readTemperature();
float hum = dht.readHumidity();
int gasValue = analogRead(MQ2);

if (isnan(temp) || isnan(hum)) return;
lcd.clear();
lcd.setCursor(0, 0);
lcd.print("Temp: ");
lcd.print(temp);
lcd.print("C");

lcd.setCursor(0, 1);
lcd.print("Hum: ");
lcd.print(hum);
lcd.print("%");

Blynk.virtualWrite(V1, temp);
Blynk.virtualWrite(V2, hum);
Blynk.virtualWrite(V3, gasValue);

// gas alarm
if (gasValue > 400) {
digitalWrite(Buzzer, HIGH);
Blynk.logEvent("gas_alert", "High gas level detected!");
} else {
digitalWrite(Buzzer, LOW);
}


// motion alarm
if (pirbutton && digitalRead(PIR) == HIGH) {
Blynk.logEvent("motion_alert", "Motion detected!");
digitalWrite(Buzzer, HIGH);
delay(2000);
digitalWrite(Buzzer, LOW);
}
}

BLYNK_WRITE(V0) {
pirbutton = param.asInt();
}

BLYNK_WRITE(V5) {
int r1 = param.asInt();
digitalWrite(relay1, r1);
lastRelayTrigger = millis();
}

BLYNK_WRITE(V6) {
int r2 = param.asInt();
digitalWrite(relay2, r2);
lastRelayTrigger = millis();
}

void failsafeCheck() {
if (millis() - lastRelayTrigger > relayTimeout) {
digitalWrite(relay1, HIGH);
digitalWrite(relay2, HIGH);
Serial.println("Failsafe: Relays turned off");
}
}

void setup() {
Serial.begin(9600);
lcd.init();
lcd.backlight();
dht.begin();

pinMode(Buzzer, OUTPUT);
pinMode(PIR, INPUT);
pinMode(trig, OUTPUT);
pinMode(echo, INPUT);
pinMode(relay1, OUTPUT);
pinMode(relay2, OUTPUT);
digitalWrite(relay1, HIGH);
digitalWrite(relay2, HIGH);

Blynk.begin(SECRET_AUTH_TOKEN, SECRET_SSID, SECRET_PASS);
timer.setInterval(10000L, checkConnection);
timer.setInterval(2000L, sendSensorData);
timer.setInterval(5000L, failsafeCheck);
}

void loop() {
if (Blynk.connected()) {
Blynk.run();
}
timer.run();
}
