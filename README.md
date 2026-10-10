#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>


#define WIFI_SSID "Irma Gilda’s iPhone"
#define WIFI_PASS "xpkaapejw47r4"

const char* LAT = "14.676";              
const char* LON = "121.044";
const int   FORECAST_HOURS = 6;

String FORECAST_URL = String("https://api.open-meteo.com/v1/forecast?latitude=") + LAT +
                      "&longitude=" + LON +
                      "&hourly=temperature_2m,precipitation_probability" +
                      "&forecast_hours=" + FORECAST_HOURS +
                      "&timezone=Asia%2FManila";


const int SDA_PIN = 16, SCL_PIN = 17;
const int SOIL_PIN    = 34;              
const int RAIN_DO_PIN = 35;              
const int DHT_PIN     = 4;


const int SOIL_DRY_RAW = 3300;           
const int SOIL_WET_RAW = 1400;           
int   setpointPct = 40;                  
float rainHoldPct = 60.0;                
const unsigned long FETCH_MS = 30UL * 60UL * 1000UL;            
const unsigned long RETRY_MS = 60UL * 1000UL;                   
const unsigned long FORECAST_MAX_AGE = 3UL * 60UL * 60UL * 1000UL;  


LiquidCrystal_I2C lcd(0x27, 16, 2);      
DHT dht(DHT_PIN, DHT11);


int   soilRaw = 0;
float soilPct = 0;
bool  raining = false;
float airTemp = NAN, airHum = NAN;


float rainChance   = 0;                  
float tempForecast = NAN;                
unsigned long forecastAt = 0;           
bool forecastOk = false;
bool everFetched = false;
unsigned long nextFetch = 0;
bool fetchNow = false;


void readSoil() {
  long sum = 0;
  for (int i = 0; i < 8; i++) sum += analogRead(SOIL_PIN);   
  soilRaw = sum / 8;
  soilPct = constrain((float)(SOIL_DRY_RAW - soilRaw) * 100.0 /
                      (SOIL_DRY_RAW - SOIL_WET_RAW), 0.0, 100.0);
}

void readRain() {
  raining = (digitalRead(RAIN_DO_PIN) == LOW);               
}

void readDht() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  if (!isnan(t)) airTemp = t;                                
  if (!isnan(h)) airHum  = h;
}

void updateLocal() {
  static unsigned long tFast = 0, tDht = 0;
  unsigned long now = millis();
  if (now - tFast >= 500)  { tFast = now; readSoil(); readRain(); }
  if (now - tDht  >= 2000) { tDht  = now; readDht(); }
}

bool forecastFresh() {
  return forecastOk && (millis() - forecastAt) < FORECAST_MAX_AGE;
}

bool fetchForecast() {
  if (WiFi.status() != WL_CONNECTED) { forecastOk = false; return false; }

  WiFiClientSecure client;
  client.setInsecure();                                      
  HTTPClient http;
  http.setTimeout(10000);
  http.begin(client, FORECAST_URL);
  int code = http.GET();

  bool good = false;
  if (code == 200) {
    String body = http.getString();
    DynamicJsonDocument doc(4096);
    if (deserializeJson(doc, body) == DeserializationError::Ok) {
      JsonArray temps = doc["hourly"]["temperature_2m"].as<JsonArray>();
      JsonArray pops  = doc["hourly"]["precipitation_probability"].as<JsonArray>();
      if (!temps.isNull() && !pops.isNull() && pops.size() > 0) {
        float maxPop = 0;
        for (float p : pops) if (p > maxPop) maxPop = p;     
        rainChance   = maxPop;
        tempForecast = temps[0] | NAN;
        forecastAt   = millis();
        forecastOk   = true;
        everFetched  = true;
        good = true;
        Serial.printf("[forecast] OK: rain %.0f %%, temp %.1f C\n", rainChance, tempForecast);
      }
    }
  } else {
    Serial.printf("[forecast] HTTP %d\n", code);
  }
  http.end();

  if (!good) forecastOk = false;                             
  nextFetch = millis() + (good ? FETCH_MS : RETRY_MS);
  return good;
}

void updateForecast() {
  unsigned long now = millis();

  if (WiFi.status() != WL_CONNECTED) {                       
    static unsigned long tRe = 0;
    if (forecastOk) Serial.println("[net] WiFi lost - forecast marked stale");
    forecastOk = false;
    if (now - tRe > 10000) { tRe = now; WiFi.reconnect(); }
    return;
  }
  if (fetchNow || (long)(now - nextFetch) >= 0) {            
    fetchNow = false;
    fetchForecast();
  }
}

enum { HOLD_NONE, HOLD_RAIN_SENSOR, HOLD_SOIL_WET, HOLD_FORECAST };

int holdReason() {
  if (raining)                  return HOLD_RAIN_SENSOR;     
  if (soilPct >= setpointPct)   return HOLD_SOIL_WET;        
  // forecast can only hold the decision back, never create it
  if (forecastFresh() && rainChance >= rainHoldPct) return HOLD_FORECAST;
  return HOLD_NONE;                                          
}

bool shouldIrrigate() { return holdReason() == HOLD_NONE; }

const char* reasonText(int h) {
  switch (h) {
    case HOLD_RAIN_SENSOR: return "HOLD: rain sensor wet";
    case HOLD_SOIL_WET:    return "HOLD: soil is wet enough";
    case HOLD_FORECAST:    return "HOLD: rain forecast";
    default:               return forecastFresh() ? "IRRIGATE: soil dry, no rain expected"
                                                  : "IRRIGATE: soil dry (forecast stale - local decides)";
  }
}

void lcdLine(int row, const char* s) {
  char buf[17];
  snprintf(buf, sizeof(buf), "%-16s", s);
  lcd.setCursor(0, row);
  lcd.print(buf);
}

void forecastLine(char* out, size_t n) {
  if (!everFetched) { snprintf(out, n, "Rain:-- no data"); return; }
  unsigned long mins = (millis() - forecastAt) / 60000UL;
  char age[8];
  if (mins < 100) snprintf(age, sizeof(age), "%lum", mins);
  else            snprintf(age, sizeof(age), "%luh", mins / 60);
  if (forecastFresh()) snprintf(out, n, "R:%3d%% Age:%s", (int)rainChance, age);
  else                 snprintf(out, n, "R:%3d%% STALE %s", (int)rainChance, age);
}

void updateLcd() {
  static unsigned long t = 0;
  unsigned long now = millis();
  if (now - t < 500) return;
  t = now;

  char l0[24], l1[24];
  if ((now / 3000) % 2 == 0) {                               
    snprintf(l0, sizeof(l0), "Soil:%2d%% %s", (int)soilPct, raining ? "WET " : "dry");
    if (isnan(airTemp)) snprintf(l1, sizeof(l1), "Air: --C  Set:%2d%%", setpointPct);
    else                snprintf(l1, sizeof(l1), "Air:%2dC  Set:%2d%%", (int)airTemp, setpointPct);
  } else {                                                   
    forecastLine(l0, sizeof(l0));
    snprintf(l1, sizeof(l1), "%s", shouldIrrigate() ? "-> IRRIGATE" : "-> HOLD");
  }
  lcdLine(0, l0);
  lcdLine(1, l1);
}

void printReport() {
  static unsigned long t = 0;
  unsigned long now = millis();
  if (now - t < 2000) return;
  t = now;

  Serial.println("--------------------------------------------------");
  Serial.printf("LOCAL     soil %.0f %% (raw %d, setpoint %d %%) | rain sensor: %s | air %.1f C, %.0f %% RH\n",
                soilPct, soilRaw, setpointPct, raining ? "WET" : "dry", airTemp, airHum);
  if (everFetched) {
    Serial.printf("RETRIEVED rain %.0f %% (hold >= %.0f %%) | forecast temp %.1f C | age %lu min | %s\n",
                  rainChance, rainHoldPct, tempForecast, (now - forecastAt) / 60000UL,
                  forecastFresh() ? "FRESH" : "STALE");
  } else {
    Serial.println("RETRIEVED no forecast yet");
  }
  Serial.printf("DECISION  %s\n", reasonText(holdReason()));
}

void handleSerial() {
  if (!Serial.available()) return;
  String line = Serial.readStringUntil('\n');
  line.trim();
  float v = 0;
  if (sscanf(line.c_str(), "setpoint %f", &v) == 1)      { setpointPct = (int)v; Serial.printf("setpoint = %d %%\n", setpointPct); }
  else if (sscanf(line.c_str(), "rainhold %f", &v) == 1) { rainHoldPct = v;      Serial.printf("rainhold = %.0f %%\n", v); }
  else if (line == "r" || line == "R")                   { fetchNow = true; }
  else if (line.length())                                { Serial.println("commands: setpoint <n>, rainhold <n>, r"); }
}

void setup() {
  Serial.begin(115200);
  delay(300);
  Serial.println("\nLAB 5 - local data + retrieved data");

  pinMode(RAIN_DO_PIN, INPUT);
  pinMode(SOIL_PIN, INPUT);
  Wire.begin(SDA_PIN, SCL_PIN);
  lcd.init();
  lcd.backlight();
  dht.begin();
  readSoil();
  readRain();

  lcdLine(0, "Connecting WiFi");
  lcdLine(1, WIFI_SSID);
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  unsigned long t0 = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - t0 < 15000) delay(250);
  if (WiFi.status() == WL_CONNECTED) {
    Serial.printf("WiFi connected, IP %s, RSSI %d dBm\n", WiFi.localIP().toString().c_str(), WiFi.RSSI());
  } else {
    Serial.println("WiFi not connected - local sensors only (degraded)");
  }
  nextFetch = millis();                                     
}

void loop() {
  handleSerial();
  updateLocal();
  updateForecast();
  updateLcd();
  printReport();
} 
