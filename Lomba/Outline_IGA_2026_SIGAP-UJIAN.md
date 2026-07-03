---
tags:
  - lomba
  - iga-2026
  - outline
created: 2026-06-27
status: draft-outline
---

# Outline Dokumen IGA 2026 — SIGAP UJIAN

> Nama kerja: **SIGAP UJIAN** (Sistem Informasi Gerak-cepat Administrasi & Pemantauan Ujian)
> Aplikasi sumber: E-Berita-Acara-Ujian (Laravel 11 + 3 React SPA)
> Instansi pengusul: SMK Kartanegara Wates, Kabupaten Kediri
> Status: OUTLINE — belum draft penuh. Setiap bagian ditandai status data: ✅ ada/siap, ⚠️ ada sebagian/perlu validasi, ❌ belum ada/perlu dibuat.

---

## 0. Identitas & Kategorisasi Inovasi

| Field                                   | Isi                                                                         | Status                                                                                                                                                                |
| --------------------------------------- | --------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Nama Inovasi                            | SIGAP UJIAN                                                                 | ✅                                                                                                                                                                     |
| Perangkat Daerah Pengusul               | Dinas Pendidikan Kab. Kediri (via SMK Kartanegara Wates sbg pelaksana)      | ✅ (nota dinas sudah ada)                                                                                                                                              |
| Urusan Pemerintahan                     | Pendidikan (urusan wajib pelayanan dasar)                                   | ✅                                                                                                                                                                     |
| Tahapan Inovasi                         | **Penerapan** (bukan Inisiatif/Ujicoba)                                     | ✅ — didukung 2.014 siswa, 138 sesi, 1.659 BA real                                                                                                                     |
| Waktu Penerapan                         | Mulai TA 2025-2026 (data riil tercatat 30 Mar–4 Jun 2026)                   | ✅ masuk window 1 Jan 2024–31 Des 2025... ⚠️ CEK ULANG: window resmi 2026 perlu dikonfirmasi ulang apakah genap mencakup Jun 2026 data — lihat Catatan Risiko di bawah |
| Asta Cita                               | Asta Cita ke-4 (SDM, sains, teknologi, pendidikan)                          | ✅                                                                                                                                                                     |
| Program Kerja Prioritas Nasional (PKPN) | Klaster C (Pendidikan) → item "Digitalisasi Pendidikan" (framing diperluas) | ⚠️ framing perlu ditulis hati-hati, lihat catatan di bawah                                                                                                            |
| Inisiator                               | ASN/Guru (tim developer internal sekolah)                                   | ✅                                                                                                                                                                     |

**Catatan Risiko #1 — RESOLVED (29 Jun 2026)**: Pedoman resmi menyatakan syarat *"telah diterapkan sejak 1 Jan 2024 s.d. 31 Des 2025... DAN/ATAU telah dilakukan pembaharuan/pengembangan pada kurun waktu tersebut"* (klausul OR). Ditemukan bukti kuat dari 2 file spreadsheet legacy (`Salinan dari ABSENSI DIGITAL PENGAWAS UJIAN.xlsx` & `...PESERTA UJIAN.xlsx`) bahwa **embrio sistem ini (scan absensi digital berbasis Google Form + Spreadsheet) sudah berjalan sejak TA 2024/2025 Ganjil**, dengan jejak data terverifikasi:

- Tabel `DB. Ujian` (legacy): 13 event ujian tercatat berurutan dari **SAS Praktek Ganjil 2024/2025** s.d. **Sidang PKL 2025/2026**
- Jadwal Pengawas riil: **14 April 2025 – 10 Desember 2025** (943 baris)
- Input aplikasi pengawas riil: **4–10 Desember 2025** (509 baris)
- `DB. Walikelas`: **48 kelas TA 2024/2025**, **56 kelas TA 2025/2026** (cocok presisi dengan tabel `kelas` di app baru = 56 — konsistensi data kuat)
- `DB. Peserta`: **1.703 siswa unik TA 2024/2025**, **2.053 siswa unik TA 2025/2026** (selaras dengan 2.014 siswa unik di app baru utk sebagian ujian 2026)

**Narasi final yang dipakai**: Waktu Penerapan Awal = **TA 2024/2025 (sistem absensi digital berbasis spreadsheet)**, Waktu Pengembangan Terbaru = **2026 (migrasi penuh ke platform Laravel + React: SIGAP UJIAN, dengan penambahan modul BA digital, TV monitoring real-time, monitoring petugas keliling)**. Ini persis skenario yang diakomodasi klausul "pembaharuan/pengembangan" di pedoman — bukan klaim dipaksakan, didukung jejak data 2 sistem yang in masuk akal.

---

## 1. Rancang Bangun (narasi wajib, minimal 300 kata)

Struktur 6 bagian sesuai rekomendasi pedoman:

1. **Dasar Hukum Inovasi** — ❌ belum ada SK/Perkada. Sementara tulis dasar hukum generik (UU 23/2014 Psl 386, Permendagri terkait) + nyatakan SK Kepala Sekolah/Dinas SEDANG DIPROSES. **Action item paralel: buat SK sebelum submit.**
2. **Permasalahan** — bahan tersedia: proses BA manual (tulis tangan, rentan hilang/rusak, rekap lambat, rawan selisih presensi). Bisa pakai temuan nyata: **data 2025 yang hilang tanpa backup** sebagai ilustrasi risiko proses lama (sebelum digitalisasi penuh) — kuat secara naratif, tapi WASPADA jangan sampai jadi bukti kelemahan sistem sendiri. Perlu framing tepat: pisahkan jelas "masalah lama (manual)" vs "technical debt baru (backup policy)" agar tidak rancu.
3. **Isu Strategis** — keterkaitan ke transformasi digital pendidikan, akuntabilitas ujian, efisiensi anggaran sekolah.
4. **Metode Pembaharuan** — QR-based presensi, TV monitoring real-time, BA digital dengan tanda tangan digital, sistem role berlapis (Admin/Panitia/Pengawas), modul monitoring petugas keliling.
5. **Keunggulan & Kebaharuan** — dibanding proses manual: real-time monitoring 2 kampus simultan, audit trail otomatis, multi-role.
6. **Tahapan Inovasi/Spesifikasi Produk** — histori pengembangan dari Sprint 1 (setup) hingga modul monitoring petugas (Juni 2026) — tunjukkan ini *living product*, bukan statis.

Status: ✅ **DRAFT SELESAI (29 Jun 2026)** — ±300 kata, siap dipoles diksi sebelum final.

> **a. Dasar Hukum Inovasi**
> SIGAP UJIAN berlandaskan UU No. 23 Tahun 2014 Pasal 386 tentang mandat inovasi daerah, PP No. 38 Tahun 2017 Pasal 22 dan 24, Permendagri No. 104 Tahun 2018, serta PMDN No. 9 Tahun 2025 Pasal 724. Secara teknis-operasional, pelaksanaannya didukung Nota Dinas Pemerintah Daerah Kabupaten Kediri kepada SMK Kartanegara Wates dan SK Susunan Panitia Ujian setiap tahun ajaran. SK penetapan inovasi daerah secara formal saat ini sedang dalam proses penyusunan oleh pihak sekolah bersama Dinas Pendidikan.
>
> **b. Permasalahan**
> Sebelum digitalisasi, seluruh proses administrasi ujian — presensi pengawas, presensi peserta, hingga berita acara — dikerjakan manual secara tulis tangan. Proses ini rawan kehilangan dokumen fisik, rentan selisih data presensi, dan memerlukan waktu rekap yang panjang karena harus dikompilasi ulang dari puluhan ruang ujian secara terpisah.
>
> **c. Isu Strategis**
> Permasalahan ini berkaitan langsung dengan urgensi transformasi digital layanan pendidikan, akuntabilitas penyelenggaraan ujian sekolah, dan efisiensi anggaran operasional ATK. Inovasi ini selaras dengan Asta Cita ke-4 (penguatan SDM, sains, dan teknologi) serta Program Kerja Prioritas Nasional Klaster Pendidikan pada agenda Digitalisasi Pendidikan.
>
> **d. Metode Pembaharuan**
> Pembaharuan dilakukan secara bertahap dan terencana. Tahap pertama (Tahun Ajaran 2024/2025) menerapkan semi-digitalisasi: presensi pengawas dan peserta dialihkan ke sistem berbasis formulir digital dan spreadsheet, sementara berita acara tetap manual. Pendekatan ini disengaja sebagai masa transisi agar guru dan tenaga kependidikan terbiasa dengan budaya kerja digital sebelum perubahan menyeluruh diterapkan. Setelah adopsi tahap awal berjalan baik, tahap kedua (2026) menghadirkan digitalisasi penuh melalui platform terintegrasi berbasis Laravel dan React: presensi otomatis via pemindaian QR, berita acara digital bertanda tangan elektronik, serta pemantauan real-time lintas kampus melalui dashboard TV.
>
> **e. Keunggulan dan Kebaharuan**
> Dibandingkan proses manual maupun sistem spreadsheet sebelumnya, SIGAP UJIAN menyatukan seluruh siklus administrasi ujian dalam satu platform: presensi, berita acara, dan pengawasan real-time lintas dua kampus sekaligus, dilengkapi modul monitoring petugas keliling dan jejak audit otomatis pada setiap entitas.
>
> **f. Tahapan Inovasi/Penggunaan Produk**
> Sejak peluncuran modul inti hingga modul monitoring petugas keliling (Juni 2026), sistem telah digunakan secara nyata oleh 2.014 siswa, 167 panitia, dan 171 pengawas, mencakup 138 sesi ujian dan menghasilkan 1.659 berita acara digital pada periode Maret–Juni 2026, menjadikannya bukti penerapan berkelanjutan, bukan sekadar uji coba.

---

## 2. Tujuan

> SIGAP UJIAN bertujuan mempercepat dan menjamin akurasi rekapitulasi presensi serta berita acara ujian yang sebelumnya dikerjakan secara manual. Lebih lanjut, inovasi ini diarahkan untuk meningkatkan akuntabilitas pengawasan ujian secara real-time di dua lokasi kampus sekaligus, sehingga setiap insiden maupun ketidaktertiban dapat terpantau dan tercatat tanpa tertunda. Pada akhirnya, sistem ini bertujuan mengurangi beban administratif manual panitia dan pengawas, membebaskan waktu mereka untuk fokus pada substansi pengawasan ujian itu sendiri, bukan pada pekerjaan klerikal yang berulang.

Status: ✅ Naratif selesai.

---

## 3. Manfaat

> Penerapan SIGAP UJIAN memberikan manfaat langsung berupa efisiensi penggunaan kertas sebanyak 4.977 lembar dan penghematan biaya ATK sekitar Rp3.815.700 selama periode Maret–Juni 2026, dibandingkan proses manual yang membutuhkan 3 lembar kertas dan 1 bulpoin per berita acara. Dari sisi waktu, kompilasi laporan per sesi ujian yang sebelumnya membutuhkan estimasi 30–60 menit kini dapat diselesaikan dalam waktu 5–10 menit, menghemat estimasi 46 hingga 126,5 jam kerja panitia dalam satu semester — angka ini masih bersifat estimasi dan akan divalidasi lebih lanjut bersama panitia. Manfaat tidak langsung tercermin dari peningkatan ketertiban ujian: tercatat 59 catatan ketidaktertiban yang berhasil terdokumentasi sistem, menunjukkan bahwa SIGAP UJIAN menangkap dan mencatat insiden secara transparan, bukan menyembunyikannya. Dengan cakupan 100% dari 56 kelas yang ada di sekolah serta keterlibatan lebih dari 2.000 siswa, 167 panitia, dan 171 pengawas, manfaat inovasi ini telah dirasakan secara menyeluruh oleh seluruh elemen sekolah, bukan hanya sebagian kecil populasi.

Status: ✅ Naratif selesai, angka waktu tetap dilabeli estimasi sampai divalidasi panitia.

---

## 4. Hasil Inovasi / Indikator Kemanfaatan (6 sub-parameter, bobot 3)

| Sub-parameter                             | Data                                                                                                                                                                                                                                                                                                                                                                   | Status                                |
| ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| a. Cakupan penerima manfaat (orang)       | 2.014 siswa + 167 panitia + 171 pengawas = **2.352 orang langsung**                                                                                                                                                                                                                                                                                                    | ✅                                     |
| b. Cakupan unit (%)                       | 56 kelas terlayani app baru / 56 kelas total sekolah TA 2025/2026 (cross-check `DB. Walikelas` legacy) = **100% cakupan kelas**                                                                                                                                                                                                                                        | ✅ resolved via data legacy            |
| c. Efisiensi belanja (%)                  | Rupiah dihemat: 1.659 BA × Rp2.300 (3 lembar HVS @Rp100 + 1 bulpoin @Rp2.000) = **Rp3.815.700** dalam periode Mar-Jun 2026 (ESTIMASI, asumsi 1 bulpoin/BA cenderung overestimate). **Keputusan (29 Jun 2026): sekolah tidak punya alokasi ATK ujian terpisah dari ATK umum, sehingga persentase tidak dihitung — tulis rupiah absolut saja, jangan paksakan angka %.** | ✅ closed, ditulis sbg rupiah saja     |
| d. Penambahan pendapatan (%)              | Kemungkinan **tidak relevan** untuk aplikasi internal non-revenue — harus dinyatakan "N/A" jujur, jangan dipaksakan                                                                                                                                                                                                                                                    | ⚠️ keputusan: tulis N/A dengan alasan |
| e. Jumlah produk                          | 5 jenis ujian terlayani (UKK, PSAJ, STS, ASAT Praktik, ASAT Teori)                                                                                                                                                                                                                                                                                                     | ✅                                     |
| f. Tren dampak/capaian positif (NEW 2026) | Tren naik: dari manual → 138 sesi terdigitalisasi dalam 1 semester, modul monitoring baru (Juni 2026) menunjukkan pengembangan berkelanjutan                                                                                                                                                                                                                           | ✅                                     |

**Action item sebelum lanjut**: butuh 2 angka dari Anda — (1) total kelas riil di sekolah (untuk hitung % cakupan unit b), (2) estimasi harga satuan kertas+ATK per lembar BA (untuk hitung c).

---

## 5. Regulasi Inovasi Daerah (bobot 3)

Status: ❌ **belum ada SK sama sekali**. Ini bobot tertinggi kedua — **prioritas urgent**, harus selesai sebelum submission Jun-Agu 2026, idealnya SK Kepala Sekolah + surat dukungan Dinas (memperkuat nota dinas yang sudah ada).

**Keputusan (29 Jun 2026)**: Anda akan mengurus langsung ke Kepala Sekolah, target selesai sebelum jendela submission Jun-Agu 2026. **Tracking item — perlu di-cek progresnya secara berkala**, karena ini satu-satunya item yang murni di luar kendali saya dan berisiko jadi bottleneck terakhir.

**Checklist Tracking SK Penetapan Inovasi Daerah:**
- [ ] Susun draft permohonan/pengajuan ke Kepala Sekolah (minta isi: nama inovasi, dasar hukum, perangkat daerah pelaksana, rancang bangun ringkas, tujuan, manfaat, waktu uji coba/anggaran — lihat poin "Struktur Keputusan KDH" di `PANDUAN_STRATEGIS...md` bagian 4)
- [ ] Ajukan ke Kepala Sekolah untuk ditandatangani/diteruskan ke Dinas
- [ ] Konfirmasi apakah SK cukup dari Kepala Sekolah atau perlu naik ke tingkat Kepala Daerah/Dinas (cek ulang ke pedoman — kemungkinan perlu SK Kepala Perangkat Daerah, bukan cuma internal sekolah)
- [ ] SK terbit — target **sebelum akhir Juli 2026** (beri buffer 1 bulan sebelum penutupan submission Agustus 2026)
- [ ] Simpan scan/salinan SK ke vault sebagai lampiran final

⚠️ **Risiko jika molor**: tanpa SK, submission tetap bisa dikirim tapi skor indikator Regulasi (bobot 3) akan nol/minim — bukan gagal total, tapi mengurangi skor total secara nyata.

---

## 6. Video Inovasi Daerah (bobot 4 — tertinggi, wajib, upload ke Tuxedovation)

Status: ❌ **belum ada sama sekali**. 5 substansi wajib: latar belakang, penjaringan ide, pemilihan ide, manfaat inovasi, dampak inovasi.

**Storyboard (29 Jun 2026)** — target durasi ±5 menit:

| # | Substansi | Durasi | Visual | Narasi/VO (draft kasar) |
|---|---|---|---|---|
| 1 | Latar Belakang | 0:00–1:00 | Re-enactment: berita acara tulis tangan, kesan berantakan | "Setiap ujian dulu menghasilkan ratusan lembar BA tulis tangan — rawan hilang, rawan selisih data." |
| 2 | Penjaringan Ide | 1:00–1:45 | Wawancara tim developer/guru | "Ide ini lahir dari pengalaman langsung tim kami sebagai pengawas dan panitia." |
| 3 | Pemilihan Ide | 1:45–2:30 | Cuplikan fase transisi 2024/2025 (Google Form/Spreadsheet) | "Kami mulai bertahap — TA 2024/2025 presensi digital dulu, supaya guru terbiasa budaya digital." |
| 3.5 | Otomatisasi Kartu Peserta | 2:30–3:00 | Demo layar: petugas input data di Google Spreadsheet → klik jalankan Autocrat → tampilkan email masuk ke Gmail siswa berisi kartu (username/password CBT + barcode) | "Kartu peserta — lengkap dengan akses CBT dan barcode presensi — dibuat dan dikirim otomatis ke setiap siswa tanpa proses cetak manual." |
| 4 | Manfaat Inovasi | 3:00–4:00 | Demo scan barcode kartu, TV monitor real-time, BA digital + ttd elektronik | "Setelah terbiasa, kami hadirkan sistem penuh dalam satu platform." |
| 5 | Dampak Inovasi | 4:00–5:15 | Overlay data: 2.014 siswa, 167 panitia, 171 pengawas, 138 sesi, 1.659 BA, Rp3,8jt dihemat | "Bukti nyata penerapan, bukan sekadar wacana." |

Skrip kata-per-kata final TIDAK ditulis sengaja — perlu suara asli tim sekolah agar otentik. Ini kerangka adegan + poin kunci wajib.

---

## 7. Replikasi (bobot 3)

Status: ❌ **DIKONFIRMASI (29 Jun 2026): tidak ada bukti replikasi sama sekali** — bahkan dialog informal yang sebelumnya disebut, ternyata tidak memiliki jejak dokumentasi apapun (tidak ada screenshot, foto, atau catatan).

**Konsekuensi jujur**: indikator Replikasi (bobot 3) ini **akan diisi kosong/tidak optimal** di submission tahun ini — saya tidak akan menulis narasi yang menyiratkan ada replikasi atau bahkan "penjajakan" jika faktanya nihil. Opsi yang tersedia:
1. **Terima skor nol di indikator ini** untuk siklus 2026, fokus memaksimalkan indikator lain yang datanya kuat (Kemanfaatan, Tahapan Inovasi)
2. **Bertindak sekarang** (sebelum submission Jun-Agu 2026): hubungi 1 SMK lain di Kab. Kediri untuk sekadar demo/perkenalan singkat, lalu dokumentasikan (foto + catatan tanggal) — meski terlambat, ini lebih baik dari nol mutlak
Saya tidak akan merekomendasikan salah satu secara default — ini keputusan trade-off waktu vs skor yang sebaiknya Anda timbang sendiri.

---

## 8. Kelemahan yang Diakui Jujur (bagian integritas narasi, bukan bagian formal formulir tapi penting untuk konsistensi jika ditanya verifikator)

- Data 2025 hilang karena belum ada kebijakan backup (sedang diperbaiki)
- 3 item technical debt: rate limiting belum ada, SQL backup belum di-gitignore, ExamReportController God class 834 baris belum direfactor
- "Test teori" exam entry menunjukkan masih ada aktivitas dev/testing langsung di environment — idealnya dipisah dari production

---

## 9. Lampiran yang Perlu Disiapkan (non-tulisan, aksi paralel Anda)

- [ ] SK/Perkada regulasi (urgent, bobot 3)
- [ ] Video 5 substansi + upload Tuxedovation (urgent, bobot 4 — tertinggi)
- [ ] Foto dokumentasi kegiatan (presensi, monitoring TV, dll)
- [ ] Dokumen anggaran pendukung (jika ada alokasi APBS/APBD untuk pengembangan)
- [ ] Jejak dokumentasi replikasi (screenshot/foto/catatan dialog)
- [ ] Validasi angka waktu rekap manual ke 2-3 panitia/pengawas senior
- [ ] Angka total kelas sekolah riil (untuk indikator Kemanfaatan b)
- [ ] Estimasi harga satuan kertas+ATK per lembar BA (untuk indikator Kemanfaatan c)

---

## 10. Urutan Kerja Selanjutnya (usulan)

1. Anda validasi outline ini — beri catatan/koreksi arah
2. Saya tulis draft penuh per section (mulai dari Rancang Bangun, karena ini "jantung" penilaian)
3. Paralel: Anda kerjakan action item non-tulisan (SK, video, dokumentasi replikasi)
4. Iterasi draft sampai final, baru diserahkan ke agent desain untuk polish visual
