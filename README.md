# MO-SiPINE: Sistem Kriptografi Video & Audio Berbasis Modulo Sine-PWLCM

Sistem ini adalah aplikasi desktop kriptografi video dan audio berbasis peta chaos (chaotic map) dengan antarmuka GUI Tkinter. Aplikasi ini menggunakan operasi XOR antara data multimedia dan keystream pseudo-random yang dibangkitkan oleh peta chaos Modulo Sine-PWLCM.

---

## 1. Arsitektur Sistem & Struktur File

Aplikasi ini dibangun menggunakan arsitektur modular dengan pembagian berkas sebagai berikut:

### Berkas Utama Backend & GUI
* `sipine.py`: Implementasi inti algoritma pembangkit keystream chaos yang dikompilasi menggunakan Numba JIT untuk mencapai performa tinggi. Berkas ini tidak boleh dimodifikasi.
* `ai_optimizer.py`: Modul pencarian parameter kunci chaos optimal berbasis Heuristic Search dan Narrowing Refinement untuk memaksimalkan nilai Shannon Entropy.
* `video_core.py`: Backend utama untuk pemrosesan enkripsi dan dekripsi video serta audio terpadu ke dalam format berkas biner kustom `VEN2`.
* `audio_core.py`: Backend khusus untuk pemrosesan enkripsi dan dekripsi berkas audio WAV mandiri (standalone) menggunakan format `AUENC`.
* `gui_video_main.py`: Titik masuk (entry point) utama aplikasi untuk membuka jendela utama GUI berukuran 480x700 px.
* `gui_video.py`: Modul antarmuka komprehensif yang mengelola alur enkripsi, dekripsi, penampil video (Video Player), serta pengujian kualitas statistik.

### Berkas Analisis & Utilitas
* `generator_nist.py`: Skrip mandiri untuk membangkitkan bitstream 1 Megabit (125.000 byte) guna keperluan pengujian acak menggunakan NIST Statistical Test Suite (SP 800-22).
* `bifurkasi.py`: Skrip visualisasi untuk membuat dan menampilkan diagram bifurkasi peta chaos Modulo Sine-PWLCM terhadap parameter kontrol.
* `lyapunov.py`: Skrip visualisasi untuk menghitung eksponen Lyapunov guna membuktikan sensitivitas dan kondisi kaotik sistem.
* `fix_avi.py`: Skrip utilitas pembantu untuk migrasi atau konversi format dari berkas `.avi` ke `.mp4`.

### Dependensi Utama
Sistem ini membutuhkan beberapa pustaka Python berikut untuk dapat berjalan:
* `numpy`, `numba`, `scipy`, `scikit-image` (Komputasi dan Pengolahan Data).
* `opencv-python (cv2)`, `Pillow (PIL)`, `matplotlib` (Multimedia dan Visualisasi).
* Perangkat lunak eksternal **FFmpeg** yang terinstal di sistem (Diperlukan untuk pemrosesan stream audio).

---

## 2. Inti Persamaan Chaotic Map

Sistem menyediakan 3 mode peta chaos yang dapat dipilih melalui antarmuka pengguna:

### Mode 0: MO-SiPINE (Modulo Sine-PWLCM — Default)
Peta chaos ini merupakan komposisi nonlinear antara Piecewise Linear Chaotic Map (PWLCM) dan fungsi Sine, dengan rumus dasar:

    x_(n+1) = (beta * sin(pi * F_PWLCM(x_n)) * 987676766.3232) mod 1.0

Dimana fungsi F_PWLCM(x) didefinisikan sebagai:
* Jika x >= 0.5 maka mengevaluasi (1 - x)
* Jika 0 <= x < p maka f(x) = x / p
* Jika p <= x < 0.5 maka f(x) = (x - p) / (0.5 - p)

### Mode 1: Sine Only
Peta chaos yang hanya memanfaatkan fungsi Sine:

    x_(n+1) = (beta * sin(pi * x_n) * 987676766.3232) mod 1.0

### Mode 2: PWLCM Only
Peta chaos yang hanya memanfaatkan fungsi PWLCM:

    x_(n+1) = (F_PWLCM(x_n) * 987676766.3232) mod 1.0

### Tahapan Pembangkitan Keystream
1. **Fase Pemanasan**: Iterasi awal dilakukan tanpa mengambil output untuk membuang fase transient awal agar sistem stabil di rezim kaotik.
2. **Sub-Sampling**: Pembuangan iterasi sebanyak N_SKIP kali per byte untuk menaikkan keacakan (Versi standar N_SKIP = 12, versi video N_SKIP = 4).
3. **Injeksi Umpan Balik**: Kondisi internal diubah secara dinamis berdasarkan byte output sebelumnya (k_prev):
   
       x1 = (x1 + k_prev / 256) mod 1.0
       x2 = (x2 + x1) mod 1.0
   
4. **Cascade XOR Folding**: Nilai pecahan float64 dikonversi ke integer 32-bit, lalu dilakukan operasi lipatan XOR antarkomponen byte untuk menghasilkan nilai byte output (uint8).

---

## 3. Optimalisasi Kunci Berbasis AI

Modul **AI Optimizer v2** bekerja secara otomatis mencari kombinasi kunci parameter terbaik (x1, x2, p, beta) dengan alur sebagai berikut:
1. Membangkitkan kandidat kunci secara acak.
2. Melakukan penyaringan awal menggunakan filter estimasi Eksponen Lyapunov (n = 300 iterasi). Hanya kandidat berstatus kaotik (Lyapunov > 0) yang akan diuji lebih lanjut.
3. Menguji keystream sampel sebesar 50.000 byte untuk menghitung nilai Shannon Entropy.
4. Jika kualitas enkripsi meningkat, sistem beralih ke Fase Lokal (Narrowing Refinement) menggunakan perturbasi Gaussian (std = 0.05) untuk mempercepat konvergensi.
5. Sistem akan keluar lebih cepat (early exit) jika tingkat entropi telah mencapai >= 7.999.

---

## 4. Format Berkas Terenkripsi (Custom Binary)

### A. Format VEN2 (Video + Audio Terpadu — 36 Byte Header)
Format berkas `.bin` hasil enkripsi menyertakan komponen Initialization Vector (IV) untuk menjamin aspek Semantic Security dan mencegah serangan Two-Time Pad:

| Offset | Ukuran | Tipe Data | Deskripsi Metadata |
| --- | --- | --- | --- |
| 0 | 4 byte | bytes | Magic ASCII `"VEN2"` |
| 4 | 4 byte | uint32 | Lebar Frame (Width) |
| 8 | 4 byte | uint32 | Tinggi Frame (Height) |
| 12 | 4 byte | uint32 | Total Frames Video |
| 16 | 4 byte | float32 | Frame Per Second (FPS) |
| 20 | 4 byte | uint32 | Audio Sample Rate |
| 24 | 2 byte | uint16 | Jumlah Channel Audio |
| 26 | 4 byte | uint32 | Total Sampel Audio (Num Audio Samples) |
| 30 | 2 byte | uint16 | Status Komponen Audio (0 = Tidak, 1 = Ya) |
| 32 | 4 byte | uint32 | Initialization Vector (IV) |
| 36+ | - | bytes | Data utama video disusul data audio terenkripsi |

### B. Format AUENC (Audio Standalone — 24 Byte Header)
Digunakan khusus untuk enkripsi berkas suara berekstensi WAV secara mandiri:

| Offset | Ukuran | Tipe Data | Deskripsi Metadata |
| --- | --- | --- | --- |
| 0 | 5 byte | bytes | Magic ASCII `"AUENC"` |
| 5 | 4 byte | uint32 | Sample Rate |
| 9 | 2 byte | uint16 | Jumlah Channel |
| 11 | 4 byte | uint32 | Jumlah Sampel (Num Samples) |
| 15 | 2 byte | uint16 | Kedalaman Bit (Bit Depth) |
| 17 | 2 byte | uint16 | Status Berkas Stereo (0 = Mono, 1 = Stereo) |
| 19 | 5 byte | bytes | Data Padding kosong |
| 24+ | - | bytes | Data mentah audio terenkripsi |

---

## 5. Panduan Penting untuk Analisis Akademis

### Aturan Interpolasi Ukuran Frame
Saat mengekstrak key-frame berkas cipher untuk keperluan uji statistik (seperti uji histogram, korelasi spasial, NPCR, UACI, dan entropi), **WAJIB** menggunakan metode pengubahan ukuran **`PIL.Image.NEAREST`**. 

Penggunaan metode interpolasi linear seperti **`LANCZOS`** akan menyebabkan nilai piksel tetangga dirata-ratakan secara artifisial. Hal ini merusak keacakan cipher dan menyebabkan kesalahan fatal pada data riset, seperti:
* Nilai Shannon Entropy turun secara tidak valid dari nilai ideal ~8.0 menjadi ~6.3.
* Sebaran grafik histogram berubah menjadi melengkung (bell-curve), padahal seharusnya flat/seragam.

### Jaminan Rekonstruksi Bit Sempurna (Lossless)
Proses dekripsi memanfaatkan FFmpeg dengan mekanisme pipe stream langsung menuju encoder khusus:
* **`libx264rgb`** (Prioritas utama): Melakukan kompresi langsung pada ruang warna BGR tanpa konversi warna, menghilangkan rounding error secara total.
* **`libx264` + `yuv444p`** (Fallback otomatis): Menggunakan format piksel 4:4:4 untuk menghindari kesalahan akibat pengurangan resolusi warna (chroma subsampling).

Melalui implementasi ini, nilai metrik Mean Squared Error (MSE) antara dokumen video asli dengan berkas hasil dekripsi bernilai tepat 0.0 dan nilai Peak Signal-to-Noise Ratio (PSNR) mencapai Infinity. Hal ini secara matematis membuktikan bahwa sistem kriptografi MO-SiPINE bersifat reversible sempurna (bit-perfect).
