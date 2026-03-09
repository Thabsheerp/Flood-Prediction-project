# Flood-Prediction-project
Iot - Based Flood Prediction And Alert System


#include "DHT.h"
#define DHTPIN 19
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

#include <Adafruit_NeoPixel.h>
#define PIN 18
#define NUMPIXELS 5
Adafruit_NeoPixel pixels(NUMPIXELS, PIN, NEO_GRB + NEO_KHZ800);

#include <WiFi.h>
#include <WiFiMulti.h>
WiFiMulti WiFiMulti;

#include <HTTPClient.h>
#include <WiFiClientSecure.h>
#include <ArduinoJson.h>
DynamicJsonDocument doc(1024);

const int trigPin = 26, echoPin = 25, rain_sensor = 32, level_sensor = 33, connection_led = 27, siren = 21;
int duration = 0, distance = 0, rain_value = 0, level_value = 0, red_threshold = 3000, yellow_threshold = 2000;
float temperature = 0, humidity = 0;
String alert = "GREEN";

int connection_interval = 500, data_interval = 1000;
unsigned long connection_millis = 0, data_millis = 0;

String base_url = "https://flood-prediction-2k25-55h6koaz7a-uc.a.run.app";

void setup() 
{
  Serial.begin(9600);

  pixels.begin();
  pixels.clear();

  for(int i=0; i<NUMPIXELS; i++) 
  {
    pixels.setPixelColor(i, pixels.Color(255, 255, 255));
  }
  pixels.show();
  
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(rain_sensor, INPUT);
  pinMode(level_sensor, INPUT);
  pinMode(connection_led, OUTPUT);
  pinMode(siren, OUTPUT);
  digitalWrite(siren,HIGH);

  dht.begin();

  digitalWrite(connection_led, HIGH);
  WiFi.mode(WIFI_STA);
  WiFiMulti.addAP("THINKFOTECH-4G", "thinkfotech123");
  Serial.print("Waiting for WiFi to connect...");
  while ((WiFiMulti.run() != WL_CONNECTED)) 
  {
    Serial.print(".");
  }
  Serial.println("connected");
  digitalWrite(connection_led, LOW);
}

void loop() 
{
  if ((WiFiMulti.run() != WL_CONNECTED)) 
  {
    ESP.restart();
  } 
  else 
  {
    data_upload();
  }
  
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  duration = pulseIn(echoPin, HIGH);
  distance = (duration * 0.0343)/2;
  delay(100);

  temperature = dht.readTemperature();
  humidity = dht.readHumidity();

  rain_value = 4095 - analogRead(rain_sensor);
  level_value = analogRead(level_sensor);

  if (level_value > red_threshold)
  {
    alert = "RED";
    digitalWrite(siren,LOW);
    for(int i=0; i<NUMPIXELS; i++) 
    {
      pixels.setPixelColor(i, pixels.Color(255, 0, 0));
    }
    pixels.show();
  }
  else if (level_value > yellow_threshold)
  {
    alert = "YELLOW";
    digitalWrite(siren,HIGH);
    for(int i=0; i<NUMPIXELS; i++) 
    {
      pixels.setPixelColor(i, pixels.Color(255, 200, 0));
    }
    pixels.show();
  }
  else
  {
    alert = "GREEN";
    digitalWrite(siren,HIGH);
    for(int i=0; i<NUMPIXELS; i++) 
    {
      pixels.setPixelColor(i, pixels.Color(0, 255, 0));
    }
    pixels.show();
  }

  if (millis() - connection_millis >= connection_interval)
  {
    connection_millis = millis();
    digitalWrite(connection_led,!digitalRead(connection_led));
  }

  if (millis() - data_millis >= data_interval)
  {
    data_millis = millis();
    Serial.print("D : ");
    Serial.print(distance);
    Serial.print(" T : ");
    Serial.print(temperature);
    Serial.print(" H : ");
    Serial.print(humidity);
    Serial.print(" R : ");
    Serial.print(rain_value);
    Serial.print(" L : ");
    Serial.print(level_value);
    Serial.print(" RH : ");
    Serial.print(red_threshold);
    Serial.print(" YH : ");
    Serial.println(yellow_threshold);
  }
}

void data_upload() 
{
  WiFiClientSecure *client = new WiFiClientSecure;
  if (client) 
  {
    client->setInsecure();
    {
      HTTPClient https;
      if (https.begin(*client, base_url + "/data")) 
      {
        https.addHeader("Content-Type", "application/json");
        String serverData = "{\"temperature\":\"" + String(temperature) + "\",\"humidity\":\"" + String(humidity) + "\",\"distance\":\"" + String(distance) + "\",\"waterLevel\":\"" + String(level_value) + "\",\"rain\":\"" + String(rain_value) + "\",\"alert\":\"" + String(alert) + "\"}";
        int httpCode = https.POST(serverData);
        if (httpCode > 0) 
        {
          if (httpCode == HTTP_CODE_OK || httpCode == HTTP_CODE_MOVED_PERMANENTLY) 
          {
            String payload = https.getString();
            deserializeJson(doc, payload);
            JsonObject obj = doc.as<JsonObject>();
            String redThresh = obj[String("redThreshold")];
            String yellowThresh = obj[String("yellowThreshold")];
            red_threshold = redThresh.toInt();
            yellow_threshold = yellowThresh.toInt();
          }
        }
      }
      https.end();
    }
  }
  delete client;
}
