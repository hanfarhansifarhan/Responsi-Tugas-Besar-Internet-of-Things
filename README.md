# Smart Stock Control

> Kendali PID persediaan gudang dua ruangan berbasis IoT (ESP32 + MQTT), dengan stok ruang A dijaga otomatis di setpoint lewat isi ulang dari ruang B.

## Identitas Kelompok

| | |
|---|---|
| **Nama Kelompok** | Kelompok 1 |
| **Mata Kuliah** | Supply Chain Management (SCM) |
| **Dosen Pengampu** | [Nama Dosen] |
| **Program Studi** | Informatika, Universitas Jenderal Soedirman (UNSOED) |

## Anggota Kelompok

| No | Nama | NIM | GitHub |
|----|------|-----|--------|
| 1 | Muhammad Nabil Zaedan Agesy | H1H024062 | [@BillyUdin](https://github.com/BillyUdin) |
| 2 | [Nama Anggota 2] | [NIM] | [@username](https://github.com/username) |
| 3 | [Nama Anggota 3] | [NIM] | [@username](https://github.com/username) |
| 4 | Wendy Virtus | H1H024048 | [@wewen06](https://github.com/wewen06) |

## Gambaran Umum Sistem

Stok manual selalu terlambat satu langkah: barang habis saat dibutuhkan (stockout), stok menumpuk dan memenuhi ruang (overstock), dan stok opname manual memakan waktu serta rawan salah hitung. Smart Stock Control memantau stok dua ruangan secara real-time dan mengotomatiskan keputusan isi ulang tanpa menunggu petugas.

Ruang B menyimpan cadangan dan ruang A melayani permintaan. Saat barang diambil dari ruang A dan stoknya turun dari setpoint, gerbang dan konveyor dibuka sebanding dengan kekurangannya. Makin dekat ke setpoint bukaan makin kecil, lalu berhenti saat stok sudah di target. Supplier mengirim barang ke ruang B lewat pintu servo.

### Tujuan

1. Memantau stok dua ruangan secara real-time dengan load cell dan ultrasonik.
2. Mengotomatiskan isi ulang ruang A dari ruang B dengan kendali PID loop tertutup.
3. Melindungi ruang dari kepenuhan dan stok cadangan dari kehabisan.
4. Memungkinkan pengawasan dan kendali manual jarak jauh lewat MQTT.

## Komponen yang Digunakan

| Komponen | Fungsi | Di simulasi Wokwi |
|----------|--------|-------------------|
| ESP32 | Menjalankan PID, WiFi, dan MQTT | ESP32 DevKit C V4 |
| Load cell + HX711 (2 set) | Berat tumpukan, dikonversi jadi jumlah unit | HX711 + load cell 5 kg |
| HC-SR04 (2 buah) | Tinggi tumpukan dan fill % | HC-SR04 |
| Servo (2 buah) | Gerbang A-B dan pintu supplier (sudut = output PID) | Servo |
| Motor DC TT + driver L298N/TB6612 | Konveyor, kecepatan lewat PWM | Relay + LED + sabuk NeoPixel |
| OLED, LED, buzzer | Indikator di lokasi | Sama |
| MQTT (HiveMQ) | Pemantauan dan kendali jarak jauh | Sama |

Motor DC dan driver dipakai di alat nyata. Di simulasi, konveyor diwakili relay dan LED.

## Diagram Alur Sistem

```
Supplier ──(pintu servo)──> [ Ruang B ] ──(gerbang + konveyor)──> [ Ruang A ] ──> Permintaan
                           stok cadangan                          stok depan      (gangguan)
                                 │                                     │
                                 └──────────── ESP32 (PID 200 ms) <────┘
                                      load cell + HX711, HC-SR04
                                                 │
                                           MQTT (HiveMQ)
```

Diagram blok dan flowchart lengkap ada di file progress (PDF).

## Mekanisme Kerja

### Pengukuran

- **Load cell + HX711** menghitung jumlah unit dengan rumus `jumlah = (berat - tare) / berat_unit`, dikalibrasi dengan `tare()` dan `set_scale()`, difilter rata-rata 5 sampel.
- **HC-SR04** menghitung kepenuhan dengan rumus `fill% = (H - d) / H x 100`, dipasang di atas rak menghadap ke bawah, difilter eksponensial 50%.
- **Verifikasi silang:** bila berat turun tetapi tinggi tumpukan tidak berubah, data ditandai tidak konsisten.

### Tiga Loop Kendali

| Loop | Tugas | Jenis | Detail |
|------|-------|-------|--------|
| 1 | Menjaga stok ruang A | PID | Setpoint 12 unit (ROP 6, Max 18). Output 0-100% menjadi sudut gerbang 0-90° dan kecepatan sabuk. Nilai awal Kp 8, Ki 0,8, Kd 0 |
| 2 | Melindungi ruang B dari kepenuhan | PID | Setpoint 75% fill, output bukaan pintu supplier, pintu ditutup paksa di fill >= 90% |
| 3 | Supervisor stok cadangan | On-off (berbasis kejadian) | Stok B <= safety stock (4 unit): alarm, kirim REORDER, tahan isi ulang A |

Rumus PID: `output = Kp x e + Ki x integral(e dt) + Kd x dPV/dt`, dengan `e = setpoint - stok terbaca`.

### Keselamatan

- **Interlock** di atas PID: stok B <= safety stock, stok A >= Max, atau B penuh memaksa output ke 0 di semua mode.
- **Anti-windup:** integral tidak menumpuk saat output mentok di 0 atau 100%.
- **Derivatif dari pengukuran:** tidak ada lonjakan output saat setpoint diubah.
- **Reset saat manual:** integral dikosongkan di mode manual.
- **Fail-safe:** MQTT putus atau 60 detik tanpa perintah manual, sistem kembali ke AUTO.
- **Prioritas:** interlock keselamatan > perintah manual > PID (mode AUTO).

### Siklus Program (tiap 200 ms)

Baca sensor, hitung stok, jalankan PID (loop 1 dan 2), evaluasi interlock, gerakkan aktuator, lalu tampilkan (OLED, LED, buzzer) dan kirim data via MQTT. Sensor dan jaringan memakai `millis()` tanpa blocking.

### Perintah MQTT

| Topik | Nilai |
|-------|-------|
| `cmd/mode` | `auto` / `manual` |
| `cmd/konveyor` | `on` / `off` |
| `cmd/pintu` | `buka` / `tutup` |
| `cmd/rop` | angka |
| `cmd/max` | angka |
| `cmd/sp` | angka (unit) |
| `cmd/spb` | angka (%) |
| `cmd/kpa`, `cmd/kia`, `cmd/kda` | gain loop 1 |
| `cmd/kpb`, `cmd/kib`, `cmd/kdb` | gain loop 2 |

Setpoint dan gain PID bisa diubah saat sistem berjalan tanpa upload ulang kode.

## Keterkaitan dengan Konsep SCM

| Konsep SCM | Diwujudkan di sistem |
|------------|----------------------|
| Reorder point dan safety stock | Menjadi setpoint dan interlock kendali |
| Sistem tarik (pull), Kanban | Ruang A menarik stok dari ruang B saat kurang |
| Multi-echelon inventory | Dua ruangan: stok depan dan gudang cadangan |
| Pencegahan stockout | Loop 3 menjaga cadangan dan memicu REORDER |
| Akurasi dan visibilitas stok | Load cell menggantikan opname manual, data via MQTT |
| Utilisasi gudang | Ultrasonik mengukur fill % tiap ruangan |
| Efek bullwhip | Data permintaan real-time mengurangi pesanan berlebih |

## Rencana Pengujian (Simulasi Wokwi)

| Skenario | Aksi di simulasi | Hasil yang diharapkan |
|----------|------------------|-----------------------|
| Stok A di bawah setpoint | Turunkan berat load cell A | Output naik, gerbang membuka, makin dekat 12 unit bukaan mengecil |
| Stok A mencapai Max | Naikkan berat load cell A | Interlock: output 0, konveyor berhenti |
| Ruang B hampir penuh | Dekatkan jarak HC-SR04 B | Pintu menyempit, tutup di 90%, buzzer bunyi |
| Cadangan B menipis | Turunkan berat load cell B | Alarm, REORDER, isi ulang A ditahan |
| Mode manual dan tuning | Kirim `cmd/mode manual`, `cmd/kpa` | Konveyor menurut perintah, gain berubah tanpa upload |
| Data tidak konsisten | Turunkan berat tanpa ubah jarak | Status data tidak konsisten |

Kolom hasil adalah yang diharapkan. Hasil aktual dicatat setelah simulasi dijalankan.

## Indikator Kinerja

Frekuensi stockout, akurasi stok, utilisasi ruang, simpangan dari setpoint, waktu respons, dan cakupan stok (days of supply). Nilai target ditentukan dari hasil pengujian.

## Pembagian Kerja

| Anggota | Tugas |
|---------|-------|
| Muhammad Nabil Zaedan Agesy | [Tugas] |
| [Nama Anggota 2] | [Tugas] |
| [Nama Anggota 3] | [Tugas] |
| [Nama Anggota 4] | [Tugas] |

## Rencana Pengembangan

1. **Tuning otomatis:** plant virtual dan autotune agar Kp, Ki, Kd tidak ditebak.
2. **Konveyor nyata:** motor DC dengan driver PWM, output PID menjadi duty cycle.
3. **Multi-barang:** berat per unit berbeda tiap jenis, ditambah RFID atau barcode.
4. **Pesanan dan dashboard:** REORDER membuat purchase order; Node-RED atau Grafana untuk riwayat dan KPI.

## Struktur Repository

```
.
├── README.md
├── docs/        # File progress (.pdf), diagram blok / flowchart
├── src/         # Kode program ESP32
└── assets/      # Gambar pendukung
```
