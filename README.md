# Smart Stock Control

**Kendali PID Persediaan Gudang Dua Ruangan Berbasis IoT**

Tugas Besar Internet of Things 1 — Program Studi Teknik Komputer

## Identitas Kelompok

**Kelompok 1 – Supply Chain Management (SCM)**

| No | Nama Lengkap | NIM |
|----|--------------|-----|
| 1 | Muhammad Nabil Zaedan Agesy | H1H024062 |
| 2 | Ahmadin Irfan Fauzi | H1H024060 |
| 3 | Farhan Nur Sahid | H1H024057 |
| 4 | Wendy Virtus | H1H024048 |

## Gambaran Umum

Smart Stock Control adalah sistem kendali persediaan gudang dua ruangan berbasis IoT. Ruang B menyimpan stok cadangan dalam jumlah besar dan dijaga di atas *safety stock*, sedangkan Ruang A adalah stok depan tempat barang diambil. Ketika barang diambil dan stok A turun dari *setpoint*, sistem membuka gerbang dan menjalankan konveyor secara otomatis untuk mengisi ulang A dari B, tanpa menunggu petugas.

### Latar Belakang

Pengelolaan stok secara manual cenderung terlambat satu langkah. Ada tiga masalah utama:

1. *Stockout*: barang habis saat dibutuhkan sehingga penjualan atau distribusi tertunda.
2. *Overstock*: stok menumpuk, ruang penuh, dan modal tertahan di gudang.
3. Data terlambat: stok opname manual memakan waktu dan rawan salah hitung.

### Tujuan Proyek

1. Memantau stok dua ruangan secara *real-time* dengan load cell dan sensor ultrasonik.
2. Mengotomatiskan isi ulang ruang A dari ruang B dengan kendali PID loop tertutup.
3. Melindungi ruang dari kepenuhan dan stok cadangan dari kehabisan.
4. Memungkinkan pengawasan dan kendali manual jarak jauh melalui MQTT.

## Loop Kendali

| Loop | Fungsi | Jenis |
|------|--------|-------|
| Loop 1 | Menjaga stok ruang A pada *setpoint* 12 unit (di antara ROP 6 dan Max 18) | PID |
| Loop 2 | Proteksi ruang B: menjaga fill 75% lewat bukaan pintu supplier; pintu ditutup paksa pada fill ≥ 90% | PID |
| Loop 3 | Supervisor stok: bila stok B ≤ *safety stock* (4 unit), alarm menyala, REORDER dikirim, dan isi ulang A ditahan | On-off |

## Daftar Komponen

Proyek dikembangkan dan diuji pada simulator **Wokwi**.

| No | Komponen | Fungsi | Di Simulasi Wokwi |
|----|----------|--------|-------------------|
| 1 | ESP32 | Menjalankan PID, WiFi, dan MQTT | ESP32 DevKit C V4 |
| 2 | Load cell + HX711 (2 set) | Mengukur berat tumpukan, dikonversi menjadi jumlah unit | HX711 + load cell 5 kg |
| 3 | HC-SR04 (2 buah) | Mengukur tinggi tumpukan dan fill % | HC-SR04 |
| 4 | Servo (2 buah) | Gerbang A–B dan pintu supplier | Servo, sudut = output PID |
| 5 | Motor DC TT + driver L298N/TB6612 | Konveyor, kecepatan diatur lewat PWM | Relay + LED + sabuk NeoPixel |
| 6 | OLED, LED, buzzer | Indikator di lokasi | Sama |
| 7 | MQTT (HiveMQ) | Pemantauan dan kendali jarak jauh | Sama |

> Catatan: motor DC dan driver dipakai pada alat nyata. Di simulasi, konveyor diwakili oleh relay dan LED.

### Pengolahan Data Sensor

| Sensor | Rumus | Pemrosesan |
|--------|-------|------------|
| Load cell + HX711 | `jumlah = (berat − tare) / berat_unit` | Dikalibrasi dengan `tare()` dan `set_scale()`; difilter rata-rata 5 sampel. PID memakai nilai kontinu, tampilan memakai nilai bulat. |
| Ultrasonik HC-SR04 | `fill% = (H − d) / H × 100` | Dipasang di atas rak menghadap ke bawah; difilter eksponensial 50%. Dipakai untuk kapasitas dan deteksi hampir penuh. |

**Verifikasi silang:** bila berat turun tetapi tinggi tumpukan tidak berubah, data ditandai tidak konsisten.

## Arsitektur Sistem

```
Supplier ──(pintu servo)──> Ruang B ──(gerbang + konveyor)──> Ruang A ──> Permintaan barang
                               │                                 │
                      Load cell + HX711                 Load cell + HX711
                          HC-SR04                           HC-SR04
                               └──────────────┬──────────────────┘
                                              ▼
                                  ESP32 DevKit C V4
                     (baca sensor, PID loop 1 & 2, interlock, WiFi + MQTT)
                                              │
                        ┌─────────────────────┼─────────────────────┐
                        ▼                     ▼                     ▼
               Indikator lokal         Broker MQTT (HiveMQ) <──> Operator
              (OLED, LED, buzzer)
```

### Loop Tertutup (Loop 1)

Setpoint (SP_A = 12) → Σ (error) → Pengendali PID (tiap 200 ms) → Aktuator (servo + konveyor) → Plant (unit di ruang A) → Stok A.
Gangguan berupa barang diambil; umpan balik dari load cell + HX711.

### Alur Program (Satu Siklus)

1. **Mulai** → inisialisasi (WiFi, MQTT, tare load cell)
2. Baca sensor (berat dan jarak ruang A dan B)
3. Hitung stok (filter, jumlah unit, fill %)
4. Mode AUTO? **Ya** → hitung PID (loop 1 dan 2, tiap 200 ms); **Tidak** → perintah manual via MQTT
5. Interlock keselamatan (paksa output = 0 bila B ≤ safety stock, A ≥ Max, atau ruang B penuh)
6. Gerakkan aktuator (sudut servo, relay / PWM konveyor)
7. Tampilkan dan kirim data (OLED, LED, buzzer, publish MQTT)
8. Ulangi dari langkah 2

## Mekanisme Kerja

| Tahap | Proses | Penjelasan |
|-------|--------|------------|
| 1 | Baca sensor | Load cell dan HC-SR04 pada ruang A dan B dibaca oleh ESP32. |
| 2 | Hitung stok | Data difilter, lalu diubah menjadi jumlah unit (load cell) dan fill % (ultrasonik). |
| 3 | PID | Loop 1 dan loop 2 dihitung tiap 200 ms. Output 0–100% menjadi sudut servo (0–90°) dan kecepatan sabuk. |
| 4 | Interlock | Aturan keras dievaluasi setelah PID sehingga selalu diprioritaskan. |
| 5 | Aktuator | Servo gerbang/pintu dan konveyor digerakkan sesuai output. |
| 6 | Tampil dan kirim | Status ditampilkan di OLED, LED, buzzer, dan dikirim lewat MQTT. |

### Kendali PID

```
output = Kp × e + Ki × ∫e dt + Kd × dPV/dt,   dengan e = setpoint − stok terbaca
```

| Komponen | Makna | Peran |
|----------|-------|-------|
| P (Proporsional) | Seberapa jauh stok dari target saat ini | Makin jauh, makin lebar gerbang dibuka |
| I (Integral) | Sudah berapa lama stok masih kurang | Bukaan ditambah pelan-pelan sampai stok tepat di target |
| D (Derivatif) | Seberapa cepat stok berubah | Meredam kebablasan; pada proyek ini dinolkan (Kd = 0) |

Nilai awal loop 1: **Kp = 8, Ki = 0,8, Kd = 0**.

### Pengaman dan Mode Operasi

1. **Interlock**: output dipaksa 0 bila stok B ≤ *safety stock*, stok A ≥ Max, atau ruang B penuh.
2. **Anti-windup**: integral tidak menumpuk saat output sudah mentok di 0 atau 100%.
3. **Mode AUTO/MANUAL**: operator dapat beralih lewat MQTT dan mengubah *setpoint* serta gain PID tanpa upload ulang kode.
4. **Fail-safe**: bila MQTT putus atau 60 detik tanpa perintah manual, sistem kembali ke AUTO.
5. **Prioritas**: (1) interlock keselamatan, (2) perintah manual, (3) PID mode AUTO.

## Rencana Pembagian Kerja

| No | Bidang Pekerjaan | Rincian Tugas | Penanggung Jawab |
|----|------------------|---------------|------------------|
| 1 | Rangkaian dan simulasi Wokwi | Merangkai ESP32, load cell + HX711, HC-SR04, servo, relay/LED, OLED, dan buzzer di Wokwi | Wendy |
| 2 | Program sensor | Pembacaan load cell dan HC-SR04, kalibrasi (tare, set_scale), filter, perhitungan jumlah unit dan fill % | Wendy |
| 3 | Program kendali PID | PID loop 1 dan 2, interlock, anti-windup, loop 3 supervisor, tuning Kp/Ki/Kd | Silo |
| 4 | MQTT dan operasi | Koneksi HiveMQ, mode AUTO/MANUAL, perintah `cmd/*`, fail-safe | Nabil |
| 5 | Pengujian | Menjalankan 6 skenario uji Wokwi dan mencatat hasil aktual | Farhan |
| 6 | Dokumentasi dan repository | README.md, upload kode ke GitHub, laporan, presentasi | Farhan |

## Rencana Pengujian Simulasi

| No | Skenario | Aksi di Simulasi | Hasil yang Diharapkan | Hasil Aktual |
|----|----------|------------------|-----------------------|--------------|
| 1 | Stok A di bawah setpoint | Turunkan berat load cell A | Output naik dan gerbang membuka; makin dekat 12 unit, bukaan mengecil | _Diisi setelah simulasi_ |
| 2 | Stok A mencapai Max | Naikkan berat load cell A | Interlock aktif: output 0, konveyor berhenti | _Diisi setelah simulasi_ |
| 3 | Ruang B hampir penuh | Dekatkan jarak HC-SR04 B | Pintu menyempit, menutup di 90%, buzzer berbunyi | _Diisi setelah simulasi_ |
| 4 | Cadangan B menipis | Turunkan berat load cell B | Alarm, REORDER dikirim, isi ulang A ditahan | _Diisi setelah simulasi_ |
| 5 | Mode manual dan tuning | Kirim `cmd/mode` manual dan `cmd/kpa` | Konveyor mengikuti perintah; gain berubah tanpa upload ulang | _Diisi setelah simulasi_ |
| 6 | Data tidak konsisten | Turunkan berat tanpa mengubah jarak | Status data tidak konsisten | _Diisi setelah simulasi_ |

## Repository

https://github.com/hanfarhansifarhan/Responsi-Tugas-Besar-Internet-of-Things
