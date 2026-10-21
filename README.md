# MazeBot EduKit — ESP32 Firmware

Firmware robot navigasi labirin (*maze-solving autonomous robot*) berbasis ESP32 DevKit V1 dengan kontroler dual motor DRV8833 dan 3× sensor jarak ultrasonik HC-SR04 berbasis interrupt. Firmware ini dilengkapi server web tertanam (*embedded web dashboard* di memori flash PROGMEM ~74 KB) dengan antarmuka industrial dark, telemetri real-time, kontrol manual D-pad, skematik interaktif SVG, dan live hardware tester tanpa dependensi internet.

---

## ⚡ Fitur Utama

- **Autonomous Wall-Following (Left-Hand Rule):** Algoritma penelusuran dinding kiri untuk pemetaan labirin (*mapping mode*).
- **Reduksi Jalur Labirin (LSRB Optimization):** Menyederhanakan jalur buntu ([L, U, L] ➔ [S], [L, U, S] ➔ [R], dll.) untuk rute tercepat (*speedrun mode*).
- **Embedded Web Dashboard (Zero CDN / 100% Offline):** Antarmuka web modern tersimpan langsung di flash ESP32 (`PROGMEM`), siap diakses saat perangkat terhubung ke WiFi robot.
- **Dual-Transport Telemetry:** Mendukung REST API HTTP (`/api/data`, `/api/drive`, `/api/stop`, `/api/save`) dan integrasi WebSocket simulator.
- **Driver Motor DRV8833 Berperforma Tinggi:**
  - Modulasi PWM frekuensi 18 kHz (bebas dengung audiosonik).
  - *Soft-transition* akselerasi dan deselerasi untuk kestabilan traksi roda.
  - Fail-safe timeout 650 ms (motor otomatis berhenti jika komunikasi terputus).
  - Manajemen daya via pin `STBY` (sleep otomatis saat mode STOP).
- **Sensor Ultrasonik Non-Blocking:** Menggunakan interrupt pin hardware (`echoISR`) bergantian sehingga tidak membebani loop utama CPU ESP32.
- **Penyimpanan Kalibrasi Permanen (NVS/Preferences):** Parameter PWM roda kiri dan kanan tersimpan di EEPROM/NVS flash memory.

---

## 🔌 Konfigurasi Pinout Perangkat Keras

Sesuai definisi pada `src/config.h`:

| Komponen | Pin Modul | GPIO ESP32 | Tipe / Keterangan |
|---|---|---|---|
| **HC-SR04 Depan** | TRIG | `GPIO 4` | Output (Pulsa 10 µs) |
| | ECHO | `GPIO 17` | Input (Interrupt CHANGE `echoISR`) |
| **HC-SR04 Kiri** | TRIG | `GPIO 19` | Output (Pulsa 10 µs) |
| | ECHO | `GPIO 18` | Input (Interrupt CHANGE `echoISR`) |
| **HC-SR04 Kanan** | TRIG | `GPIO 14` | Output (Pulsa 10 µs) |
| | ECHO | `GPIO 13` | Input (Interrupt CHANGE `echoISR`) |
| **DRV8833 Motor Kiri** | AIN1 | `GPIO 27` | PWM Channel (18 kHz, 8-bit) |
| | AIN2 | `GPIO 26` | PWM Channel (18 kHz, 8-bit) |
| **DRV8833 Motor Kanan** | BIN1 | `GPIO 25` | PWM Channel (18 kHz, 8-bit) |
| | BIN2 | `GPIO 33` | PWM Channel (18 kHz, 8-bit) |
| **DRV8833 Standby** | STBY | `GPIO 23` | Output (Active HIGH) |
| *TCRT5000 Line Tracker (Opsional)* | DO | `GPIO 27` | *Shared header / opsional* |

---

## 📶 Koneksi WiFi & Akses Dashboard

ESP32 beroperasi dalam mode **SoftAP (Access Point)** mandiri:

- **SSID WiFi:** `MazeSolver`
- **Password:** `12345678`
- **IP Address:** `192.168.4.1`
- **Captive Portal:** Otomatis membuka peramban pada perangkat Android, iOS, dan Windows setelah terkoneksi ke WiFi.
- **URL Dashboard:** `http://192.168.4.1/`

---

## 📡 Dokumentasi Endpoint REST API

Semua endpoint menyertakan header `Access-Control-Allow-Origin: *` untuk pengujian lintas domain:

### 1. `GET /api/data`
Mengambil data telemetri jarak sensor dan status motor.
- **Respons (JSON):**
  ```json
  {
    "depan": 18.5,
    "kiri": 5.2,
    "kanan": 22.0,
    "mode": "STOP",
    "pwmKiri": 0,
    "pwmKanan": 0,
    "savedKiri": 220,
    "savedKanan": 220
  }
  ```

### 2. `POST /api/drive`
Menjalankan motor dengan mode dan nilai PWM tertentu.
- **Format:** `application/x-www-form-urlencoded`
- **Parameter:**
  - `mode`: `maju` | `mundur` | `belok_kiri` | `belok_kanan` | `uji_kiri` | `uji_kanan`
  - `l`: Nilai PWM motor kiri (`0` - `255`)
  - `r`: Nilai PWM motor kanan (`0` - `255`)
- **Contoh Payload:** `mode=maju&l=220&r=220`

### 3. `POST /api/stop`
Menghentikan seluruh motor seketika (mengaktifkan mode standby pada DRV8833).
- **Respons:** `{"ok": true}`

### 4. `POST /api/save`
Menyimpan kalibrasi nilai PWM default ke NVS flash memory ESP32.
- **Parameter:** `l=<pwm_kiri>&r=<pwm_kanan>`
- **Contoh Payload:** `l=210&r=225`

---

## 🛠️ Panduan Kompilasi & Flash Firmware

### Menggunakan PlatformIO (Disarankan)
1. Clone repositori ini:
   ```bash
   git clone git@github.com:CakUrang-dev/firmware-mazebot.git
   cd firmware-mazebot
   ```
2. Buka folder proyek di **VS Code** yang sudah terpasang ekstensi **PlatformIO IDE**.
3. Hubungkan ESP32 DevKit V1 ke port USB komputer.
4. Klik tombol **PlatformIO: Build** (`✓`) dan **PlatformIO: Upload** (`→`) pada status bar bawah.

### Konfigurasi `platformio.ini`
```ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
upload_speed = 921600
board_build.partitions = min_spiffs.csv
```

---

## 📁 Struktur Berkas

```
firmware-mazebot/
├── include/
├── lib/
├── src/
│   ├── config.h         # Pinout GPIO, konfigurasi WiFi, timer, dan konstanta PWM
│   ├── main.cpp         # Setup utama, loop pengontrolan, dan startup sequence
│   ├── motor.h / .cpp   # Driver kontroler motor DRV8833 (PWM 18 kHz, soft-transition)
│   ├── sensor.h / .cpp  # Pembacaan 3x HC-SR04 berbasis interrupt echo non-blocking
│   ├── settings.h / .cpp# Manajemen kalibrasi PWM tersimpan di Preferences/NVS
│   ├── web_page.h       # Raw HTML/CSS/JS dashboard terkompresi di memori flash (PROGMEM)
│   ├── web_server.h / .cpp # Web server HTTP, DNS captive portal, dan REST handler
├── platformio.ini       # Konfigurasi environment PlatformIO ESP32
└── README.md            # Dokumentasi teknis proyek
```

---

## 🎓 Capstone Project Context
Proyek ini merupakan bagian dari ekosistem **MazeBot EduKit** — Platform Robotika & IoT Edukatif untuk pembelajaran siswa jenjang SD, SMP, dan SMA berbasis Kurikulum Merdeka dan Capstone Design Teknik Elektro Universitas Negeri Malang.
