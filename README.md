# plantiemoji-c6

A plant that tells you how it feels. Waveshare ESP32-C6-LCD-1.47 + capacitive soil moisture sensor v1.2 (+ optional BH1750 light, SHT31 temp/humidity). Four moods: thirsty, okay, happy, too wet.

Status: written against the Waveshare docs and library APIs, not flashed by the author. Expect to tune the display offset and the calibration numbers. Treat it as a starter.

## 1. Wiring

Pins already used by the board (Waveshare wiki): LCD MOSI 6, SCLK 7, CS 14, DC 15, RST 21, BL 22. microSD CS 4, MISO 5 (MOSI/SCLK shared with LCD). RGB LED 8. BOOT 9. USB D-/D+ 12/13.

Free header pins used here:

| Signal | GPIO | Why |
|---|---|---|
| Soil sensor AOUT | **GPIO1** | ADC1_CH1. GPIO0-3 are free on the header and ADC-capable (GPIO0 is a fine alternative). |
| I2C SDA (BH1750 + SHT31) | **GPIO2** | Free, not a strapping pin. |
| I2C SCL (BH1750 + SHT31) | **GPIO3** | Free, not a strapping pin. |
| Power | **3V3** | Power everything from 3V3, not 5V. |
| Ground | **GND** | |

Avoid GPIO4, 5 (SD slot, strapping), 8 (RGB LED, strapping), 9 (BOOT), 12/13 (USB), 15 (strapping + LCD DC).

```
Soil v1.2:  VCC -> 3V3   GND -> GND   AOUT -> GPIO1
BH1750:     VCC -> 3V3   GND -> GND   SDA -> GPIO2   SCL -> GPIO3   (ADDR floating/GND = 0x23)
SHT31:      VCC -> 3V3   GND -> GND   SDA -> GPIO2   SCL -> GPIO3   (ADDR low = 0x44)
```
Both I2C parts share the same two wires (different addresses). Breakout boards have their own pull-ups; if you stack several, the combined pull-up is still fine at 100-400 kHz.

Header order, from the schematic/third-party pinout: GPIO9(BOOT), 18, 19, 20, 23, 12, 13, 17(RX), 16(TX), 5V, GND, 3V3, 0, 1, 2, 3, 4, 5. Confirm against the silkscreen on your board before powering up.

### Moisture sensor voltage (read this)
- The v1.2 has an onboard 3.3 V regulator and outputs roughly 1.2 V (wet) to 3.0 V (dry). The C6 ADC at 11 dB attenuation reads up to about 3.1 V, so it fits. The code reads in millivolts via `analogReadMilliVolts()`.
- Cheap clones vary. Run it from 3V3 and read the raw mV in the serial monitor first. Never feed a clone that lacks the regulator from 5V and then wire its output to the C6: GPIO is not 5 V tolerant.
- Calibrate: probe in air = `SOIL_DRY_MV`, probe in water up to the marked line = `SOIL_WET_MV`. Defaults (2600/1250) are typical, not yours.
- Seal the top edge and electronics with nail polish or heat shrink. Keep the electronics above the soil line. Water on the board ruins it.
- ADC1 only: Wi-Fi does not disturb it. If you later add Wi-Fi, keep it on ADC1 pins (GPIO0-6).

## 2. Arduino IDE setup
1. Boards Manager: install **esp32 by Espressif Systems**, version 3.0.0 or later (Waveshare requires it).
2. Library Manager: install **GFX Library for Arduino** (moononournation, Arduino_GFX). No other libraries needed: BH1750 and SHT31 are read with plain Wire.
3. Tools: Board **ESP32C6 Dev Module**, **USB CDC On Boot: Enabled** (needed for Serial over the USB-C), Flash 4MB. Pick the port. If it does not appear, hold BOOT while plugging in.
4. Open `plantiemoji/plantiemoji.ino`, upload, open Serial Monitor at 115200.

Display driver: ST7789, 172x320 IPS in a 240x320 controller RAM, so the constructor uses a 34 px column offset: `Arduino_ST7789(bus, 21, rotation, true, 172, 320, 34, 0, 34, 0)`. `Arduino_ESP32SPI(DC=15, CS=14, SCK=7, MOSI=6, MISO=none)`. Backlight is GPIO22, driven high.
If the image is shifted by ~34 px or colours are inverted: change the offsets or the IPS flag (true/false). Rotation 0-3 selects portrait/landscape.
Waveshare warns the panel can get hot at full brightness; PWM the backlight with `analogWrite(22, 128)` if so.

## 3. Moods
Moisture % = linear map of mV between DRY and WET.
- under 25%: THIRSTY (droopy lids, frown, sweat drop)
- 25-50%: OKAY (flat mouth)
- 50-80%: HAPPY (smile, blush)
- over 80%: TOO WET (wide eyes, wavy mouth, drops)
Thresholds have a 3% hysteresis so the face does not flicker. Lux and temp/RH are shown as text; hook them into the mood function if you want (dark = sleepy, hot = sweaty).

## 4. Reusing the probe you already have
Cheap commercial probes fall into three types:
1. **Analog meter** (needle or LED bar, no wireless, usually a pair of metal tines): resistive. You can pull a signal, but only with a voltage divider from a 3.3 V GPIO pulsed on briefly (to avoid electrolysis eating the tines). Works, but readings drift and the tines corrode. Not worth it.
2. **Tuya/Zigbee/Bluetooth sensor** (battery powered, app, a chip behind a sealed head): there is no analog output to tap. Options: read it over Zigbee (the C6 has an 802.15.4 radio, but that is a bigger project) or via the Tuya cloud/local API over Wi-Fi. Worth it only if you want to avoid buying anything.
3. **Capacitive probe with a 3-pin connector (VCC/GND/AOUT)**: plug straight into the wiring above.

How to tell: open the head. Visible chips (Tuya TYWE/ZS3L module, battery holder, buttons) = type 2. Only a small PCB with two probes and a meter = type 1. Brand and model usually answer it in one search. Rule of thumb: if it is not type 3, spend ~10 NIS on the v1.2.

## 4b. OPTIONAL: use your existing Tuya Zigbee probe via the Tuya cloud (option 2)
Not needed for the base build. Your probe talks Zigbee to a Tuya gateway, so the ESP32 can't read it directly; it can ask the Tuya cloud instead.
1. Create a free account at platform.tuya.com (Tuya IoT Platform), Cloud > Development > Create Cloud Project. Pick the data center that matches your Tuya app account's region (for Israel most likely Central Europe, endpoint https://openapi.tuyaeu.com, but check the region your app account lives in).
2. In the project enable the API services (IoT Core, Authorization Token Management, Device Status Notification).
3. Devices tab > Link Tuya App Account > scan the QR from the Tuya Smart / Smart Life app (the app the gateway and probe are paired in). Your probe then shows up with its **Device ID**.
4. Copy the project's **Access ID** and **Access Secret** (Overview tab).
5. Find the moisture data point: in the project's Device Debugging or via `GET /v1.0/devices/{id}/status`. Tuya soil sensors (category zwjcy) usually report moisture as code `humidity` (0-100) and temperature as `temp_current`, but confirm on your model.
6. Call flow (all headers signed HMAC-SHA256, uppercase hex, 13-digit ms timestamp; docs: https://developer.tuya.com/en/docs/iot/api-request?id=Ka4a8uuo1j4t4): get a token with `GET /v1.0/token?grant_type=1`, then `GET /v1.0/devices/{device_id}/status` with the `access_token` header. A starter is in `optional_tuya_cloud/tuya_soil.ino.txt`.
Caveats: untested; Zigbee soil sensors update every 5-60 min, so the face lags; the Tuya free cloud plan has trial/quota limits and the device-link may need re-authorising; your readings depend on Tuya cloud staying up. Pairing the probe straight to the C6's Zigbee radio (option 3) is possible in principle, but it is a bigger project and not covered here.

## 5. Shopping list (AliExpress)
AliExpress listings change daily, so these are search pages, not pinned listings. Prefer sellers with 4.7+ stars and many orders; check the photos match.

| Item | Qty | Search | Notes |
|---|---|---|---|
| Capacitive soil moisture sensor **v1.2** | 1-2 | https://www.aliexpress.com/w/wholesale-capacitive-soil-moisture-sensor-v1.2.html | Must say capacitive v1.2 (blade PCB with a TL555 chip and a coating), not the cheap two-prong resistive one. Buy a 2-pack. |
| BH1750 (GY-302) light sensor, optional | 1 | https://www.aliexpress.com/w/wholesale-bh1750.html | 3.3 V compatible module. |
| SHT31 (GY-SHT31-D) breakout, optional | 1 | search "GY-SHT31-D" | Get the 3.3 V version. |
| Dupont jumper wires female-female 10-20 cm | 1 pack | search "dupont jumper wire female to female" | Or a 3-pin PH2.0 cable for the v1.2 connector. |
| USB-C cable, plant pot, nail polish for sealing | | | |
