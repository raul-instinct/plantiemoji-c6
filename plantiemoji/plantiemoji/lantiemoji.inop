// Plantiemoji-style plant mood face
// Board: Waveshare ESP32-C6-LCD-1.47 (ST7789 172x320), Arduino core esp32 >= 3.0.0
// Library: "GFX Library for Arduino" (moononournation) - NOT yet flashed by the author, see README caveats.
#include <Arduino_GFX_Library.h>
#include <Wire.h>

// ---- Display pins (Waveshare wiki) ----
#define TFT_MOSI 6
#define TFT_SCLK 7
#define TFT_CS   14
#define TFT_DC   15
#define TFT_RST  21
#define TFT_BL   22

// ---- Sensor pins (free header pins) ----
#define SOIL_PIN 1   // GPIO1 = ADC1_CH1
#define I2C_SDA  2
#define I2C_SCL  3

// ---- Calibration: read the serial output, put the probe in air (DRY) and in a glass of water up to the line (WET) ----
#define SOIL_DRY_MV 2600
#define SOIL_WET_MV 1250

Arduino_DataBus *bus = new Arduino_ESP32SPI(TFT_DC, TFT_CS, TFT_SCLK, TFT_MOSI, GFX_NOT_DEFINED);
// 172x320 panel sits in a 240x320 controller: 34px column offset. Rotation 1 = landscape 320x172.
Arduino_GFX *gfx = new Arduino_ST7789(bus, TFT_RST, 1 /*rotation*/, true /*IPS*/, 172, 320, 34, 0, 34, 0);

enum Mood { THIRSTY, OKAY, HAPPY, OVERWATERED };
const char *MOOD_NAME[] = {"THIRSTY", "OKAY", "HAPPY", "TOO WET"};

bool haveBH = false, haveSHT = false;
float lux = NAN, tempC = NAN, rh = NAN;

// ---------- sensors ----------
int readSoilMv() {
  uint32_t sum = 0;
  for (int i = 0; i < 32; i++) { sum += analogReadMilliVolts(SOIL_PIN); delay(2); }
  return sum / 32;
}
int soilPercent(int mv) {
  long p = map(mv, SOIL_DRY_MV, SOIL_WET_MV, 0, 100);
  return constrain(p, 0, 100);
}
bool i2cPresent(uint8_t a) { Wire.beginTransmission(a); return Wire.endTransmission() == 0; }
float readBH1750() {
  Wire.beginTransmission(0x23); Wire.write(0x10); if (Wire.endTransmission()) return NAN; // continuous hi-res
  delay(180);
  if (Wire.requestFrom(0x23, 2) != 2) return NAN;
  uint16_t raw = (Wire.read() << 8) | Wire.read();
  return raw / 1.2f;
}
bool readSHT31(float &t, float &h) {
  Wire.beginTransmission(0x44); Wire.write(0x24); Wire.write(0x00); if (Wire.endTransmission()) return false;
  delay(20);
  if (Wire.requestFrom(0x44, 6) != 6) return false;
  uint8_t b[6]; for (int i = 0; i < 6; i++) b[i] = Wire.read();
  t = -45 + 175.0f * ((b[0] << 8) | b[1]) / 65535.0f;
  h = 100.0f * ((b[3] << 8) | b[4]) / 65535.0f;
  return true;
}

// ---------- mood with hysteresis ----------
Mood moodFor(int pct, Mood cur) {
  const int h = 3; // percent
  int t1 = 25, t2 = 50, t3 = 80;
  switch (cur) { // widen the current band slightly to stop flicker
    case THIRSTY:     t1 += h; break;
    case OKAY:        t1 -= h; t2 += h; break;
    case HAPPY:       t2 -= h; t3 += h; break;
    case OVERWATERED: t3 -= h; break;
  }
  if (pct < t1) return THIRSTY;
  if (pct < t2) return OKAY;
  if (pct < t3) return HAPPY;
  return OVERWATERED;
}

// ---------- drawing ----------
#define C(r,g,b) gfx->color565(r,g,b)
const int CX = 110, CY = 86, R = 76; // face on the left, text panel on the right

void thickCurve(int x0, int x1, int yBase, int depth, int w, uint16_t col) { // parabola, depth>0 = smile
  for (int x = x0; x <= x1; x++) {
    float u = (x - (x0 + x1) / 2.0f) / ((x1 - x0) / 2.0f);
    int y = yBase + (int)(depth * (1 - u * u));
    gfx->fillCircle(x, y, w, col);
  }
}
void eye(int x, int y, int r, uint16_t col) { gfx->fillCircle(x, y, r, col); gfx->fillCircle(x + r / 3, y - r / 3, r / 3, WHITE); }

void drawFace(Mood m, int pct, int mv) {
  gfx->fillScreen(BLACK);
  uint16_t skin = (m == THIRSTY) ? C(222, 184, 100) : (m == OVERWATERED) ? C(120, 200, 215) : (m == OKAY) ? C(240, 215, 90) : C(255, 214, 51);
  uint16_t ink = C(60, 40, 20);
  gfx->fillCircle(CX, CY, R, skin);
  gfx->drawCircle(CX, CY, R, ink);
  int ex = 28, ey = CY - 18;
  switch (m) {
    case HAPPY:
      eye(CX - ex, ey, 9, ink); eye(CX + ex, ey, 9, ink);
      gfx->fillCircle(CX - 46, CY + 8, 9, C(255, 140, 120)); gfx->fillCircle(CX + 46, CY + 8, 9, C(255, 140, 120)); // blush
      thickCurve(CX - 30, CX + 30, CY + 18, 20, 3, ink);
      break;
    case OKAY:
      eye(CX - ex, ey, 8, ink); eye(CX + ex, ey, 8, ink);
      gfx->fillRoundRect(CX - 24, CY + 28, 48, 6, 3, ink);
      break;
    case THIRSTY:
      gfx->fillCircle(CX - ex, ey + 2, 7, ink); gfx->fillCircle(CX + ex, ey + 2, 7, ink);
      gfx->fillRect(CX - ex - 14, ey - 12, 30, 9, skin); gfx->fillRect(CX + ex - 16, ey - 12, 30, 9, skin); // heavy lids
      gfx->drawLine(CX - ex - 14, ey - 3, CX - ex + 14, ey - 3, ink); gfx->drawLine(CX + ex - 14, ey - 3, CX + ex + 14, ey - 3, ink);
      thickCurve(CX - 26, CX + 26, CY + 40, -14, 3, ink);                      // frown
      gfx->fillTriangle(CX + 62, CY - 40, CX + 54, CY - 22, CX + 70, CY - 22, C(90, 170, 255)); gfx->fillCircle(CX + 62, CY - 20, 8, C(90, 170, 255)); // sweat drop
      for (int i = 0; i < 4; i++) gfx->drawLine(CX - 30 + i * 18, CY + 52, CX - 22 + i * 18, CY + 58, ink); // cracks
      break;
    case OVERWATERED:
      gfx->fillCircle(CX - ex, ey, 14, WHITE); gfx->fillCircle(CX + ex, ey, 14, WHITE);
      gfx->fillCircle(CX - ex, ey + 4, 6, ink); gfx->fillCircle(CX + ex, ey + 4, 6, ink);
      for (int x = CX - 30; x <= CX + 30; x++) gfx->fillCircle(x, CY + 34 + (int)(5 * sin((x - CX) / 5.0f)), 3, ink); // wavy mouth
      gfx->fillCircle(CX - 62, CY - 38, 6, C(40, 120, 255)); gfx->fillCircle(CX + 66, CY - 30, 5, C(40, 120, 255));
      gfx->fillCircle(CX - 66, CY - 18, 4, C(40, 120, 255));
      break;
  }
  // text panel
  gfx->setTextColor(WHITE); gfx->setTextSize(3); gfx->setCursor(205, 20); gfx->print(MOOD_NAME[m]);
  gfx->setTextSize(5); gfx->setCursor(205, 56); gfx->printf("%d%%", pct);
  gfx->setTextSize(1); gfx->setTextColor(C(160, 160, 160));
  gfx->setCursor(205, 108); gfx->printf("soil %d mV", mv);
  int y = 122;
  if (!isnan(tempC)) { gfx->setCursor(205, y); gfx->printf("%.1f C  %.0f%% RH", tempC, rh); y += 14; }
  if (!isnan(lux))   { gfx->setCursor(205, y); gfx->printf("%.0f lux", lux); }
}

Mood mood = OKAY;
int lastShownPct = -100;

void setup() {
  Serial.begin(115200);
  pinMode(TFT_BL, OUTPUT); digitalWrite(TFT_BL, HIGH); // lower with analogWrite() if the panel runs hot
  gfx->begin();
  gfx->fillScreen(BLACK);
  analogReadResolution(12);
  analogSetPinAttenuation(SOIL_PIN, ADC_11db);
  Wire.begin(I2C_SDA, I2C_SCL);
  haveBH = i2cPresent(0x23);
  haveSHT = i2cPresent(0x44);
  Serial.printf("BH1750: %d  SHT31: %d\n", haveBH, haveSHT);
}

void loop() {
  int mv = readSoilMv();
  int pct = soilPercent(mv);
  if (haveBH) lux = readBH1750();
  if (haveSHT) readSHT31(tempC, rh);
  Mood next = moodFor(pct, mood);
  Serial.printf("soil=%d mV  moisture=%d%%  mood=%s  lux=%.0f  T=%.1f  RH=%.0f\n", mv, pct, MOOD_NAME[next], lux, tempC, rh);
  if (next != mood || abs(pct - lastShownPct) >= 2) {
    mood = next; lastShownPct = pct;
    drawFace(mood, pct, mv);
  }
  delay(2000);
}
