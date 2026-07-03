---
tags:
  - lomba
  - iga-2026
  - dokumen-final
created: 2026-06-30
status: ready-for-design-pass
source_outline: "[[Outline_IGA_2026_SIGAP-UJIAN]]"
---

# Dokumen Lomba IGA 2026 — SIGAP UJIAN

> **Catatan**: Dokumen ini adalah kompilasi rapi dari hasil diskusi & riset data di [[Outline_IGA_2026_SIGAP-UJIAN]]. Bagian yang masih kosong/belum tersedia ditandai eksplisit `[BELUM TERSEDIA]` — JANGAN dihapus atau diisi dengan klaim kosong sebelum datanya benar-benar ada.

---

## A. Identitas Inovasi

| Field | Isi |
|---|---|
| **Nama Inovasi** | SIGAP UJIAN (Sistem Informasi Gerak-cepat Administrasi & Pemantauan Ujian) |
| **Instansi Pengusul** | SMK Kartanegara Wates, Kabupaten Kediri |
| **Perangkat Daerah Pelaksana** | Dinas Pendidikan Kabupaten Kediri (via SMK Kartanegara Wates sebagai unit pelaksana) |
| **Urusan Pemerintahan** | Pendidikan (Urusan Wajib Pelayanan Dasar) |
| **Tahapan Inovasi** | Penerapan |
| **Waktu Uji Coba** | Tahun Ajaran 2024/2025 Ganjil (sistem absensi digital berbasis spreadsheet) |
| **Waktu Penerapan Awal** | Tahun Ajaran 2024/2025 |
| **Waktu Pengembangan Terbaru** | 2026 (migrasi penuh ke platform Laravel + React, modul BA digital, TV monitoring real-time, monitoring petugas keliling) |
| **Inisiator** | ASN/Guru (tim developer internal sekolah) |
| **Asta Cita** | Asta Cita ke-4 — Memperkuat Pembangunan SDM, Sains, dan Teknologi |
| **Program Kerja Prioritas Nasional (PKPN)** | Klaster C (Pendidikan) → Digitalisasi Pendidikan |

---

## B. Rancang Bangun Inovasi Daerah dan Pokok Perubahan (min. 300 kata)

**a. Dasar Hukum Inovasi**

SIGAP UJIAN berlandaskan UU No. 23 Tahun 2014 Pasal 386 tentang mandat inovasi daerah, PP No. 38 Tahun 2017 Pasal 22 dan 24, Permendagri No. 104 Tahun 2018, serta PMDN No. 9 Tahun 2025 Pasal 724. Secara teknis-operasional, pelaksanaannya didukung Nota Dinas Pemerintah Daerah Kabupaten Kediri kepada SMK Kartanegara Wates dan SK Susunan Panitia Ujian setiap tahun ajaran. SK penetapan inovasi daerah secara formal saat ini sedang dalam proses penyusunan oleh pihak sekolah bersama Dinas Pendidikan. `[SK FORMAL: BELUM TERSEDIA — lihat checklist tracking di outline]`

**b. Permasalahan**

Sebelum digitalisasi, seluruh proses administrasi ujian — presensi pengawas, presensi peserta, hingga berita acara — dikerjakan manual secara tulis tangan. Proses ini rawan kehilangan dokumen fisik, rentan selisih data presensi, dan memerlukan waktu rekap yang panjang karena harus dikompilasi ulang dari puluhan ruang ujian secara terpisah.

**c. Isu Strategis**

Permasalahan ini berkaitan langsung dengan urgensi transformasi digital layanan pendidikan, akuntabilitas penyelenggaraan ujian sekolah, dan efisiensi anggaran operasional ATK. Inovasi ini selaras dengan Asta Cita ke-4 (penguatan SDM, sains, dan teknologi) serta Program Kerja Prioritas Nasional Klaster Pendidikan pada agenda Digitalisasi Pendidikan.

**d. Metode Pembaharuan**

Pembaharuan dilakukan secara bertahap dan terencana. Tahap pertama (Tahun Ajaran 2024/2025) menerapkan semi-digitalisasi: presensi pengawas dan peserta dialihkan ke sistem berbasis formulir digital dan spreadsheet, sementara berita acara tetap manual. Pendekatan ini disengaja sebagai masa transisi agar guru dan tenaga kependidikan terbiasa dengan budaya kerja digital sebelum perubahan menyeluruh diterapkan. Setelah adopsi tahap awal berjalan baik, tahap kedua (2026) menghadirkan digitalisasi penuh melalui platform terintegrasi berbasis Laravel dan React: presensi otomatis via pemindaian QR, berita acara digital bertanda tangan elektronik, serta pemantauan real-time lintas kampus melalui dashboard TV.

**e. Keunggulan dan Kebaharuan**

Dibandingkan proses manual maupun sistem spreadsheet sebelumnya, SIGAP UJIAN menyatukan seluruh siklus administrasi ujian dalam satu platform: presensi, berita acara, dan pengawasan real-time lintas dua kampus sekaligus, dilengkapi modul monitoring petugas keliling dan jejak audit otomatis pada setiap entitas.

**f. Tahapan Inovasi/Penggunaan Produk**

*Tahapan Inovasi*: Sejak peluncuran modul inti hingga modul monitoring petugas keliling (Juni 2026), sistem telah digunakan secara nyata oleh 2.014 siswa, 167 panitia, dan 171 pengawas, mencakup 138 sesi ujian dan menghasilkan 1.659 berita acara digital pada periode Maret–Juni 2026, menjadikannya bukti penerapan berkelanjutan, bukan sekadar uji coba.

*Penggunaan Produk* — alur kerja sistem dibagi tiga tahap:

1. **Pra-ujian**: Petugas mengolah data jadwal ujian pada Google Spreadsheet, yang kemudian memicu pembuatan kartu peserta secara otomatis melalui add-on Autocrat — kartu langsung terkirim ke alamat Gmail masing-masing siswa tanpa proses cetak atau distribusi manual satu per satu. Setiap kartu memuat kredensial (username dan password) untuk masuk ke aplikasi Computer-Based Test (CBT) serta barcode untuk presensi kehadiran, menyatukan dua kebutuhan administratif dalam satu dokumen digital.
2. **Hari pelaksanaan**: Presensi peserta dan pengawas tercatat otomatis melalui pemindaian barcode pada kartu peserta. Selama ujian berlangsung, dashboard TV menampilkan status kehadiran secara real-time di dua lokasi kampus, sementara petugas keliling memverifikasi kondisi lapangan melalui aplikasi pemantauan.
3. **Pasca-ujian**: Pengawas menyusun berita acara digital — jumlah peserta hadir dan tidak hadir terhitung otomatis dari data presensi, dilengkapi catatan ketidaktertiban (bila ada), dan disahkan melalui tanda tangan elektronik. Panitia selanjutnya merekapitulasi dan mengekspor laporan agregat dari seluruh sesi ujian.

*(±300 kata)*

---

## C. Tujuan Inovasi Daerah

> SIGAP UJIAN bertujuan mempercepat dan menjamin akurasi rekapitulasi presensi serta berita acara ujian yang sebelumnya dikerjakan secara manual. Lebih lanjut, inovasi ini diarahkan untuk meningkatkan akuntabilitas pengawasan ujian secara real-time di dua lokasi kampus sekaligus, sehingga setiap insiden maupun ketidaktertiban dapat terpantau dan tercatat tanpa tertunda. Pada akhirnya, sistem ini bertujuan mengurangi beban administratif manual panitia dan pengawas, membebaskan waktu mereka untuk fokus pada substansi pengawasan ujian itu sendiri, bukan pada pekerjaan klerikal yang berulang.

---

## D. Manfaat yang Diperoleh

> Penerapan SIGAP UJIAN memberikan manfaat langsung berupa efisiensi penggunaan kertas sebanyak 4.977 lembar dan penghematan biaya ATK sekitar Rp3.815.700 selama periode Maret–Juni 2026, dibandingkan proses manual yang membutuhkan 3 lembar kertas dan 1 bulpoin per berita acara. Dari sisi waktu, kompilasi laporan per sesi ujian yang sebelumnya membutuhkan estimasi 30–60 menit kini dapat diselesaikan dalam waktu 5–10 menit, menghemat estimasi 46 hingga 126,5 jam kerja panitia dalam satu semester — angka ini masih bersifat estimasi dan akan divalidasi lebih lanjut bersama panitia. Manfaat tidak langsung tercermin dari peningkatan ketertiban ujian: tercatat 59 catatan ketidaktertiban yang berhasil terdokumentasi sistem, menunjukkan bahwa SIGAP UJIAN menangkap dan mencatat insiden secara transparan, bukan menyembunyikannya. Dengan cakupan 100% dari 56 kelas yang ada di sekolah serta keterlibatan lebih dari 2.000 siswa, 167 panitia, dan 171 pengawas, manfaat inovasi ini telah dirasakan secara menyeluruh oleh seluruh elemen sekolah, bukan hanya sebagian kecil populasi.

---

## E. Hasil Inovasi / Indikator Kemanfaatan

| Sub-parameter | Data |
|---|---|
| a. Cakupan penerima manfaat | 2.014 siswa + 167 panitia + 171 pengawas = **2.352 orang langsung** |
| b. Cakupan unit | 56 dari 56 kelas sekolah TA 2025/2026 = **100%** |
| c. Efisiensi belanja | **Rp3.815.700** dihemat (estimasi, periode Mar–Jun 2026) — persentase tidak dihitung karena sekolah tidak memiliki alokasi ATK ujian terpisah |
| d. Penambahan pendapatan | **N/A** — aplikasi internal non-revenue, tidak relevan untuk diukur |
| e. Jumlah produk | 5 jenis ujian terlayani: UKK, PSAJ, STS Genap, ASAT Praktik, ASAT Teori |
| f. Tren dampak/capaian positif | Tren naik — dari sistem manual (2024/2025) ke semi-digital, ke full digital (2026) dengan penambahan modul monitoring petugas (Juni 2026) menunjukkan pengembangan berkelanjutan |

---

## F. Regulasi Inovasi Daerah

`[STATUS: BELUM TERSEDIA — SK Penetapan Inovasi Daerah masih dalam proses]`

Dasar yang sudah ada saat ini: Nota Dinas Pemerintah Daerah Kabupaten Kediri + SK Susunan Panitia Ujian (operasional, bukan pengakuan inovasi). Checklist tracking proses penerbitan SK tersedia di [[Outline_IGA_2026_SIGAP-UJIAN#5-regulasi-inovasi-daerah-bobot-3]].

---

## G. Video Inovasi Daerah (wajib, upload Tuxedovation)

`[STATUS: BELUM DIPRODUKSI]`

Storyboard 5 substansi (Latar Belakang, Penjaringan Ide, Pemilihan Ide, Manfaat, Dampak) — durasi ±5 menit — sudah tersedia di [[Outline_IGA_2026_SIGAP-UJIAN#6-video-inovasi-daerah-bobot-4--tertinggi-wajib-upload-ke-tuxedovation]]. Produksi & skrip kata-per-kata final menjadi tanggung jawab tim sekolah agar otentik.

---

## H. Replikasi

`[STATUS: TIDAK ADA BUKTI — dikonfirmasi 29 Jun 2026]`

Belum ada dialog, dokumentasi, maupun jejak apapun terkait minat replikasi dari sekolah lain. Indikator ini (bobot 3) diperkirakan tidak memperoleh skor optimal pada siklus pelaporan 2026, kecuali ada tindakan konkret (demo/perkenalan ke sekolah lain + dokumentasi) sebelum jendela submission Juni–Agustus 2026.

---

## I. Lampiran yang Masih Diperlukan

- [ ] SK Penetapan Inovasi Daerah
- [ ] Video 5 substansi (upload Tuxedovation)
- [ ] Foto dokumentasi kegiatan — **spesifikasi detail per lokasi, komposisi, dan caption ada di [[Lampiran_Foto_IGA_2026_SIGAP-UJIAN]]**, jangan ambil foto tanpa rujuk dokumen itu dulu
- [ ] Dokumentasi replikasi (jika opsi tindak lanjut diambil)
- [ ] Validasi angka waktu rekap manual ke panitia/pengawas senior

---

## Riwayat Data Sumber

Seluruh angka di dokumen ini berasal dari ekstraksi langsung:
- Database produksi `berita_acara_ujian_baru (1).sql` (periode Mar–Jun 2026)
- Spreadsheet legacy `ABSENSI DIGITAL PENGAWAS UJIAN.xlsx` & `ABSENSI DIGITAL PESERTA UJIAN.xlsx` (periode TA 2024/2025 – 2025/2026)

Detail metodologi ekstraksi dan seluruh pertimbangan/trade-off ada di [[Outline_IGA_2026_SIGAP-UJIAN]] — dokumen ini adalah versi ringkas siap pakai, outline tetap jadi rujukan kerja.
