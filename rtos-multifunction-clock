#include <Wire.h>
#include <U8g2lib.h>
#include <Adafruit_INA219.h>
#include <Adafruit_VEML6070.h>
#include <RTClib.h>
#include <DHT.h>
#include <math.h>

/* ================= OLED ================= */
U8G2_SH1106_128X64_NONAME_F_HW_I2C u8g2(U8G2_R0, U8X8_PIN_NONE);

/* ================= SENSOR ================= */
Adafruit_INA219 ina219;
Adafruit_VEML6070 uv;
RTC_DS3231 rtc;

/* ================= DHT11 ================= */
#define DHTPIN 8
#define DHTTYPE DHT11
DHT dht(DHTPIN, DHTTYPE);

/* ================= BUTTON ================= */
#define BTN_MODE 2
#define BTN_UP   1
#define BTN_DOWN 0

/* ================= UI OFFSET ================= */
#define OFFSET_Y 4
#define PIN_OFFSET 12

/* ================= MODE ================= */
enum ModeState {
  MODE_NORMAL,
  MODE_SET_HOUR,
  MODE_SET_MIN,
  MODE_SET_SEC,
  MODE_SET_DAY,
  MODE_SET_MONTH,
  MODE_SET_YEAR
};
volatile ModeState modeState = MODE_NORMAL;

/* ================= TIME SET ================= */
int setHour, setMin, setSec, setDay, setMonth, setYear;

/* ================= GLOBAL DATA ================= */
char timeStr[9];
char dateStr[11];
float temperature = 0, humidity = 0;
float batteryVoltage = 0;
int batteryPercent = 0;
uint16_t uvRaw = 0;

/* ================= BUTTON INTERRUPT ================= */
volatile bool evtMode = false;
volatile bool evtUp   = false;
volatile bool evtDown = false;

volatile unsigned long lastIsrTime = 0;
const unsigned long debounceIsr = 200;
unsigned long bootTime;

/* ================= BATTERY ================= */
int calcBatteryPercent(float v) {
  if (v >= 4.2) return 100;
  if (v <= 3.2) return 0;
  return (v - 3.2) * 100 / (4.2 - 3.2);
}

/* ================= ISR ================= */
void IRAM_ATTR isrBtnMode() {
  unsigned long now = millis();
  if (now - lastIsrTime > debounceIsr) {
    evtMode = true;
    lastIsrTime = now;
  }
}

void IRAM_ATTR isrBtnUp() {
  unsigned long now = millis();
  if (now - lastIsrTime > debounceIsr) {
    evtUp = true;
    lastIsrTime = now;
  }
}

void IRAM_ATTR isrBtnDown() {
  unsigned long now = millis();
  if (now - lastIsrTime > debounceIsr) {
    evtDown = true;
    lastIsrTime = now;
  }
}

/* ================= ICON ================= */
void drawBatteryIcon(int x, int y, int percent) {
  y += PIN_OFFSET;
  u8g2.drawFrame(x, y + OFFSET_Y, 18, 8);
  u8g2.drawBox(x + 18, y + 2 + OFFSET_Y, 2, 4);
  u8g2.drawBox(x + 1, y + 1 + OFFSET_Y,
               map(percent, 0, 100, 0, 16), 6);
}

void drawSunIcon(int x, int y) {
  y += OFFSET_Y;
  u8g2.drawDisc(x, y, 3);
  for (int i = 0; i < 360; i += 45) {
    float r = i * DEG_TO_RAD;
    u8g2.drawLine(
      x + cos(r) * 5, y + sin(r) * 5,
      x + cos(r) * 8, y + sin(r) * 8
    );
  }
}

/* ================= TASK TIME ================= */
void TimeTask(void *pv) {
  rtc.begin();
  while (1) {
    if (modeState == MODE_NORMAL) {
      DateTime now = rtc.now();
      sprintf(timeStr, "%02d:%02d:%02d",
              now.hour(), now.minute(), now.second());
      sprintf(dateStr, "%02d/%02d/%04d",
              now.day(), now.month(), now.year());
    }
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

/* ================= TASK BATTERY ================= */
void BatteryTask(void *pv) {
  ina219.begin();
  while (1) {
    batteryVoltage = ina219.getBusVoltage_V();
    batteryPercent = calcBatteryPercent(batteryVoltage);
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}

/* ================= TASK UV ================= */
void UVTask(void *pv) {
  uv.begin(VEML6070_1_T);
  while (1) {
    uvRaw = uv.readUV();
    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

/* ================= TASK DHT ================= */
void DHTTask(void *pv) {
  dht.begin();
  while (1) {
    float h = dht.readHumidity();
    float t = dht.readTemperature();
    if (!isnan(h) && !isnan(t)) {
      humidity = h;
      temperature = t;
    }
    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}

/* ================= TASK BUTTON (EVENT-BASED) ================= */
void ButtonTask(void *pv) {
  while (1) {

    if (millis() - bootTime < 1500) {
      vTaskDelay(pdMS_TO_TICKS(20));
      continue;
    }

    if (evtMode) {
      evtMode = false;

      if (modeState == MODE_NORMAL) {
        DateTime now = rtc.now();
        setHour  = now.hour();
        setMin   = now.minute();
        setSec   = now.second();
        setDay   = now.day();
        setMonth = now.month();
        setYear  = now.year();
        modeState = MODE_SET_HOUR;
      }
      else if (modeState == MODE_SET_YEAR) {
        rtc.adjust(DateTime(
          setYear, setMonth, setDay,
          setHour, setMin, setSec
        ));
        modeState = MODE_NORMAL;
      }
      else {
        modeState = (ModeState)(modeState + 1);
      }
    }

    if (evtUp) {
      evtUp = false;
      switch (modeState) {
        case MODE_SET_HOUR:  setHour = (setHour + 1) % 24; break;
        case MODE_SET_MIN:   setMin  = (setMin + 1) % 60; break;
        case MODE_SET_SEC:   setSec  = (setSec + 1) % 60; break;
        case MODE_SET_DAY:   setDay   = constrain(setDay + 1, 1, 31); break;
        case MODE_SET_MONTH: setMonth = constrain(setMonth + 1, 1, 12); break;
        case MODE_SET_YEAR:  setYear++; break;
        default: break;
      }
    }

    if (evtDown) {
      evtDown = false;
      switch (modeState) {
        case MODE_SET_HOUR:  setHour = (setHour + 23) % 24; break;
        case MODE_SET_MIN:   setMin  = (setMin + 59) % 60; break;
        case MODE_SET_SEC:   setSec  = (setSec + 59) % 60; break;
        case MODE_SET_DAY:   setDay   = constrain(setDay - 1, 1, 31); break;
        case MODE_SET_MONTH: setMonth = constrain(setMonth - 1, 1, 12); break;
        case MODE_SET_YEAR:  setYear--; break;
        default: break;
      }
    }

    vTaskDelay(pdMS_TO_TICKS(10));
  }
}

/* ================= TASK DISPLAY ================= */
void DisplayTask(void *pv) {
  while (1) {
    u8g2.clearBuffer();
    u8g2.setFont(u8g2_font_6x12_tf);

    if (modeState == MODE_NORMAL) {

      drawBatteryIcon(0, 10, batteryPercent);
      u8g2.setCursor(22, 20 + OFFSET_Y + PIN_OFFSET);
      u8g2.print(batteryPercent);
      u8g2.print("%");

      u8g2.setCursor(78, 28 + OFFSET_Y);
      u8g2.print(timeStr);

      u8g2.setCursor(68, 42 + OFFSET_Y);
      u8g2.print(dateStr);

      u8g2.setCursor(0, 54 + OFFSET_Y);
      u8g2.print(temperature, 0);
      u8g2.print("C ");
      u8g2.print(humidity, 0);
      u8g2.print("%");

      int uvY = 60 + OFFSET_Y;
      drawSunIcon(78, uvY - OFFSET_Y - 4);
      u8g2.setCursor(88, uvY);
      u8g2.print("UV");
      u8g2.setCursor(106, uvY);
      u8g2.print(uvRaw / 10);

    } else {

      u8g2.setCursor(32, 16);
      u8g2.print("SET TIME");

      u8g2.setCursor(30, 32);
      u8g2.printf("%02d:%02d:%02d", setHour, setMin, setSec);

      u8g2.setCursor(20, 52);
      u8g2.printf("%02d/%02d/%04d", setDay, setMonth, setYear);

      switch (modeState) {
        case MODE_SET_HOUR:  u8g2.drawFrame(29, 21, 14, 14); break;
        case MODE_SET_MIN:   u8g2.drawFrame(47, 21, 14, 14); break;
        case MODE_SET_SEC:   u8g2.drawFrame(65, 21, 14, 14); break;
        case MODE_SET_DAY:   u8g2.drawFrame(19, 41, 14, 14); break;
        case MODE_SET_MONTH: u8g2.drawFrame(37, 41, 14, 14); break;
        case MODE_SET_YEAR:  u8g2.drawFrame(55, 41, 26, 14); break;
        default: break;
      }
    }

    u8g2.sendBuffer();
    vTaskDelay(pdMS_TO_TICKS(200));
  }
}

/* ================= SETUP ================= */
void setup() {
  Serial.begin(115200);

  pinMode(BTN_MODE, INPUT_PULLUP);
  pinMode(BTN_UP,   INPUT_PULLUP);
  pinMode(BTN_DOWN, INPUT_PULLUP);

  attachInterrupt(digitalPinToInterrupt(BTN_MODE), isrBtnMode, FALLING);
  attachInterrupt(digitalPinToInterrupt(BTN_UP),   isrBtnUp,   FALLING);
  attachInterrupt(digitalPinToInterrupt(BTN_DOWN), isrBtnDown, FALLING);

  bootTime = millis();

  Wire.begin(4, 3);
  u8g2.begin();

  xTaskCreate(TimeTask,    "Time",    4096, NULL, 1, NULL);
  xTaskCreate(BatteryTask,"Battery", 4096, NULL, 1, NULL);
  xTaskCreate(UVTask,     "UV",      4096, NULL, 1, NULL);
  xTaskCreate(DHTTask,    "DHT",     4096, NULL, 1, NULL);
  xTaskCreate(ButtonTask, "Button",  4096, NULL, 2, NULL);
  xTaskCreate(DisplayTask,"Display", 4096, NULL, 1, NULL);
}

void loop() {}
