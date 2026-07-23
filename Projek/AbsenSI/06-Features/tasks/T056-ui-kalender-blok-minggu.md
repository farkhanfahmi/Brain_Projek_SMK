# T056 — UI: Kalender Visual Input Rentang Blok Minggu A/B

## Depends on
T054 (schema & API `block_week_ranges`), T050 (layout dashboard admin_jurnal, dropdown semester harus sudah ada)

## Objective
Buat UI kalender visual untuk admin_jurnal menandai tiap minggu dalam 1 semester sebagai `A` atau `B` — MENGGANTIKAN form input tanggal manual satu-per-satu yang berisiko human error tinggi untuk belasan pasangan rentang per semester.

## Context
- **App:** `apps/web`
- **Route:** `/admin-jurnal/jadwal-blok` (menu baru di sidebar admin_jurnal)
- **Role:** `admin_jurnal`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "📅 Jadwal: Mode Blok (Minggu A/B) vs Mode Normal", terutama open question "UI direkomendasikan kalender visual" yang dijawab oleh task ini
- **⚠️ Ref WAJIB dibaca sebelum menulis UI:** `Projek/AbsenSI/06-Features/design-system/MASTER.md` + `01-colors.md` — sistem desain ini **HANYA punya 1 warna aksen (oranye)**, TIDAK ADA warna kedua (biru/hijau/dst) untuk membedakan kategori seperti "Minggu A vs Minggu B". Baca bagian "Konten Kalender" di bawah untuk cara membedakan A/B tanpa melanggar aturan 1-warna-aksen ini.

## Spec Detail

### Halaman `/admin-jurnal/jadwal-blok`
- **Dropdown "Semester"** di atas (sama pola seperti T050 halaman Jadwal Mengajar) — pilih semester yang mau dikelola rentang bloknya. Halaman ini HANYA relevan kalau semester yang dipilih `mode: blok` — kalau `mode: normal`, tampilkan pesan "Semester ini menggunakan Mode Normal, tidak perlu rentang blok" dan sembunyikan kalender

### Tampilan Kalender
- Grid kalender mingguan membentang dari `semester.tanggal_mulai` sampai `tanggal_selesai` — setiap BARIS mewakili 1 minggu kalender (Senin–Minggu), bukan grid bulanan biasa (supaya jelas per-minggu, sesuai granularitas `block_week_ranges`)
- Tiap baris minggu punya toggle 3 state — **PENTING: sistem desain ini cuma 1 warna aksen (oranye), jadi A/B dibedakan lewat KOMBINASI warna+tekstur, bukan 2 hue berbeda:**
  - **Belum ditandai**: bg `--color-bg-surface-subtle` (netral pucat), tanpa label
  - **A**: bg `--color-primary` (oranye solid, full-strength) + label teks putih "A" — pakai warna aksen penuh, konsisten dengan pola "active state" nav item di `03-components.md`
  - **B**: bg `--color-primary-soft-2` (tint oranye pertengahan, BUKAN warna lain) + label teks `--color-text-primary` "B" — dibedakan dari A lewat KEPEKATAN warna (solid vs tint), bukan hue berbeda, sesuai aturan "monochrome orange ramp" untuk elemen yang butuh beberapa tingkat di `01-colors.md` poin 4
- Klik 1 baris minggu → toggle antar state (kosong → A → B → kosong, siklus) — **setiap toggle langsung memanggil API** (`POST /block-week-ranges` saat set ke A/B, `DELETE /block-week-ranges/:id` saat kembali ke kosong), tidak ada tombol "Simpan Semua" terpisah — perubahan per-baris langsung persisten
- **Progress bar/indikator "Kelengkapan"** di atas kalender — panggil `GET /block-week-ranges/coverage-check?semester_id=` saat halaman dimuat dan setelah tiap toggle, tampilkan persentase minggu yang sudah ditandai vs total minggu dalam semester. Pakai pola **Progress/Target Pill** dari `03-components.md`: track bg `--color-primary-soft`, fill bg `--color-primary`
- Baris minggu yang overlap dengan semester LAIN (hasil tolakan backend 409) → tampilkan pesan error inline di baris itu (bukan alert generik), dengan detail semester/rentang yang bentrok dari response backend

### Interaksi Khusus
- **Highlight visual untuk minggu yang overlap validasi (b)** — kalau admin mencoba toggle minggu yang tanggalnya masih milik semester lain (misal Semester Genap dicoba tandai padahal masih tumpang tindih Semester Ganjil yang berjalan), baris itu tetap kosong SETELAH gagal, dengan border warna `--color-danger-text` (SATU-SATUNYA tempat warna merah dipakai di halaman ini, sesuai aturan "red = negative indicator only") + tooltip pesan error dari backend
- Setelah semua minggu ditandai (coverage 100%), tampilkan badge sukses pakai Badge Pill spec: bg `--color-success-bg`, text `--color-success-text`, label "Rentang blok lengkap untuk semester ini"

## JANGAN
- ❌ JANGAN buat form input tanggal manual biasa (text/date input berbaris) sebagai pengganti kalender visual — ini task yang secara spesifik ada untuk MENGGANTI pendekatan itu karena risiko human error, jangan implementasikan cara lama
- ❌ JANGAN buat tombol "Simpan Semua" yang menunda API call — tiap toggle baris langsung panggil API, supaya admin dapat feedback validasi (termasuk error overlap) SEKETIKA per baris, bukan baru tahu ada kesalahan setelah submit banyak baris sekaligus
- ❌ JANGAN sembunyikan detail rentang/semester yang bentrok saat validasi gagal — tampilkan info spesifik dari response backend (T054 sudah menyediakan detail ini), supaya admin langsung tahu apa yang perlu diperbaiki
- ❌ JANGAN buat halaman ini bisa diakses untuk semester `mode: normal` — sembunyikan kalender, tampilkan pesan penjelasan saja
- ❌ JANGAN pakai warna kedua (biru untuk A, hijau untuk B, atau kombinasi hue lain) untuk membedakan minggu A vs B — sistem desain ini cuma 1 warna aksen oranye, dibedakan lewat kepekatan (solid `--color-primary` vs tint `--color-primary-soft-2`), BUKAN hue berbeda. Ini pelanggaran paling gampang terulang di UI proyek ini, perhatikan betul

## Files
- **Buat:** `apps/web/app/(admin-jurnal)/jadwal-blok/page.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/jadwal-blok/components/kalender-minggu.tsx`
- **Buat:** `apps/web/app/(admin-jurnal)/jadwal-blok/components/coverage-indicator.tsx`
- **Modifikasi:** `apps/web/app/(admin-jurnal)/layout.tsx` — tambah menu sidebar "Jadwal Blok Minggu"

## Acceptance Criteria
- [ ] Pilih semester `mode: blok` → kalender mingguan tampil dari tanggal_mulai sampai tanggal_selesai semester
- [ ] Klik 1 baris minggu → toggle ke A, langsung tersimpan (refresh halaman, state tetap A)
- [ ] Toggle 1 baris minggu ke rentang yang overlap semester lain → gagal, error spesifik tampil di baris itu, state tidak berubah
- [ ] Coverage indicator update setiap kali 1 minggu ditandai/dihapus
- [ ] Pilih semester `mode: normal` → kalender tidak tampil, pesan penjelasan muncul
- [ ] Semua minggu ditandai lengkap → badge "lengkap" muncul, cocok dengan `coverage-check` dari backend
- [ ] Minggu A dan minggu B dibedakan HANYA lewat kepekatan warna oranye (solid vs tint) — tidak ada warna non-oranye (biru/hijau/dll) dipakai untuk membedakan keduanya
