# SolarGrid AI — Android App + ESP32 Firmware

A real-time solar microgrid monitoring app built with Kotlin + Jetpack Compose,
connected to an ESP32 prototype via Firebase Realtime Database.

---

## Project Structure

```
SolarGridAI/
├── app/                          ← Android Studio project
│   ├── src/main/java/com/solargrid/app/
│   │   ├── MainActivity.kt       ← Entry point + bottom nav
│   │   ├── data/
│   │   │   ├── model/SensorData.kt
│   │   │   └── repository/FirebaseRepository.kt
│   │   └── ui/
│   │       ├── dashboard/        ← Live metrics screen
│   │       ├── history/          ← Reading history + chart
│   │       ├── relay/            ← Relay on/off control
│   │       └── theme/Theme.kt
│   └── google-services.json      ← ⚠️ REPLACE WITH YOUR OWN
└── esp32_firmware/
    └── esp32_solargrid.ino       ← Flash to your ESP32
```

---

## STEP 1 — Firebase Setup (Do this FIRST)

1. Go to https://console.firebase.google.com
2. Click **"Add project"** → name it (e.g. "solargrid-ai") → Create
3. In the left sidebar: **Build → Realtime Database → Create database**
   - Choose your region
   - Start in **test mode** (allows all reads/writes during development)
4. Copy your **database URL** — it looks like:
   `https://solargrid-ai-default-rtdb.firebaseio.com`
5. Go to **Project Settings (gear icon) → Service accounts → Database secrets**
   - Click **"Show"** next to the secret key — copy it
6. Go to **Project Settings → Your apps → Add app → Android**
   - Package name: `com.solargrid.app`
   - App nickname: SolarGrid AI
   - Download **google-services.json**
7. **Replace** `app/google-services.json` with your downloaded file

### Firebase Database Rules (for development)
In Firebase Console → Realtime Database → Rules, set:
```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

---

## STEP 2 — ESP32 Firmware

### Hardware you need
| Component | Notes |
|---|---|
| ESP32 Dev Board | Any 38-pin ESP32 works |
| INA219 module | I2C current/voltage sensor |
| 5V Relay module | Single channel |
| LM2596 buck converter | Set output to exactly 5V before connecting ESP32 |
| 12V DC adapter (2A) | Your "solar panel" simulation |
| 12V LED strip/bulb | Your "load" |
| Breadboard + jumper wires | Male-to-Male and Female-to-Male |

### Wiring
```
12V Adapter (+) ──┬── LM2596 VIN+
                  └── INA219 VIN+
                  └── Relay COM

12V Adapter (−) ──── LM2596 VIN−  ──── ESP32 GND

LM2596 VOUT+ (5V) ── ESP32 VIN
LM2596 VOUT−      ── ESP32 GND

INA219 VCC  ── ESP32 3.3V
INA219 GND  ── ESP32 GND
INA219 SDA  ── ESP32 GPIO21
INA219 SCL  ── ESP32 GPIO22
INA219 VIN− ── Relay NO terminal

Relay VCC   ── LM2596 5V
Relay GND   ── ESP32 GND
Relay IN    ── ESP32 GPIO26
Relay NO    ── 12V LED positive

12V LED negative ── 12V Adapter negative
```

### Flash the firmware
1. Open Arduino IDE (download from https://www.arduino.cc/en/software)
2. Add ESP32 board support:
   - **File → Preferences → Additional boards manager URLs:**
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
   - **Tools → Board → Boards Manager** → search "esp32" → Install "esp32 by Espressif"
3. Install libraries via **Tools → Manage Libraries:**
   - `Adafruit INA219` by Adafruit
   - `ArduinoJson` by Benoit Blanchon
4. Open `esp32_firmware/esp32_solargrid.ino`
5. Edit these 4 lines at the top:
   ```cpp
   const char* WIFI_SSID     = "YOUR_WIFI_SSID";
   const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";
   const char* FIREBASE_HOST = "https://YOUR-PROJECT-default-rtdb.firebaseio.com";
   const char* FIREBASE_AUTH = "YOUR_DATABASE_SECRET_KEY";
   ```
6. **Tools → Board → ESP32 Dev Module**
7. **Tools → Port → COM3** (or whichever port your ESP32 shows)
8. Click **Upload (→)**
9. Open Serial Monitor at 115200 baud to see live output

---

## STEP 3 — Android App in Android Studio

### Requirements
- Android Studio Hedgehog or newer
- JDK 11+
- Android device or emulator (API 26+)

### Steps
1. Open Android Studio → **Open** → select the `SolarGridAI/` folder
2. Wait for Gradle sync to complete (first sync downloads ~500MB)
3. Replace `app/google-services.json` with your Firebase file (from Step 1)
4. Click **Run (▶)** — select your device

### If Gradle sync fails
- File → Invalidate Caches → Invalidate and Restart
- Check you have internet connection (Gradle downloads dependencies)
- Make sure JDK is set: File → Project Structure → SDK Location → JDK

---

## How the data flows

```
ESP32 reads INA219
    ↓  every 10 seconds
HTTP PUT → Firebase RTDB /sensor_data  (overwrites latest)
HTTP POST → Firebase RTDB /history     (appends new entry)
    ↓
Kotlin app Firebase listener detects change
    ↓
StateFlow emits new SensorData
    ↓
Compose recomposition updates UI
    ↓  (when you tap relay toggle)
Kotlin writes to Firebase /relay_command/state
    ↓
ESP32 reads relay_command on next cycle
    ↓
GPIO26 HIGH/LOW → Relay coil → LED ON/OFF
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| App shows "Waiting for device" | Check ESP32 is powered and Serial Monitor shows "Connected!" |
| ESP32 shows HTTP error -1 | Check Wi-Fi credentials and Firebase URL |
| INA219 reads 0.00V | Check I2C wiring (SDA=21, SCL=22) and 3.3V power to sensor |
| Relay doesn't respond | App writes to Firebase in ~1s; ESP32 checks every 10s — wait up to 10s |
| Gradle sync error on first open | Let it complete (can take 5-10 min on first run) |

---

## App Screens

| Screen | Description |
|---|---|
| Dashboard | Live voltage, current, power, Wi-Fi signal; power history bar chart; smart alerts |
| History | All logged readings with timestamps; summary stats |
| Relay | Toggle LED load on/off; see command flow explanation |
