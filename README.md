# SMS Forwarder to Telegram & Web Gateway (ESP32-S3 + SIM800L + Laravel 12)

Sistem IoT **SMS Forwarder to Telegram** berbasis mikrokontroler **ESP32-S3**, modul seluler **SIM800L**, dan backend **Laravel 12 (PHP 8.3)**. Sistem ini dirancang khusus untuk meneruskan pesan SMS masuk (seperti kode verifikasi OTP, notifikasi perbankan, dan pesan darurat) secara *real-time* langsung ke **Telegram (Pribadi, Grup, atau Channel)** dengan efisiensi daya tinggi (*eco-standby*), ketahanan fail-safe (*offline retry queue*), serta dilengkapi **Web Admin Dashboard** dan **Two-Way Telegram Bot Command Center**.

---

## Fitur Utama

### 1. Real-Time SMS Forwarder ke Telegram (Fitur Utama)
- **Dual-Mode Forwarding Engine (Hybrid Architecture)**:
  - **Mode 1 (Direct ESP32-to-Telegram HTTPS)**: ESP32 mem-forward SMS langsung ke Telegram Bot API via koneksi HTTPS terenkripsi TLS/SSL (dilengkapi validasi Root CA). Berjalan mandiri (*standalone*) tanpa bergantung pada server backend.
  - **Mode 3 (Laravel Backend Relay)**: Saat data SMS masuk disimpan ke database REST API Laravel, backend secara otomatis meneruskan notifikasi SMS ke Telegram menggunakan `TelegramService`.
- **Format Notifikasi Pesan Terstruktur**: Pesan Telegram diformat menggunakan HTML rapi yang memuat nomor pengirim, nama perangkat gateway, waktu penerimaan, dan isi pesan lengkap.
- **Fail-Safe Offline Retry Queue**: Apabila jaringan WiFi terputus atau API Telegram sedang tidak dapat dijangkau, SMS tidak akan hilang. Data ditampung dalam antrean memori RAM dan dikirim ulang secara otomatis saat koneksi internet kembali normal (hingga 5 kali percobaan).
- **Two-Way Telegram Bot Command Center**: Memantau dan mengontrol gateway dari jarak jauh kapan saja langsung dari ruang percakapan bot Telegram:
  - `/status` : Informasi performa perangkat (frekuensi CPU, suhu chip, alokasi RAM, operator GSM, sinyal CSQ/dBm, status SIM card, tegangan modem, alamat IP, dan uptime).
  - `/signal` atau `/csq` : Pengecekan kuat sinyal GSM (skala 0-31 CSQ, estimasi desibel dBm, kategori kualitas sinyal, dan status registrasi jaringan).
  - `/voltage` : Pengecekan tegangan suplai daya modul SIM800L secara langsung.
  - `/sync_sms` : Penarikan paksa seluruh SMS yang masih tersimpan di memori kartu SIM (`AT+CMGL="ALL"`).
  - `/restart_gsm` : Re-inisialisasi ulang modem GSM SIM800L tanpa merestart chip ESP32.
  - `/reboot` : Restart hardware chip mikrokontroler ESP32 dari jarak jauh.
  - `/help` : Panduan daftar seluruh perintah bot.
- **Whitelist Otorisasi Chat ID**: Hanya akun atau grup Telegram dengan Chat ID terdaftar yang diizinkan mengeksekusi perintah bot; perintah dari Chat ID asing otomatis ditolak (*Access Denied*).

### 2. Keamanan Admin Dashboard (Laravel 12 Session Hardened)
- Autentikasi Admin berbasis sesi dengan proteksi **Anti Brute-Force Rate Limiting** (maksimal 5 percobaan gagal per menit).
- Proteksi **Anti Session-Fixation** (`session()->regenerate()`) dan validasi token **CSRF** pada seluruh formulir serta aksi logout.
- Hak akses eksklusif untuk admin terdaftar (seeder awal).

### 3. Device Authentication & REST API Gateway (IoT Endpoints)
- Keamanan otorisasi REST API modul ESP32 berbasis **Device Token** unik (HTTP Header `X-Device-Token`, Bearer, atau JSON Body).
- Middleware `EnsureValidDeviceToken` yang memvalidasi setiap request IoT.
- Endpoint Heartbeat status perangkat (`POST /api/device/heartbeat`).
- Endpoint Respon eksekusi AT Command (`POST /api/device/command-response`).
- Endpoint Penyimpanan & Ingest SMS Masuk (`POST /api/sms`).

### 4. Pemantauan Status SIM Card & Jaringan Real-Time
- Status Kartu SIM: `READY`, `UNKNOWN`, atau `SIM PIN`.
- Status Registrasi Jaringan Seluler (`AT+CREG?`): `+CREG: 0,1` (Registered Home), `+CREG: 0,5` (Registered Roaming), `+CREG: 0,2` (Searching).
- Kuat Sinyal SIM800L (CSQ 0-31, estimasi dBm, dan kategori: Sangat Baik, Baik, Cukup, Lemah).
- Operator Seluler terdeteksi (`AT+COPS?`).
- Last Seen Activity Tracker dan status Online/Offline perangkat.

### 5. Penarikan Pesan dari Memori SIM (SIM Storage Sync)
- Tombol **Tarik Pesan dari SIM** pada Web Admin Dashboard dan perintah bot Telegram `/sync_sms`.
- Mekanisme otomatis membaca seluruh SMS tersimpan di memori SIM (`AT+CMGL="ALL"`) saat modul pertama kali tersambung maupun atas permintaan jarak jauh.

### 6. Interactive GSM Live Console & AT Terminal
- Eksekusi AT Command secara remote dari Web Dashboard langsung ke modul SIM800L.
- Tombol pintas AT Command instan: `SYNC_SIM_SMS`, `AT+CMGL="ALL"`, `AT+CPMS?`, `AT+CREG?`, `AT+CSQ`, `AT+CPIN?`, `AT+COPS?`, `AT+CBC`.

### 7. On-Chip Web Captive Portal & NVS Manager (Mode 2)
- Konfigurasi parameter perangkat tanpa perlu mengunggah ulang (*flash*) program: tahan tombol BOOT selama 3 detik untuk mengaktifkan Access Point mandiri.
- Antarmuka web lokal di `192.168.4.1` untuk mengatur SSID, Password WiFi, Telegram Bot Token, Telegram Chat ID, URL API Laravel, dan Device Token.
- Seluruh konfigurasi tersimpan permanen di memori Non-Volatile Storage (NVS) Flash chip ESP32.
- Dilengkapi tombol pengujian pengiriman pesan ke Telegram langsung dari portal web.

### 8. Layar Monitoring OLED SSD1306 0.96 Inci
- Menampilkan informasi visual status WiFi, alamat IP, kekuatan sinyal GSM, operator seluler, status SIM, konektivitas Telegram & Laravel, serta voltase modem.
- Pop-up notifikasi visual saat ada SMS baru masuk.
- **Smart Sleep Timer**: Layar otomatis mati setelah 15 detik untuk mencegah *burn-in* dan menghemat konsumsi daya; layar bangun kembali saat SMS masuk atau ketika tombol BOOT ditekan 1 kali.

---

## Struktur Folder Proyek

```text
app/
├── Http/
│   ├── Controllers/
│   │   ├── Api/
│   │   │   ├── DeviceController.php     # Endpoint POST /api/device/heartbeat & /api/device/command-response
│   │   │   └── SmsController.php        # Endpoint POST /api/sms
│   │   └── Web/
│   │       ├── AuthController.php       # Controller Login, Logout & Rate Limiting
│   │       ├── DashboardController.php  # Controller halaman utama Dashboard
│   │       ├── DeviceController.php     # CRUD Perangkat ESP32, syncSimSms & Live AT Console
│   │       └── SmsController.php        # List, Detail, Toggle, Delete SMS & syncFromSim
│   ├── Middleware/
│   │   └── EnsureValidDeviceToken.php   # Otorisasi token device pada API ESP32
│   ├── Requests/
│   │   ├── Api/
│   │   │   ├── DeviceHeartbeatRequest.php
│   │   │   └── StoreSmsRequest.php
│   │   └── Web/
│   │       ├── StoreDeviceRequest.php
│   │       └── UpdateDeviceRequest.php
│   └── Resources/
│       ├── DeviceResource.php           # API Transformer untuk Device
│       └── SmsResource.php              # API Transformer untuk SMS
├── Models/
│   ├── Device.php                       # Model Device & Relasi
│   ├── Sms.php                          # Model SMS, Scope Search & Filter
│   └── User.php                         # Model Admin User
└── Services/
    ├── DeviceService.php                # Service Logic Device, Heartbeat & AT Queue
    ├── SmsService.php                   # Service Logic SMS, Storage & Telegram Trigger
    └── TelegramService.php              # Service Notifikasi Telegram (HTTP Client)

include/
├── config.h                             # Konfigurasi Firmware ESP32 (WiFi, Token, Pinout)
└── config.h.example                     # Template Konfigurasi Firmware

src/
└── main.cpp                             # Firmware ESP32-S3: Tri-Mode Orchestrator & FreeRTOS Dual-Core

lib/
├── ApiClient/                           # HTTP Client ESP32 untuk REST API Laravel (Mode 3)
├── DisplayManager/                      # Controller Layar OLED SSD1306 & Auto-Sleep
├── GSMManager/                          # Driver SIM800L: AT State Machine, RI Interrupt & Sleep
├── TelegramClient/                      # Direct Telegram HTTPS Client (Mode 1 Standalone)
└── WifiManagerCustom/                   # On-Chip Web Portal (Mode 2) & NVS Storage Manager

database/
├── migrations/
│   ├── 0001_01_01_000000_create_users_table.php
│   ├── 2026_08_07_000001_create_devices_table.php
│   ├── 2026_08_07_000002_create_sms_table.php
│   └── 2026_08_07_000003_add_gsm_debug_columns_to_devices_table.php
└── seeders/
    ├── DatabaseSeeder.php
    ├── DeviceSeeder.php
    ├── SmsSeeder.php
    └── UserSeeder.php                   # Default Admin Seeder

resources/views/
├── auth/
│   └── login.blade.php                  # Halaman Login Admin
├── layouts/
│   └── app.blade.php                    # Master Layout Bootstrap 5, Sidebar & Navbar Profile
├── dashboard.blade.php                  # Dashboard Statistik & Chart
├── devices/
│   └── index.blade.php                  # CRUD Devices, Modal Token & Live AT Console
└── sms/
    └── index.blade.php                  # Inbox SMS, Search, Filter & Modal Detail

routes/
├── api.php                              # REST API Routes (Khusus ESP32 dengan token)
└── web.php                              # Web Dashboard Routes (Terproteksi auth)
```

---

## Skema Hardware, Pinout & Pengkabelan (ESP32-S3 + SIM800L)

### 1. Tabel Pinout ESP32-S3 ke Modul GSM & Komponen

| Komponen | Pin Modul | Pin ESP32-S3 | Tipe Sinyal | Keterangan & Peran Teknis |
| :--- | :--- | :--- | :--- | :--- |
| **SIM800L** | `TXD` | `GPIO 16` (RX2) | Input Digital | Menerima data serial UART dari SIM800L ke ESP32 (Baud rate: 9600) |
| **SIM800L** | `RXD` | `GPIO 17` (TX2) | Output Digital | Mengirim perintah AT serial UART dari ESP32 ke SIM800L |
| **SIM800L** | `VCC` | Step-Down Buck | Catu Daya 3.8V - 4.2V | Wajib menggunakan regulator daya eksternal independen (Arus puncak 2A) |
| **SIM800L** | `GND` | Common `GND` | Ground Bersama | Wajib disambungkan bersama ke pin GND ESP32 |
| **SIM800L** | `RST` | `GPIO 7` | Output (Active LOW) | **Hardware Reset Watchdog**: Memicu reset fisik 120ms jika modem tidak merespons |
| **SIM800L** | `DTR` | `GPIO 6` | Output (Control) | **Sleep/Wake Control**: HIGH = Modem tidur (`AT+CSCLK=1`), LOW = Modem bangun |
| **SIM800L** | `RING` / `RI` | `GPIO 5` | Input (Interrupt) | **SMS Hardware Interrupt**: Pulsa LOW saat ada SMS masuk (Event-Driven) |
| **OLED SSD1306** | `SDA` | `GPIO 8` | I2C Data | Jalur data komunikasi layar monitoring 0.96 inci |
| **OLED SSD1306** | `SCL` | `GPIO 9` | I2C Clock | Jalur clock komunikasi layar monitoring 0.96 inci |
| **OLED SSD1306** | `VCC` / `GND` | `3.3V` / `GND` | Power | Mengambil suplai 3.3V langsung dari pin board ESP32 |
| **Tombol BOOT** | `BOOT` | `GPIO 0` | Input (Pull-Up) | Klik 1x: Wake-up OLED; Tahan 3 detik: Masuk Web Captive Portal |

---

### 2. Penjelasan Mendalam Pin Khusus SIM800L (RI, DTR, RST)

1. **Pin RI (Ring Indicator / GPIO 5 - Hardware Interrupt)**:
   - **Tujuan**: Mencegah pemborosan daya dan latensi dari proses *busy-loop polling* berulang.
   - **Prinsip Operasi**: Ketika SIM800L mendeteksi SMS baru yang masuk dari jaringan seluler, pin RI akan ditarik ke level tegangan **LOW** selama kurang lebih 120 milidetik. ESP32 menangkap transisi sinyal ini menggunakan interrupt internal mikrokontroler (`attachInterrupt` mode `FALLING`).
   - **Keunggulan**: Sistem bereaksi secara *event-driven*. ESP32 dapat langsung membaca isi pesan tanpa perlu terus-menerus mengirimkan perintah `AT+CMGL` setiap beberapa detik, sehingga SMS langsung diteruskan ke Telegram dalam hitungan detik.

2. **Pin DTR (Data Terminal Ready / GPIO 6 - Sleep & Wake Control)**:
   - **Tujuan**: Mengatur mode hemat energi modem menggunakan fitur standar `AT+CSCLK=1`.
   - **Prinsip Operasi**:
     - **DTR HIGH**: Modul SIM800L beralih ke mode *deep sleep*. Konsumsi arus modem turun dari normal ~25 mA menjadi **kurang dari 1.5 mA**, sementara sirkuit penerima sinyal seluler tetap siaga.
     - **DTR LOW**: Membangunkan modul SIM800L dalam waktu sekitar 50 milidetik sebelum ESP32 mengeksekusi instruksi AT (seperti pembacaan SMS, penghapusan pesan, atau pengecekan sinyal).

3. **Pin RST (Reset / GPIO 7 - Auto-Recovery Hardware Watchdog)**:
   - **Tujuan**: Pemulihan otomatis (*self-healing*) jika modem GSM mengalami *freeze* atau *unresponsive*.
   - **Prinsip Operasi**: Firmware ESP32 memantau respon komunikasi AT secara berkala. Apabila terjadi kegagalan komunikasi serial berturut-turut sebanyak 3 kali, ESP32 secara otomatis menarik pin RST ke level LOW selama 120 milidetik untuk melakukan restart fisik pada modul SIM800L, kemudian mere-inisialisasi modem kembali ke kondisi siap pakai secara otomatis.

---

### 3. Ketentuan Catu Daya (Penting)
Modul seluler SIM800L memerlukan lonjakan arus seketika (*burst current*) hingga **2 Ampere** saat melakukan registrasi jaringan seluler atau saat menerima dan memproses SMS.
- Jangan memberi suplai tegangan SIM800L langsung dari pin 5V atau 3.3V board ESP32 (akan mengakibatkan penurunan tegangan/brownout dan mikrokontroler restart berulang).
- Gunakan modul Step-Down Buck Converter (seperti LM2596 atau MP1584) yang diatur pada tegangan keluaran stabil **4.0 Volt (rentang kerja aman: 3.8V hingga 4.2V)**.
- Pasang kapasitor elektrolit berkapasitas **1000 uF hingga 2200 uF (Low ESR)** secara paralel pada pin `VCC` dan `GND` sedekat mungkin dengan modul SIM800L untuk meredam drop tegangan saat transmisi RF.
- Seluruh jalur ground (GND sumber daya, GND modul SIM800L, dan GND board ESP32) wajib dihubungkan bersama (*Common Ground*).

---

## Power Management, FreeRTOS Dual-Core & Arsitektur Sleep/Awake

Sistem dirancang untuk operasional stabil 24 jam non-stop dengan konsumsi daya rendah dan suhu kerja chip yang dingin melalui pemisahan task FreeRTOS pada kedua inti prosesor ESP32-S3.

### 1. Manajemen Tugas Dual-Core (FreeRTOS Architecture)

Prosesor ESP32-S3 memiliki dua inti eksekusi independen (Core 0 dan Core 1). Pembagian beban kerja dirancang terpisah agar proses jabat tangan TLS/SSL internet tidak mengganggu komunikasi serial hardware:

```text
┌──────────────────────────────────────────┐      ┌──────────────────────────────────────────┐
│             CORE 0 (FreeRTOS)            │      │            CORE 1 (Main Loop)            │
├──────────────────────────────────────────┤      ├──────────────────────────────────────────┤
│ • Task: TelegramBotTask (Priority 1)     │      │ • GSM Engine: Serial UART SIM800L        │
│ • Stack Size: 8 KB (Buffer HTTPS TLS)    │      │ • Event Interrupt: Pin RI (SMS Masuk)    │
│ • Menjalankan polling perintah Telegram  │      │ • State Machine AT Command & Parsing     │
│   (/status, /signal, /voltage, dll.)     │      │ • Layar OLED SSD1306 & Auto-Sleep Timer  │
│   setiap 2.5 detik via HTTPS non-block   │      │ • Web Captive Portal (WifiManagerCustom) │
│ • Validasi Whitelist Chat ID Pengirim    │      │ • Ingest REST API Laravel & Retry Queue  │
└─────────────────────┬────────────────────┘      └────────────────────┬─────────────────────┘
                      │                                                │
                      └──────────────► [ gsmMutex ] ◄──────────────────┘
                            Sinkronisasi Thread-Safe Modul GSM
```

- **Core 0 (Background Asynchronous Worker)**:
  - Menjalankan background task `TelegramBotTask` (`xTaskCreatePinnedToCore` dengan stack 8 KB).
  - Melakukan polling perintah masuk dari Telegram Bot (`handleTelegramBotCommands`) secara berkala setiap 2.5 detik melalui koneksi HTTPS TLS/SSL tanpa memblokir siklus loop utama di Core 1.
- **Core 1 (Hardware Engine & Main Loop)**:
  - Menjalankan fungsi utama `loop()`.
  - Mengelola driver GSM SIM800L, menangkap trigger interrupt pin RI, memperbarui tampilan visual layar OLED, menangani Web Captive Portal, serta mengirimkan SMS ke Telegram dan Laravel.
- **Sinkronisasi Thread-Safe (gsmMutex)**:
  - Menggunakan FreeRTOS Mutex (`xSemaphoreCreateMutex()`) untuk mengamankan akses serial SIM800L dan variabel telemetri bersama. Mencegah bentrokan data (*race condition*) ketika Core 0 mengeksekusi pembacaan status (seperti `/signal` atau `/voltage`) saat Core 1 sedang memproses SMS masuk.
  - Mutex dilepas (`xSemaphoreGive`) saat ESP32 melakukan request jaringan HTTP/HTTPS agar komunikasi serial tidak terhambat.

---

### 2. Mekanisme Sleep & Awake (Eco-Standby 24/7)

Untuk memastikan konsumsi energi minimal dan suhu chip tetap rendah:

1. **Penurunan Frekuensi Clock CPU (80 MHz)**:
   - Frekuensi CPU ESP32-S3 diturunkan dari 240 MHz menjadi **80 MHz** saat boot (`setCpuFrequencyMhz(80)`). Clock 80 MHz sangat memadai untuk komputasi FreeRTOS, mbedTLS, dan serial GSM, namun secara signifikan menurunkan disipasi panas chip.
2. **Protokol WiFi Modem-Sleep (DTIM)**:
   - Mengaktifkan `WiFi.setSleep(WIFI_PS_MIN_MODEM)`. Modul radio WiFi tidur di antara interval transmisi DTIM router, memangkas konsumsi arus WiFi dari ~120 mA menjadi ~15 mA.
3. **SIM800L DTR Sleep Mode**:
   - Modem dikonfigurasi ke mode hemat energi `AT+CSCLK=1`. Saat tidak ada aktivitas, pin DTR ditarik ke level HIGH sehingga konsumsi arus modem turun menjadi **kurang dari 1.5 mA**.
4. **Event-Driven Awake via Pin RI**:
   - Prosesor tidak melakukan looping polling aktif. Sinyal pulsa falling edge dari pin RI modem SIM800L secara otomatis memicu prosesor untuk segera membaca SMS baru yang masuk.
5. **OLED Smart Screen Saver & Wakeup**:
   - Layar OLED otomatis padam setelah 15 detik masa siaga untuk mencegah degradasi piksel (*burn-in*) dan menghemat daya. Layar akan bangun kembali secara otomatis saat ada SMS masuk atau saat tombol BOOT ditekan 1 kali.
6. **FreeRTOS Task Yielding (WFI - Wait For Interrupt)**:
   - Task Telegram Bot menggunakan jeda non-blocking `vTaskDelay(pdMS_TO_TICKS(2500))` dan loop utama menggunakan `delay(15)`. CPU mikrokontroler secara otomatis memasuki status *Wait For Interrupt* (WFI) saat tidak ada komputasi aktif (*zero busy-loop*).

---

### 3. Tabel Benchmark Efisiensi Daya & Suhu Chip

| Parameter Pengujian | Baseline (Tanpa Optimasi) | Kondisi Teroptimasi (Eco-Mode Aktif) | Tingkat Efisiensi |
| :--- | :--- | :--- | :--- |
| **Clock CPU ESP32-S3** | 240 MHz (Dual-Core Maksimal) | **80 MHz Standby** | Pengurangan beban clock 66% |
| **Mode Radio WiFi** | Full Power Active (~120 mA) | **Modem-Sleep DTIM (~15 mA)** | Penghematan arus radio ~87% |
| **Mode Modem SIM800L** | Active Listen (~25 mA) | **DTR Sleep Mode (< 1.5 mA)** | Penghematan daya modem ~94% |
| **Beban Siklus Core 1** | 100% Busy-Loop Polling | **FreeRTOS Yield (WFI IDLE)** | Pengurangan beban kerja CPU |
| **Deteksi SMS Masuk** | Polling AT+CMGL tiap 15 detik | **100% Event-Driven (Pin RI Interrupt)** | Zero Polling Overhead |
| **Total Konsumsi Arus Standby** | ~250 - 350 mA (~1.5 Watt) | **~35 - 65 mA (~0.25 Watt)** | **Penghematan daya total ~80%** |
| **Suhu Internal Chip (TSENS)** | 45.2 C – 46.0 C (Baseline 240 MHz) | **38.2 C (Eco-Mode 80 MHz)** | **Penurunan suhu chip ~7.8 C** |

---

## Instalasi & Konfigurasi

### 1. Kebutuhan Sistem (Prerequisites)
- **PHP 8.3** atau lebih baru
- **Composer 2.x**
- **Database**: MariaDB / MySQL / SQLite
- **PlatformIO IDE** (direkomendasikan via VS Code) atau Arduino IDE

---

### 2. Panduan Setup Telegram Bot (Token & Chat ID)

Untuk mengaktifkan pengiriman notifikasi SMS ke Telegram:

1. **Membuat Bot Telegram**:
   - Buka aplikasi Telegram dan cari akun resmi `@BotFather`.
   - Kirimkan perintah `/newbot`, lalu ikuti instruksi yang diberikan (tentukan nama tampilan bot dan username unik yang diakhiri kata `bot`).
   - Salin **HTTP API Token** yang diberikan oleh `@BotFather` (contoh: `1234567890:ABCdefGhIJKlmNoPQRsTUVwxyZ`).
2. **Mendapatkan Chat ID Tujuan**:
   - Cari bot `@userinfobot` atau `@GetIDsBot` di Telegram, lalu klik **/start**.
   - Salin angka **Id** pengguna Anda (contoh: `123456789`).
   - *Untuk Grup/Channel*: Masukkan bot Anda ke grup atau channel sebagai Administrator, lalu jalankan `@GetIDsBot` di grup tersebut untuk menyalin Group ID (angka berawalan minus, contoh: `-1001234567890`).
3. **Memulai Percakapan dengan Bot**:
   - Buka ruang obrolan dengan bot yang baru dibuat di Telegram dan klik tombol **/start** agar bot memperoleh izin mengirimkan pesan ke akun Anda.

---

### 3. Langkah Instalasi Backend Laravel 12

```bash
# 1. Clone repository proyek & masuk ke direktori
git clone <repository_url>
cd SMS_Forwarder

# 2. Install dependensi composer
composer install

# 3. Salin file environment
cp .env.example .env

# 4. Generate application key
php artisan key:generate
```

---

### 4. Konfigurasi File `.env` (Database & Telegram)

Buka file `.env` pada direktori root dan sesuaikan konfigurasi koneksi database serta kredensial Telegram:

```env
APP_NAME="SMS Gateway ESP32"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost:8000

# Konfigurasi Database (MySQL / MariaDB)
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=sms_fwd
DB_USERNAME=root
DB_PASSWORD=

# Kredensial Notifikasi Telegram Bot
TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here
TELEGRAM_CHAT_ID=your_telegram_chat_id_here
```

---

### 5. Menjalankan Database Migration & Seeder

```bash
# Jalankan migrasi tabel beserta akun admin dan device seeder bawaan
php artisan migrate:fresh --seed
```

> **Kredensial Default Akun Admin:**
> - **URL Login:** `http://127.0.0.1:8000/login`
> - **Email:** `admin`
> - **Password:** `admin`

Jalankan server lokal Laravel:
```bash
php artisan serve
```

---

### 6. Konfigurasi Firmware ESP32 (`include/config.h`)

Sebelum melakukan proses *compile* dan *upload* firmware ke board ESP32-S3:

1. Salin template file konfigurasi:
   ```bash
   cp include/config.h.example include/config.h
   ```
2. Buka `include/config.h` dan sesuaikan parameter berikut:

```cpp
// 1. Kredensial WiFi (Fallback jika memori NVS kosong)
#define WIFI_SSID             "NAMA_WIFI_ANDA"
#define WIFI_PASSWORD         "PASSWORD_WIFI_ANDA"

// 2. Kredensial Backend Laravel (Mode 3)
#define API_URL               "http://192.168.1.100:8000"           // Alamat IP Server Laravel
#define DEVICE_TOKEN          "ESP32_DEFAULT_SECRET_TOKEN_12345"    // Token yang terdaftar di tabel devices
#define DEFAULT_SERVER_SYNC_ENABLED true

// 3. Kredensial Telegram Langsung dari ESP32 (Mode 1 Standalone)
#define DEFAULT_TELEGRAM_BOT_TOKEN  "0000000000:XXXXXxXxXXxxXXxxXX0xXXX0XXxX00x0XXX"
#define DEFAULT_TELEGRAM_CHAT_ID    "0000000000"

// 4. Konfigurasi Pinout Hardware SIM800L
#define SIM800_RX             16  // ESP32 GPIO 16 terhubung ke SIM800L TX
#define SIM800_TX             17  // ESP32 GPIO 17 terhubung ke SIM800L RX
#define SIM800_BAUD           9600
#define SIM800_RST_PIN        7   // Pin Hardware Reset (Active LOW)
#define SIM800_DTR_PIN        6   // Pin Sleep/Wake Control (HIGH=Sleep, LOW=Wake)
#define SIM800_RI_PIN         5   // Pin Ring Indicator Interrupt (Active LOW)

// 5. Konfigurasi Layar OLED SSD1306 I2C
#define OLED_SDA              8
#define OLED_SCL              9
```

3. **Konfigurasi Tanpa Re-Flash (On-Chip Web Portal)**:
   Apabila perangkat sudah terpasang dan ingin mengubah pengaturan tanpa kabel data:
   - Tekan dan tahan tombol **BOOT (GPIO 0)** selama **3 detik**.
   - ESP32 akan menyalakan Access Point lokal: `ESP32-SMS-Gateway-XXXX`.
   - Sambungkan smartphone atau laptop ke WiFi tersebut, lalu buka browser ke alamat:
     `http://192.168.4.1`
   - Isi form konfigurasi (SSID, Password WiFi, Telegram Token, Chat ID, dan URL Laravel), lalu klik **Simpan & Restart**. Seluruh data disimpan permanen ke memori NVS Flash ESP32.

---

## Dokumentasi Integrasi & API

### 1. Telegram Bot Integration & Remote Commands

#### A. Format Notifikasi SMS Masuk (HTML Mode)
Pesan SMS yang diterima dari kartu SIM diteruskan secara otomatis ke Telegram dengan format tampilan HTML berikut:

```text
SMS BARU DITERIMA
━━━━━━━━━━━━━━━━━━━━
Pengirim : +6281234567890
Waktu    : 2026-09-06 21:30:00
━━━━━━━━━━━━━━━━━━━━
Isi Pesan:
Kode verifikasi OTP Anda adalah 749201. Berlaku selama 5 menit. Jangan bagikan kepada siapa pun.
```

#### B. Direct-to-Telegram HTTPS API (ESP32 Firmware)
ESP32 menggunakan `WiFiClientSecure` dengan verifikasi sertifikat Root CA resmi untuk mengirimkan payload ke endpoint Telegram Cloud API:
- **Endpoint**: `POST https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/sendMessage`
- **Headers**: `Content-Type: application/json` atau `application/x-www-form-urlencoded`
- **Payload Request**:
  ```json
  {
    "chat_id": "00000000000",
    "text": "<b>SMS BARU DITERIMA</b>\n...",
    "parse_mode": "HTML"
  }
  ```

#### C. Daftar Perintah Interaktif Telegram Bot

Admin dapat mengirimkan perintah teks langsung ke bot Telegram untuk pemantauan dan kontrol:

| Perintah Bot | Deskripsi Fungsi | Contoh Respon |
| :--- | :--- | :--- |
| `/status` | Ringkasan telemetri sistem gateway | Uptime, CPU 80MHz, suhu chip, alokasi heap RAM, sinyal CSQ/dBm, status SIM, tegangan GSM, IP WiFi |
| `/signal` atau `/csq` | Pengecekan parameter sinyal seluler | Nama operator, nilai CSQ (0-31), estimasi dBm, kategori sinyal, status CREG |
| `/voltage` | Pengecekan voltase catu daya SIM800L | Informasi tegangan real-time (contoh: `4.02 V - Normal & Stabil`) |
| `/sync_sms` | Penarikan paksa seluruh SMS dari kartu SIM | Membaca dan mem-forward seluruh SMS tersimpan via `AT+CMGL="ALL"` |
| `/restart_gsm` | Inisialisasi ulang modul GSM | Me-reset driver serial SIM800L tanpa merestart chip ESP32 |
| `/reboot` | Restart mikrokontroler ESP32 | ESP32 melakukan reboot hardware via software trigger (`ESP.restart()`) |
| `/help` | Menampilkan panduan seluruh perintah | Ringkasan panduan penggunaan perintah bot |

> **Catatan Keamanan Otorisasi**: Jika ada pesan perintah yang dikirim dari akun dengan Chat ID yang tidak terdaftar pada `TELEGRAM_CHAT_ID`, ESP32 akan menolak eksekusi dan membalas dengan pesan: `Akses Ditolak. Anda tidak memiliki izin untuk mengontrol gateway ini.`

---

### 2. Laravel REST API Endpoints (IoT Gateway)

Semua endpoint IoT pada backend Laravel mewajibkan penyertaan token perangkat pada HTTP Header `X-Device-Token` atau field JSON `token`.

#### A. Ingest & Forward Incoming SMS (`POST /api/sms`)
Dikirimkan oleh ESP32 saat menerima SMS masuk dari kartu SIM. Backend Laravel akan menyimpan data SMS ke database sekaligus memicu pengiriman notifikasi Telegram melalui `TelegramService`.

- **Headers**:
  - `Content-Type: application/json`
  - `X-Device-Token: <DEVICE_TOKEN>`
- **Request Body**:
  ```json
  {
    "token": "ESP32_DEFAULT_SECRET_TOKEN_12345",
    "phone": "+6281234567890",
    "message": "Kode OTP akun Anda adalah 492019. Jangan berikan ke siapa pun.",
    "received_at": "2026-09-06 21:30:00"
  }
  ```
- **Response Success (201 Created)**:
  ```json
  {
    "success": true,
    "message": "SMS saved successfully.",
    "data": {
      "id": 101,
      "device_id": 1,
      "device_name": "ESP32-Gateway-Node01",
      "phone": "+6281234567890",
      "message": "Kode OTP akun Anda adalah 492019. Jangan berikan ke siapa pun.",
      "received_at": "2026-09-06 21:30:00",
      "processed": false
    }
  }
  ```

---

#### B. Heartbeat Device (`POST /api/device/heartbeat`)
Dikirimkan secara berkala oleh ESP32 (setiap 10-30 detik) untuk melaporkan status kesehatan perangkat dan mengambil antrean instruksi AT Command dari Web Admin.

- **Headers**:
  - `Content-Type: application/json`
  - `X-Device-Token: <DEVICE_TOKEN>`
- **Request Body**:
  ```json
  {
    "token": "ESP32_DEFAULT_SECRET_TOKEN_12345",
    "signal": 28,
    "operator": "TELKOMSEL",
    "sim_status": "READY",
    "reg_status": "Registered Home (+CREG: 0,1)"
  }
  ```
- **Response Success (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Heartbeat received successfully.",
    "data": {
      "id": 1,
      "name": "ESP32-Gateway-Node01",
      "status": "online",
      "is_online": true,
      "signal": 28,
      "operator": "TELKOMSEL",
      "sim_status": "READY",
      "reg_status": "Registered Home (+CREG: 0,1)",
      "pending_command": null,
      "last_seen": "2026-09-06 21:30:00",
      "last_seen_human": "1 second ago"
    }
  }
  ```

---

#### C. Send Command Response (`POST /api/device/command-response`)
Dikirimkan oleh ESP32 setelah selesai mengeksekusi instruksi AT Command yang diinstruksikan oleh Web Admin.

- **Headers**:
  - `Content-Type: application/json`
  - `X-Device-Token: <DEVICE_TOKEN>`
- **Request Body**:
  ```json
  {
    "token": "ESP32_DEFAULT_SECRET_TOKEN_12345",
    "command": "AT+CSQ",
    "response": "+CSQ: 28,0\n\nOK"
  }
  ```
- **Response Success (200 OK)**:
  ```json
  {
    "success": true,
    "message": "Command response updated successfully.",
    "data": {
      "id": 1,
      "name": "ESP32-Gateway-Node01",
      "pending_command": null,
      "command_response": "+CSQ: 28,0\n\nOK",
      "command_updated_at": "2026-09-06 21:30:05"
    }
  }
  ```

---

## Automated Testing

Jalankan pengujian otomatis untuk unit test dan feature test backend Laravel:

```bash
# Menjalankan seluruh test suite
php artisan test

# Menjalankan khusus pengujian autentikasi dan rate limiting
php artisan test --filter=AuthTest
```

---

## Catatan Teknis: Koneksi HTTPS Mikrokontroler & Batasan Ngrok Free Tier

Saat menghubungkan mikrokontroler ESP32 ke backend Laravel melalui layanan tunneling publik, perhatikan karakteristik khusus **Ngrok Free Tier**:

1. **Batasan Teknis Ngrok Free Tier pada Mikrokontroler**:
   - **Rantai Sertifikat Wildcard ECDSA P-384**: Layanan Ngrok Free Tier menggunakan sertifikat wildcard dengan ukuran rantai sertifikat besar (~4.5 KB). Pustaka enkripsi mikrokontroler ESP32 (`mbedTLS`) secara default mengalokasikan buffer handshake TLS terbatas (~4 KB). Akibatnya, proses jabat tangan TLS ke domain `*.ngrok-free.dev` atau `*.ngrok-free.app` dapat mengalami kegagalan dengan kode error:
     ```text
     MBEDTLS_ERR_SSL_CONN_EOF (-29312)
     ```
   - **Halaman Peringatan Interstitial**: Ngrok free secara otomatis menyisipkan respon halaman peringatan HTML (*interstitial browser warning*) pada request pertama dari klien non-browser, yang dapat menyebabkan kegagalan parsing JSON pada firmware ESP32.
2. **Rekomendasi Operasional**:
   - **Pengujian di Jaringan Lokal**: Gunakan alamat IP lokal langsung (contoh: `http://192.168.1.xxx:8000`) pada pengaturan `API_URL`. Komunikasi HTTP lokal bebas dari beban enkripsi TLS dan bekerja 100% stabil untuk mikrokontroler.
   - **Lingkungan Produksi (Live VPS / Cloud Server)**: Gunakan domain publik dengan sertifikat SSL standar bertipe RSA 2048-bit (seperti Let's Encrypt atau DigiCert) pada VPS (Ubuntu/Nginx) atau Cloud PaaS (Railway, Render, DigitalOcean). Sertifikat SSL standar didukung secara bawaan oleh mbedTLS ESP32 tanpa kendala alokasi buffer memori.

---

## Lisensi
Proyek ini dilisensikan di bawah [MIT License](LICENSE).
