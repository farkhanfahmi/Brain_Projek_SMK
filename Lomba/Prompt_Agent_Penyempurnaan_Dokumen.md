---
tags:
  - lomba
  - iga-2026
  - prompt-handoff
created: 2026-06-30
status: ready-to-use
---

# Prompt Bertahap untuk Claude.ai (Pro) — Penyempurnaan Dokumen IGA 2026

> Dirancang untuk dieksekusi di **claude.ai** secara bertahap (bukan satu prompt raksasa), karena dokumen + lampirannya cukup panjang untuk diproses maksimal dalam satu balasan. Ikuti urutan Tahap 0 → 4 di bawah, satu per satu, dalam **satu percakapan yang sama** (jangan buka chat baru di tengah jalan, supaya konteks tidak hilang).

---

## Cara Pakai (baca dulu sebelum mulai)

1. **Pakai fitur Projects di claude.ai** kalau tersedia di paket Pro Anda — upload 3 file ini ke Project Knowledge: `Dokumen_Lomba_IGA_2026_SIGAP-UJIAN.md`, `Lampiran_Foto_IGA_2026_SIGAP-UJIAN.md`, `Outline_IGA_2026_SIGAP-UJIAN.md`. Ini membuat ketiganya selalu jadi rujukan di setiap balasan tanpa perlu ditempel ulang.
2. **Kalau tidak pakai Projects** (chat biasa), tempel isi `Dokumen_Lomba_IGA_2026_SIGAP-UJIAN.md` langsung di Tahap 0 sebagai lampiran teks.
3. Jalankan Tahap 0 dulu dan **tunggu konfirmasi Claude** sebelum lanjut ke tahap berikutnya — supaya Anda tahu Claude benar-benar paham batasannya, bukan langsung lompat menulis.
4. Setiap tahap menghasilkan bagian dokumen yang sudah direvisi — salin hasilnya ke dokumen kerja Anda sebelum lanjut tahap berikutnya.

---

## TAHAP 0 — Brief Awal & Konfirmasi Pemahaman

```
Anda akan membantu menyempurnakan dokumen submission kompetisi Innovative Government Award (IGA) 2026 — kompetisi inovasi daerah resmi Kemendagri/BSKDN. Dokumen dinilai tim verifikator pemerintah, jadi tone harus formal-administratif khas dokumen pemerintahan Indonesia, bukan gaya marketing.

Saya akan kerjakan ini bertahap dalam beberapa giliran percakapan supaya Anda bisa fokus per bagian. Sebelum saya kirim dokumennya, ini batasan keras yang HARUS Anda ikuti di semua tahap:

1. JANGAN mengubah, membulatkan, atau "merapikan" angka apa pun (jumlah siswa, panitia, pengawas, kelas, sesi, berita acara, rupiah, lembar kertas). Semua angka itu hasil ekstraksi langsung dari database produksi dan spreadsheet sekolah, bukan estimasi yang bisa dipercantik.
2. JANGAN mengisi bagian yang ditandai [BELUM TERSEDIA] dengan kalimat yang menyiratkan item itu sudah ada (ini soal SK regulasi, video, bukti replikasi — ketiganya memang belum ada secara faktual).
3. JANGAN menghapus label "estimasi" pada angka yang memang estimasi (contoh: rentang waktu 46-126,5 jam, asumsi harga ATK per berita acara). Mengubahnya jadi angka tunggal tanpa label membuat klaim lebih lemah, bukan lebih kuat.
4. JANGAN menambahkan klaim baru yang tidak saya berikan sumbernya — termasuk dilarang mengarang data pembanding, testimoni, atau angka yang "kedengarannya pas".
5. JANGAN mengubah framing kategorisasi PKPN (Program Kerja Prioritas Nasional) yang sudah saya tetapkan tanpa bertanya dulu ke saya.
6. Kalau Anda ragu apakah suatu perubahan melanggar poin di atas — TANYA DULU, jangan asumsikan dan jalan terus.

Yang BOLEH Anda lakukan: perbaiki diksi, alur kalimat, transisi antar paragraf, tata bahasa, konsistensi istilah — selama tidak mengubah fakta/angka/struktur poin wajib.

Tolong konfirmasi Anda paham 6 batasan di atas, dan tanyakan ke saya kalau ada yang masih ambigu, SEBELUM saya kirim dokumen di giliran berikutnya.
```

**→ Tunggu balasan Claude.** Kalau Claude mengajukan pertanyaan, jawab dulu sebelum lanjut Tahap 1.

---

## TAHAP 1 — Revisi Bagian Identitas & Rancang Bangun

```
Terima kasih, lanjut ke bagian pertama. Ini Bagian A (Identitas Inovasi) dan Bagian B (Rancang Bangun, narasi 300 kata wajib dengan 6 sub-poin a-f) dari dokumen saya:

[TEMPEL ISI BAGIAN A & B DARI Dokumen_Lomba_IGA_2026_SIGAP-UJIAN.md DI SINI]

Tolong:
1. Pertahankan struktur 6 sub-poin (a-f) di Rancang Bangun — JANGAN dilebur jadi prosa bebas tanpa sub-poin, karena verifikator IGA mengecek per-poin.
2. Pertahankan narasi "fase semi-digital TA 2024/2025 → fase full-digital 2026" sebagai argumen inti yang menyelesaikan masalah eligibility waktu penerapan — pastikan ini terasa sebagai SATU cerita berkelanjutan, bukan dua hal terpisah.
3. Halu​skan diksi dan alur kalimat agar lebih persuasif untuk pembaca verifikator pemerintah, tanpa melanggar 6 batasan di Tahap 0.
4. Setelah merevisi, beri saya daftar terpisah: apa yang Anda ubah dan kenapa.
```

---

## TAHAP 2 — Revisi Bagian Tujuan, Manfaat, Hasil Inovasi

```
Lanjut ke Bagian C (Tujuan), D (Manfaat), dan E (Hasil Inovasi/Kemanfaatan):

[TEMPEL ISI BAGIAN C, D, E DI SINI]

Tolong:
1. Pastikan transisi dari Bagian B ke C-D terasa mengalir, bukan terasa seperti bagian terpisah yang ditempel.
2. Bagian E berbentuk tabel 6 sub-parameter — pertahankan formatnya, jangan diubah jadi narasi bebas karena ini memang format wajib formulir.
3. Tetap ikuti 6 batasan dari Tahap 0, terutama soal angka dan label estimasi.
4. Beri saya daftar perubahan + alasan seperti sebelumnya.
```

---

## TAHAP 3 — Revisi Bagian Regulasi, Video, Replikasi, Lampiran

```
Lanjut ke Bagian F (Regulasi), G (Video), H (Replikasi), I (Lampiran):

[TEMPEL ISI BAGIAN F, G, H, I DI SINI]

Tolong:
1. Bagian-bagian yang bertanda [BELUM TERSEDIA]/[STATUS: TIDAK ADA BUKTI] HARUS tetap eksplisit kosong/berstatus demikian — perbaiki hanya kalimat penjelasannya supaya lebih jelas dan profesional, BUKAN mengisinya dengan klaim bahwa itemnya sudah ada.
2. Tetap ikuti 6 batasan dari Tahap 0.
3. Beri saya daftar perubahan + alasan.
```

---

## TAHAP 4 — Pengecekan Konsistensi Akhir

```
Sekarang saya akan kirim seluruh dokumen hasil revisi Tahap 1-3 digabung jadi satu:

[TEMPEL SELURUH HASIL GABUNGAN DI SINI]

Tolong lakukan pengecekan akhir:
1. Apakah ada angka yang tidak konsisten antar bagian (misal jumlah siswa disebut beda di dua tempat)?
2. Apakah istilah "SIGAP UJIAN" dan nama-nama lain konsisten penulisannya di seluruh dokumen?
3. Apakah narasi "fase semi-digital → full-digital" tetap terasa sebagai satu cerita utuh dari awal sampai akhir dokumen?
4. Apakah ada bagian yang secara tidak sengaja mengisi placeholder [BELUM TERSEDIA] dengan klaim yang seharusnya tidak ada?

Laporkan semua temuan SEBELUM melakukan perbaikan apa pun — saya ingin review dulu sebelum disetujui.
```

---

## Catatan Tambahan

- Kalau di tengah proses Claude memberi hasil yang melanggar salah satu dari 6 batasan, **koreksi langsung di giliran itu** ("Tolong revisi lagi — Anda melanggar batasan #2 dengan mengisi bagian SK seolah sudah ada") — jangan lanjut ke tahap berikutnya sebelum itu diperbaiki.
- File `Lampiran_Foto_IGA_2026_SIGAP-UJIAN.md` tidak perlu direvisi tulisannya (isinya spesifikasi teknis, bukan narasi formulir) — cukup dijadikan referensi konteks kalau Claude butuh tahu rencana visual saat menulis ulang bagian terkait foto.
