# 🎓 EduSense: Adaptive Seating & Peer Interaction Analyzer

<div align="center">

![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)
![Theme](https://img.shields.io/badge/theme-IoT%20Pendidikan-green.svg)
![Status](https://img.shields.io/badge/status-In%20Development-yellow.svg)
![License](https://img.shields.io/badge/license-MIT-orange.svg)

**Sistem berbasis Computer Vision dan Wearable IoT untuk menganalisis interaksi antar siswa, mendeteksi tingkat fokus secara real-time, dan menghasilkan rekomendasi rotasi bangku adaptif yang terintegrasi dengan Dapodik.**

[Tentang Proyek](#-tentang-proyek) · [Arsitektur Sistem](#-arsitektur-sistem) · [Hardware](#-spesifikasi-hardware) · [Algoritma](#-algoritma-adaptive-seating) · [Instalasi](#-instalasi) · [Roadmap](#-roadmap)

</div>

---

## 📌 Tentang Proyek

EduSense menjawab satu masalah konkret: guru tidak bisa memantau 30 siswa sekaligus. Sistem ini memasang kamera wide-angle dan gelang wearable ringan di setiap meja, lalu memproses data secara lokal di edge device (Raspberry Pi / Jetson Nano) sebelum dikirim ke dashboard guru.

Output utamanya ada dua:
1. **Heatmap real-time** — guru melihat meja mana yang "panas" (gaduh/tidak fokus) dalam hitungan detik
2. **Laporan rotasi bangku mingguan** — rekomendasi PDF otomatis berbasis korelasi nilai Dapodik dan data perilaku sensor

### Mengapa EduSense?

| Masalah | Solusi EduSense |
|---|---|
| Guru tidak tahu siapa yang mengobrol di belakang | YOLO pose + BLE gelang mendeteksi interaksi negatif secara otomatis |
| Penempatan bangku berdasarkan intuisi, bukan data | Algoritma korelasi nilai × distraksi menghasilkan saran spesifik per pasangan siswa |
| Data Dapodik tidak terhubung ke kondisi kelas | Sinkronisasi mingguan, lalu guru drag-and-drop nama ke denah meja digital |
| Server cloud mahal dan lambat untuk video | Edge inference di Raspberry Pi — cloud hanya menerima anomali, bukan raw video |

---

## 🔄 Arsitektur Sistem

### Alur Data Utama

```mermaid
flowchart TD
    subgraph SENSOR["📡 Lapisan Sensor"]
        CAM["📷 IP Camera / USB Cam\n1080p · FOV 120°\n1 kamera per 2-3 meja"]
        WEAR["⌚ Gelang ESP32-C3\nMPU6050 + MAX30102\nBLE Adv. tiap 2 detik"]
        MIC["🎤 MEMS Microphone\nOpsional · Analisis dB"]
    end

    subgraph EDGE["🖥️ Edge Gateway (RPi 5 / Jetson Nano)"]
        BLE["BLE Scanner\n30–40 gelang simultan"]
        YOLO["YOLOv8-pose\n+ DeepSORT Tracker\n15 FPS inference"]
        AUDIO["Audio Processor\ndB Threshold > 65"]
        FUSE["Data Fusion Engine\nKorelasi CV + Wearable + Audio"]
    end

    subgraph CLOUD["☁️ Backend Server"]
        MQTT["MQTT Broker\nMosquitto"]
        API["FastAPI Backend"]
        DB["PostgreSQL + PostGIS\nDenah & Time-series"]
        DAPODIK["Dapodik REST API\nRead-Only Sync"]
    end

    subgraph DASHBOARD["🖥️ Dashboard Guru"]
        HM["Heatmap Kelas\nReal-time"]
        NOTIF["Push Notification\nHP Guru"]
        REPORT["PDF Report\nRotasi Mingguan"]
        DRAG["Drag-and-Drop\nDenah Meja Digital"]
    end

    CAM -->|"Frame Video"| YOLO
    WEAR -->|"BLE Hex Data"| BLE
    MIC -->|"PCM Audio"| AUDIO

    YOLO --> FUSE
    BLE --> FUSE
    AUDIO --> FUSE

    FUSE -->|"Event Anomali Saja"| MQTT
    MQTT --> API
    API <-->|"Sinkronisasi Nilai\n1x seminggu"| DAPODIK
    API --> DB

    DB --> HM
    DB --> REPORT
    HM --> NOTIF
    DAPODIK --> DRAG
    DRAG --> DB

    style SENSOR fill:#e8f4f8,stroke:#2196F3
    style EDGE fill:#fff8e1,stroke:#FF9800
    style CLOUD fill:#f3e5f5,stroke:#9C27B0
    style DASHBOARD fill:#e8f5e9,stroke:#4CAF50
```

---

### Alur Proses Computer Vision

```mermaid
flowchart LR
    A["Frame Kamera\n1080p @ 15 FPS"] --> B["YOLOv8-pose\nKeypoint Detection"]
    B --> C["DeepSORT\nPer-student ID Tracking"]

    C --> D{Analisis\nKeypoint}

    D -->|"Jarak hidung\nmengecil < threshold\n+ bahu sinkron ≤ 0.5s"| E["✅ Interaksi Positif\n(Diskusi Tugas)"]
    D -->|"Pitch/Yaw wajah\n> 45° dari papan tulis"| F["⚠️ Distraksi Terdeteksi"]
    D -->|"Kepala turun\nstatis > 2 menit"| G["😴 Mengantuk / Bosan"]
    D -->|"Tidak ada gerakan\nwajah & bahu stabil"| H["📝 Fokus Menulis / Diam"]

    E --> I["Interaction Label:\n'Diskusi Tugas'"]
    F --> J["Visual Distraction Score:\nWaktu_tidak_fokus /\nDurasi_pelajaran × 100%"]
    G --> K["Trigger: Saran Metode\nPengajaran ke Guru"]
    H --> L["Interaction Label:\n'Diam / Fokus'"]

    I --> M["Output per Meja:\n• interaction_duration_sec\n• focus_percentage\n• interaction_label"]
    J --> M
    K --> M
    L --> M
```

---

### Alur Proses Wearable (Gelang)

```mermaid
flowchart TD
    A["⌚ Gelang ESP32-C3\nBLE Advertising tiap 2 detik\nFormat: Hex String"] --> B["BLE Scanner\nRaspberry Pi / Jetson"]

    B --> C["Parse Hex Data\nAcc + Gyro + HR + HRV"]

    C --> D["Hitung RMSSD\n(HRV Analysis)"]
    C --> E["Hitung Motion Count\nper 10 menit"]

    D --> F{Interpretasi HRV\n+ Gerakan}

    F -->|"HRV tinggi\nGerakan rendah"| G["🧠 Kognitif Loading\n(Fokus Berpikir)"]
    F -->|"HRV tinggi\nGerakan tinggi"| H["😰 Stres / Interaksi\nSosial Berlebihan"]
    F -->|"HRV rendah\nGerakan rendah"| I["😴 Under-stimulated\n(Bosan / Ngantuk)"]

    E --> J{Level Gerakan}
    J -->|"< 5x / 10 menit"| K["🟢 Level 1 - Normal"]
    J -->|"5–10x / 10 menit"| L["🟡 Level 2 - Waspada"]
    J -->|"> 15x / 10 menit"| M["🔴 Level 3 - Batas Toleransi"]

    G --> N["Gabung ke\nData Fusion Engine"]
    H --> N
    I --> N
    K --> N
    L --> N
    M -->|"Trigger Alert"| O["🔔 Push Notifikasi Guru\n'Meja X: Andi & Budi\nmelebihi batas 3 menit'"]
    M --> N
```

---

### Alur Algoritma Adaptive Seating

```mermaid
flowchart TD
    A["Data Mingguan Terkumpul"] --> B["Ambil Data Dapodik\nNilai Harian + Nilai Semester"]
    A --> C["Ambil Data Sensor\nDistraksi + Interaksi"]

    B --> D["Hitung Delta Nilai\nPerubahan nilai siswa\nsaat duduk dengan siswa X"]
    C --> E["Hitung Indeks Distraksi\nRata-rata durasi interaksi\nnegatif per jam"]

    D --> F["Formula Skor Bangku:\nSkor = 0.4 × ΔNilai\n      − 0.6 × IndeksDistraksi"]
    E --> F

    F --> G{Skor Bangku}

    G -->|"Skor Positif\n(ΔNilai tinggi,\ndistraksi rendah)"| H["✅ Pertahankan Bangku\nPeer Tutoring Efektif"]
    G -->|"Skor Negatif\n(Nilai turun,\ndistraksi tinggi)"| I["🔄 Rotasi Pisah\nGanggu Fokus"]
    G -->|"Skor Netral\n(tidak signifikan)"| J["👨‍🏫 Intervensi Guru\nBerikan Tugas Kolaboratif"]

    H --> K["Generate PDF Report\nSetiap Akhir Pekan"]
    I --> K
    J --> K

    K --> L["Guru Review\nDrag-and-Drop Denah Baru"]
    L --> M["Rotasi Bangku\nDiimplementasikan"]
    M --> A
```

---

### Alur Integrasi Dapodik

```mermaid
sequenceDiagram
    participant G as 👨‍🏫 Guru
    participant D as Dashboard EduSense
    participant A as FastAPI Backend
    participant DB as PostgreSQL
    participant DAP as Dapodik API

    G->>D: Login & Pilih Kelas
    D->>A: Request sinkronisasi data siswa
    A->>DAP: GET /siswa?rombel=X (Read-Only)
    DAP-->>A: NISN, Nama, Nilai Harian, Nilai Semester
    A->>DB: Simpan / Update data siswa
    DB-->>D: Daftar nama siswa tersedia

    G->>D: Drag-and-drop nama ke denah meja
    D->>DB: Simpan posisi duduk terbaru

    Note over A,DB: Proses berlangsung tiap minggu atau manual

    A->>DB: Jalankan Peer Performance\nCorrelation Model
    DB-->>A: Skor bangku per pasangan siswa
    A->>D: Kirim rekomendasi rotasi
    D->>G: Tampilkan PDF laporan\n+ denah rotasi baru
```

---

## 🔧 Spesifikasi Hardware

### Kamera

| Parameter | Detail |
|---|---|
| Tipe | Wide-angle USB Camera 1080p / IP Camera RTSP |
| Field of View | Minimal 120° horizontal |
| Penempatan | Depan kelas, sudut 45° menghadap meja |
| Cakupan | 1 kamera per 2–3 meja (4–6 siswa) |
| Rekomendasi | Logitech C930e atau ESP32-CAM |

### Gelang Wearable

| Parameter | Detail |
|---|---|
| MCU | ESP32-C3 Mini (BLE 5.0) |
| Sensor Gerak | MPU6050 — Accelerometer & Gyroscope |
| Sensor Biometrik | MAX30102 — Heart Rate & HRV |
| Baterai | Li-Po 200mAh (tahan 8 jam) |
| Transmisi | BLE Advertising tiap 2 detik |
| Format Data | Hexadecimal String (ringkas) |

### Gateway (Edge Device)

| Parameter | Detail |
|---|---|
| Hardware | Raspberry Pi 5 4GB atau NVIDIA Jetson Nano |
| Inference | YOLOv8-pose real-time @ 15 FPS |
| BLE Capacity | Scan 30–40 gelang simultan |
| Upload Strategi | Event-based — hanya kirim anomali ke cloud |
| Audio (Opsional) | MEMS Microphone untuk analisis kebisingan (dB) |

---

## 🧠 Model Computer Vision

```
Model   : YOLOv8-pose (Keypoint Detection) + YOLOv8-face (Head Pose)
Tracker : DeepSORT (ID unik per siswa, persisten antar frame)
```

**Keypoint yang dianalisis:**
- `nose` — orientasi wajah (arah pandang)
- `left_shoulder` + `right_shoulder` — postur dan sinkronisasi gerak

**Logika deteksi perilaku:**

| Kondisi | Label | Kalkulasi |
|---|---|---|
| Jarak Euclidean hidung mengecil + bahu sinkron ≤ 0.5s | Diskusi Tugas | Geometri real-time |
| Pitch/Yaw wajah > 45° dari papan tulis | Distraksi | Head pose estimation |
| Kepala turun, HRV rendah, gerak minimal | Mengantuk | Fusi CV + Wearable |
| Wajah stabil menghadap depan | Fokus | Default state |

**Output per meja:**
```json
{
  "desk_id": "M3",
  "interaction_duration_seconds": 142,
  "focus_percentage": 67.4,
  "interaction_label": "Mengobrol"
}
```

---

## 💡 Analisis Wearable (HRV + Gerak)

### Interpretasi HRV (Metode RMSSD)

| Kondisi HRV | Kondisi Gerak | Interpretasi |
|---|---|---|
| Tinggi | Rendah | 🧠 Kognitif Loading — Fokus berpikir dalam |
| Tinggi | Tinggi | 😰 Stres / Interaksi sosial berlebihan |
| Rendah | Rendah | 😴 Under-stimulated — Bosan atau mengantuk |

### Level Ambang Gerakan

| Level | Threshold | Status |
|---|---|---|
| 🟢 Level 1 | < 5 gerakan / 10 menit | Normal |
| 🟡 Level 2 | 5–10 gerakan / 10 menit | Waspada |
| 🔴 Level 3 | > 15 gerakan / 10 menit | Batas toleransi — trigger alert |

> **Korelasi Audio:** Jika kebisingan kelas > 65 dB **dan** gelang bergetar secara bersamaan → sistem menandai kelas sebagai gaduh.

---

## 📊 Algoritma Adaptive Seating

```
Skor_Bangku = (0.4 × Delta_Nilai) − (0.6 × Indeks_Distraksi)
```

| Variabel | Sumber Data | Keterangan |
|---|---|---|
| `Delta_Nilai` | Dapodik | Perubahan nilai mingguan siswa saat duduk bersama siswa X |
| `Indeks_Distraksi` | YOLO + Gelang | Rata-rata durasi interaksi negatif per jam |

**Output rekomendasi:**

| Hasil Skor | Rekomendasi | Keterangan |
|---|---|---|
| Positif | ✅ Pertahankan Bangku | Peer tutoring berlangsung efektif |
| Negatif | 🔄 Rotasi Pisah | Keberadaan teman ini menurunkan fokus |
| Netral | 👨‍🏫 Intervensi Guru | Berikan tugas kolaboratif terstruktur |

Laporan PDF digenerate otomatis setiap akhir pekan, berisi saran rotasi spesifik per pasangan meja.

---

## 🖥️ Fitur Dashboard

### Real-time Heatmap Kelas

```
🔵 Biru   → Fokus   (HRV stabil, wajah menghadap depan)
🟠 Oranye → Waspada (1 siswa tidak fokus / gelisah)
🔴 Merah  → Kritis  (Batas toleransi bercanda terlampaui)
```

Notifikasi langsung ke HP guru:
> *"Meja 3: Andi & Budi melebihi batas interaksi 3 menit."*

### Laporan Mingguan (PDF)

- **Saran Pisah:** Nama siswa + alasan (Penurunan nilai −X%, Gelisah Y%)
- **Saran Gabung:** Nama siswa + alasan (Pola HRV sinkron, nilai naik bersama)
- **Evaluasi Metode Mengajar:** Jika >50% wajah menunduk dan HRV rendah → *"Ganti metode ceramah dengan ice breaking."*

---

## 🛠️ Tech Stack

```
Backend     : Python 3.11 + FastAPI + MQTT (Mosquitto)
Frontend    : React.js + Leaflet.js (denah interaktif)
Database    : PostgreSQL + PostGIS
ML / CV     : PyTorch / TensorFlow Lite (edge inference)
Tracking    : YOLOv8-pose + DeepSORT
Deployment  : Docker Compose
```

### Struktur Direktori

```
edusense/
├── edge/                       # Kode untuk Raspberry Pi / Jetson Nano
│   ├── cv_pipeline/
│   │   ├── yolo_inference.py   # YOLOv8-pose inference
│   │   ├── deepsort_tracker.py # DeepSORT tracking
│   │   └── behavior_logic.py   # Logika distraksi & interaksi
│   ├── ble_scanner/
│   │   ├── scanner.py          # BLE Scan 30–40 gelang
│   │   └── hrv_processor.py    # Kalkulasi RMSSD
│   └── data_fusion.py          # Fusi data CV + Wearable + Audio
│
├── backend/                    # FastAPI server
│   ├── main.py
│   ├── routers/
│   │   ├── dapodik.py          # Sinkronisasi Dapodik
│   │   ├── seating.py          # Algoritma rotasi bangku
│   │   └── reports.py          # Generate PDF
│   └── models/                 # SQLAlchemy models
│
├── firmware/                   # Kode ESP32-C3 gelang
│   ├── main.ino
│   ├── mpu6050_reader.h
│   └── max30102_reader.h
│
├── frontend/                   # React.js dashboard
│   ├── src/
│   │   ├── components/
│   │   │   ├── ClassHeatmap.jsx
│   │   │   ├── SeatDragDrop.jsx
│   │   │   └── WeeklyReport.jsx
│   │   └── App.jsx
│
├── docker-compose.yml
└── README.md
```

---

## 🚀 Instalasi

### Prasyarat

- Python 3.11+
- Node.js 20+
- Docker & Docker Compose
- Raspberry Pi 5 / Jetson Nano (untuk edge)
- PostgreSQL 15+ dengan ekstensi PostGIS

### Clone & Setup

```bash
git clone https://github.com/username/edusense.git
cd edusense
```

### Jalankan dengan Docker Compose

```bash
# Copy file environment
cp .env.example .env

# Edit konfigurasi (database, MQTT, Dapodik API key)
nano .env

# Build dan jalankan semua service
docker-compose up --build
```

### Setup Edge Device (Raspberry Pi)

```bash
# Install dependensi
cd edge
pip install -r requirements.txt

# Install YOLOv8
pip install ultralytics

# Jalankan pipeline
python main.py --camera rtsp://192.168.1.100:554/stream --ble-scan
```

### Flash Firmware Gelang

```bash
# Buka firmware/ di Arduino IDE
# Pilih board: ESP32-C3 Dev Module
# Upload main.ino ke ESP32-C3
```

### Akses Dashboard

```
URL     : http://localhost:3000
API     : http://localhost:8000/docs
MQTT    : mqtt://localhost:1883
```

---

## 📅 Roadmap

```mermaid
gantt
    title EduSense Development Roadmap
    dateFormat  YYYY-MM-DD
    section Phase 1 · PoC
    Setup kamera + YOLOv8 (1 meja)     :p1a, 2024-01-01, 14d
    Uji akurasi deteksi perilaku        :p1b, after p1a, 16d

    section Phase 2 · Wearable
    Prototipe gelang ESP32-C3           :p2a, after p1b, 30d
    Kalibrasi accelerometer             :p2b, after p2a, 30d

    section Phase 3 · Dapodik & Analytics
    Simulasi sinkronisasi Dapodik       :p3a, after p2b, 15d
    Validasi rumus korelasi nilai       :p3b, after p3a, 15d

    section Phase 4 · Piloting Sekolah
    Deploy 1 kelas (30 siswa)           :p4a, after p3b, 90d
```

### Target Metrik Keberhasilan (Phase 4)

| Metrik | Target |
|---|---|
| Penurunan laporan "siswa rame sendiri" | −30% |
| Peningkatan nilai rata-rata kelas setelah rotasi AI | +10% |
| Latensi notifikasi guru | < 5 detik |
| Akurasi deteksi perilaku (CV) | > 85% |

---

## 🔐 Privasi & Etika

- Data video **tidak pernah dikirim ke cloud** — inference berjalan sepenuhnya di edge device
- Cloud hanya menerima **event anomali terstruktur** (JSON), bukan raw video
- Akses Dapodik bersifat **Read-Only** — sistem tidak mengubah atau menulis data Dapodik
- Seluruh data siswa dienkripsi at-rest di PostgreSQL
- Gelang wearable tidak menyimpan data — semua pemrosesan di gateway

---

## 🤝 Kontribusi

1. Fork repositori ini
2. Buat branch fitur: `git checkout -b feature/nama-fitur`
3. Commit perubahan: `git commit -m 'feat: tambah fitur X'`
4. Push ke branch: `git push origin feature/nama-fitur`
5. Buka Pull Request

---

## 📄 Lisensi

Proyek ini berlisensi MIT. Lihat file [LICENSE](LICENSE) untuk detail lengkap.

---

<div align="center">

Dibuat untuk membantu guru Indonesia membuat keputusan berbasis data, bukan asumsi.

**EduSense · IoT Pendidikan · v1.0.0**

</div>
