#define BLYNK_TEMPLATE_ID "TMPL3l_Teq5vE"
#define BLYNK_TEMPLATE_NAME "Hydrophonic"
#define BLYNK_AUTH_TOKEN "Ci-3NnkEOUsdAKLftCtDQ2_b1D0_GNj7"
#define BLYNK_PRINT Serial

#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <DHT.h>

// I2C LCD address (use 0x27 for most displays)
LiquidCrystal_I2C lcd(0x27, 16, 2);

// WiFi credentials
char ssid[] = "Keep it on the download";
char pass[] = "0000000000";

// DHT sensor pins and type
#define DHTTYPE DHT11
#define DHTPIN1 19
#define DHTPIN2 18
#define DHTPIN3 13
#define DHTPIN4 14

DHT dht1(DHTPIN1, DHTTYPE);
DHT dht2(DHTPIN2, DHTTYPE);
DHT dht3(DHTPIN3, DHTTYPE);
DHT dht4(DHTPIN4, DHTTYPE);

#define PH_PIN 35
#define TDS_PIN 34
#define ESPADC 4095
#define ESPVOLTAGE 3300

#define RELAY_PIN 23

BlynkTimer timer;

// Variables to display
float t1, h1, t2, h2, t3, h3, t4, h4, phValue, tdsValue;

void sendAllDHT() {
    h1 = dht1.readHumidity();
    t1 = dht1.readTemperature();
    h2 = dht2.readHumidity();
    t2 = dht2.readTemperature();
    h3 = dht3.readHumidity();
    t3 = dht3.readTemperature();
    h4 = dht4.readHumidity();
    t4 = dht4.readTemperature();

    if (isnan(t1) || isnan(h1) || isnan(t2) || isnan(h2) || isnan(t3) || isnan(h3) || isnan(t4) || isnan(h4)) {
        Serial.println("Failed to read from one or more DHT sensors!");
        return;
    }

    Blynk.virtualWrite(V1, t1);
    Blynk.virtualWrite(V2, h1);
    Blynk.virtualWrite(V3, t2);
    Blynk.virtualWrite(V4, h2);
    Blynk.virtualWrite(V5, t3);
    Blynk.virtualWrite(V6, h3);
    Blynk.virtualWrite(V7, t4);
    Blynk.virtualWrite(V8, h4);

    Serial.print("Comp 1: Temp="); Serial.print(t1); Serial.print(" Hum="); Serial.println(h1);
    Serial.print("Comp 2: Temp="); Serial.print(t2); Serial.print(" Hum="); Serial.println(h2);
    Serial.print("Comp 3: Temp="); Serial.print(t3); Serial.print(" Hum="); Serial.println(h3);
    Serial.print("Comp 4: Temp="); Serial.print(t4); Serial.print(" Hum="); Serial.println(h4);
}

void sendPH() {
    int analogValue = analogRead(PH_PIN);
    float voltage = analogValue * (ESPVOLTAGE / (float)ESPADC);
    phValue = 3.5 * (voltage / 1000.0);

    Serial.print("pH Analog: "); Serial.print(analogValue);
    Serial.print("  Voltage: "); Serial.print(voltage);
    Serial.print("  pH: "); Serial.println(phValue);

    Blynk.virtualWrite(V0, phValue);
}

void sendTDS() {
    int tdsAnalog = analogRead(TDS_PIN);
    float tdsVoltage = tdsAnalog * (ESPVOLTAGE / (float)ESPADC);
    tdsValue = (tdsVoltage / 3300.0) * 500;

    Serial.print("TDS Analog: "); Serial.print(tdsAnalog);
    Serial.print("  Voltage: "); Serial.print(tdsVoltage);
    Serial.print("  TDS: "); Serial.println(tdsValue);

    Blynk.virtualWrite(V9, tdsValue);
}

void updateLCD() {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("T1:"); lcd.print((int)t1);
    lcd.print(" H1:"); lcd.print((int)h1);

    lcd.setCursor(0, 1);
    lcd.print("pH:"); lcd.print(phValue, 1);
    lcd.print(" TDS:"); lcd.print((int)tdsValue);
}

void setup() {
    Serial.begin(115200);
    delay(1000); // Serial stabilization

    pinMode(RELAY_PIN, OUTPUT);
    digitalWrite(RELAY_PIN, LOW);

    lcd.init();
    lcd.backlight();

    Blynk.begin(BLYNK_AUTH_TOKEN, ssid, pass);

    dht1.begin(); dht2.begin(); dht3.begin(); dht4.begin();

    timer.setInterval(2000L, sendAllDHT);
    timer.setInterval(3000L, sendPH);
    timer.setInterval(3000L, sendTDS);
    timer.setInterval(2000L, updateLCD);
}

// Only one BLYNK_WRITE per virtual pin (V10)
BLYNK_WRITE(V10) {
    int relayState = param.asInt();
    digitalWrite(RELAY_PIN, relayState);
}

// Only one loop function
void loop() {
    Blynk.run();
    timer.run();
}
