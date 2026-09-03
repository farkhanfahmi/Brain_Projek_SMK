# Task-WEB-007: Waktu Server Real-time di Bawah Badge Status Tap Gerbang

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi dengan user (referensi visual JurnalePro Ceklok KBM). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Haiku
**Tingkat Effort:** low
**Alasan pemilihan:** Murni tambahan clock berjalan client-side (setInterval + format waktu), tidak ada perubahan backend/API sama sekali.

## 2. Konteks & Tujuan Utama

Referensi user: aplikasi JurnalePro (pendahulu AbsenSI) menampilkan jam berjalan real-time (format `HH:MM:SS WIB`) di header halaman "Ceklok KBM" — membantu guru tahu waktu SEKARANG relatif terhadap jendela waktu sesi mengajarnya, tanpa perlu cek jam HP terpisah.

**Keputusan user:** tambahkan jam berjalan di halaman `/guru/jadwal` (Jadwal Mengajar Hari Ini), diletakkan **di bawah** badge "Belum Tap Gerbang"/"Sudah Tap Gerbang" di header card.

**PENTING — koreksi format:** user menyebut "HH:MM:DD" di permintaan, ini kemungkinan salah ketik — mengacu ke referensi visual JurnalePro yang ditunjukkan (`8:36:19 WIB`), format yang benar adalah **HH:MM:SS** (jam:menit:detik), BUKAN hari (DD). Konfirmasi pemahaman ini ke user di awal eksekusi kalau ternyata memang literally minta tanggal, tapi kemungkinan besar salah ketik dari "SS".

**Depends on:** Tidak ada — independen dari task lain di batch ini.

## 3. Langkah Eksekusi Detail

1. Di `apps/web/src/app/(guru)/guru/jadwal/jadwal-view.tsx`, buat komponen kecil `<JamServerBerjalan />` (client component, bisa inline di file yang sama atau file terpisah `components/jam-berjalan.tsx` mengikuti pola folder `components/` yang sudah ada di situ).
2. Implementasi: `useState` untuk waktu saat ini, `useEffect` dengan `setInterval(1000ms)` untuk update tiap detik, format `toLocaleTimeString("id-ID", { hour: "2-digit", minute: "2-digit", second: "2-digit" })` + suffix `" WIB"` (KONSISTEN dengan seluruh proyek yang sudah asumsikan timezone WIB, JANGAN pakai `Intl.DateTimeFormat` dengan timezone berbeda).
3. **PENTING — ini WAKTU CLIENT (jam device guru), BUKAN waktu server sungguhan** — proyek ini TIDAK punya endpoint "waktu server" khusus (dicek, tidak ada). Menyinkronkan ke waktu server asli via polling endpoint terpisah adalah OVER-ENGINEERING untuk kebutuhan sekadar "tampilkan jam berjalan" — client clock sudah cukup akurat untuk tujuan ini (bukan validasi keamanan, cuma visual). **Beri nama variabel/komponen yang jujur** (`JamBerjalan`, bukan `JamServer`) supaya tidak menyesatkan dev berikutnya mengira ini disinkronkan ke server.
4. Render di bawah badge status (baris `sudahTapGerbang ? ... : ...` di `jadwal-view.tsx`) — style teks kecil (`text-caption text-ink-secondary`, konsisten style existing), format tampilan: `08:36:19 WIB`.
5. **Bersihkan interval saat unmount** (`return () => clearInterval(id)` di `useEffect`) — wajib, mencegah memory leak/warning React saat guru pindah halaman.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(guru)/guru/jadwal/jadwal-view.tsx`
- **Kemungkinan file baru:** `apps/web/src/app/(guru)/guru/jadwal/components/jam-berjalan.tsx` (kalau dipisah, ikuti pola folder existing)

**Dilarang dilakukan:**
- Jangan tambah endpoint backend baru untuk "waktu server" — di luar scope, client clock cukup untuk kebutuhan visual ini.
- Jangan pakai library tambahan (dayjs, date-fns, dll) — `toLocaleTimeString` bawaan JS cukup, KONSISTEN dengan pola proyek yang minim dependency untuk hal sederhana begini.

**Skenario kegagalan yang WAJIB ditangani:**
- Device guru salah setting jam/timezone → di luar kendali aplikasi, tidak perlu ditangani khusus (perilaku sama seperti jam OS mana pun).
- Komponen unmount saat interval masih jalan → WAJIB cleanup, jangan sampai warning "state update on unmounted component" muncul di console.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Jam berjalan tampil di bawah badge status tap gerbang, format `HH:MM:SS WIB`, update tiap detik
- [ ] Tidak ada endpoint API baru ditambahkan
- [ ] Interval dibersihkan saat komponen unmount (tidak ada memory leak)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 50 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
