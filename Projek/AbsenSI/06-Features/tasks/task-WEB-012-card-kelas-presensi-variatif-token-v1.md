# Task-WEB-012: Card Kelas Presensi Lebih Variatif (Token v1 Existing, Non-Monoton)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi dengan user (screenshot halaman Presensi terasa monoton). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> **KEPUTUSAN EKSPLISIT USER:** styling pakai token v1 yang SUDAH ADA (`packages/ui/src/globals.css`), BUKAN migrasi Design System v2 — waktu develop terbatas, v2 belum punya generator token→CSS (butuh infrastruktur baru dulu, migrasi 63 file `(guru)` diklasifikasi "Kompleksitas Tinggi" di vault). Migrasi v2 PENUH ditunda ke fase terpisah nanti.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Murni styling (className), tidak ada logic baru. Data yang mau ditambahkan (`jumlah_hadir`, dll) SUDAH tersedia dari endpoint calendar (kalau diperluas ke ringkasan per-kelas) atau bisa dilewati kalau dianggap scope creep — task ini fokus VISUAL dulu.

## 2. Konteks & Tujuan Utama

User menilai halaman `/guru/presensi` (step "Pilih Kelas") terasa monoton — semua card kelas berwarna putih polos, icon chip identik (krem-oranye pucat berulang di setiap baris), tidak ada hierarki visual antar kelas.

**Root cause dikonfirmasi Hermes:** proyek belum menerapkan variasi warna sama sekali di komponen ini — token v1 (`globals.css`) sebenarnya SUDAH punya beberapa warna semantic (`success`, `danger`, `status-shipped`, `status-processing`) yang BELUM dimanfaatkan sama sekali di halaman ini, semua card pakai kombinasi warna yang identik.

**Keputusan user:** perbaiki keterbacaan/variasi visual pakai token v1 EXISTING (`primary-soft`, `success-bg`, `status-shipped-bg`, `status-processing-bg`, dst — SEMUA sudah ada di `globals.css`, TIDAK ADA token baru yang perlu dibuat) — BUKAN migrasi ke Design System v2.

## 3. Langkah Eksekusi Detail

1. Di `apps/web/src/app/(guru)/guru/presensi/presensi-view.tsx`, ganti icon chip kelas (baris ~45-47, saat ini SELALU `bg-primary-soft text-primary`) jadi **rotasi warna berbeda per baris** menggunakan token semantic v1 yang SUDAH ADA — buat array warna, pilih berdasarkan index atau hash nama kelas (supaya KONSISTEN per kelas yang sama, bukan random tiap render):
   ```ts
   const CHIP_COLORS = [
     { bg: 'bg-primary-soft', text: 'text-primary' },
     { bg: 'bg-success-bg', text: 'text-success-text' },
     { bg: 'bg-status-shipped-bg', text: 'text-status-shipped-text' },
     { bg: 'bg-status-processing-bg', text: 'text-status-processing-text' },
   ];
   function warnaUntukKelas(nama: string) {
     const hash = nama.split('').reduce((acc, c) => acc + c.charCodeAt(0), 0);
     return CHIP_COLORS[hash % CHIP_COLORS.length];
   }
   ```
   Verifikasi 4 kombinasi ini SEMUA sudah terdaftar di Tailwind config (`packages/config/tailwind.config.ts`) sebagai class yang valid — kalau ada yang belum ter-generate sebagai utility class, cek konfigurasi dulu sebelum dipakai (jangan asumsi otomatis jalan).

2. **Tambahkan sedikit informasi tambahan di card** (opsional, tingkatkan value tanpa cuma ganti warna) — pertimbangkan render `jumlah_hadir`/`jumlah_tidak_ada_di_kelas` TERAKHIR (mis. dari sesi terbaru kelas itu) sebagai teks kecil di card, MEMANFAATKAN data yang sudah dihitung backend (`KalenderHariRow`) tapi belum pernah dirender di mana pun (temuan audit Hermes). **Ini butuh 1 call API tambahan per kelas atau endpoint ringkasan baru — kalau dianggap terlalu besar untuk task ini, SKIP dan laporkan sebagai usulan terpisah**, fokus utama task ini adalah variasi warna.

3. **Pertimbangkan border-left-accent tipis** (`border-l-4`, warna sama dengan chip icon) di card kelas untuk penegasan visual tambahan tanpa terlalu ramai — opsional, terapkan kalau hasil chip warna saja masih terasa kurang variatif setelah screenshot-check.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(guru)/guru/presensi/presensi-view.tsx`

**Dilarang dilakukan:**
- Jangan pakai token/hex baru di luar `globals.css` v1 yang sudah ada — SEMUA 4 kombinasi warna yang diusulkan SUDAH terdaftar (`success`, `status-shipped`, `status-processing`, `primary-soft`), tidak perlu tambah CSS variable baru.
- Jangan migrasi komponen ini ke Design System v2 — KEPUTUSAN EKSPLISIT user, di luar scope task ini.
- Jangan ubah struktur data/API kecuali benar-benar diperlukan untuk poin 2 (ringkasan hadir) — kalau ternyata perlu endpoint baru, itu keputusan terpisah, bukan otomatis dikerjakan tanpa konfirmasi.

**Skenario kegagalan yang WAJIB ditangani:**
- Guru dengan banyak kelas (5+) → warna berulang (siklus 4 warna) TETAP OK, tidak perlu warna unik tak terbatas — pastikan tidak ada 2 kelas BERSEBELAHAN dengan warna sama persis kalau kebetulan collision hash (opsional refinement, bukan hard requirement).
- Nama kelas kosong/aneh (seharusnya tidak terjadi tapi jaga-jaga) → fallback ke warna default (`primary-soft`) kalau hash gagal/nama kosong.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Icon chip kelas punya variasi warna (minimal 4 kombinasi berbeda), konsisten per kelas yang sama
- [ ] Semua warna dari token v1 `globals.css` yang SUDAH ADA, tidak ada CSS baru
- [ ] Halaman terasa lebih variatif secara visual dibanding sebelumnya (verifikasi via screenshot sebelum/sesudah)
- [ ] TIDAK ada migrasi ke Design System v2 di task ini

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (KECUALI poin 2 ringkasan hadir yang eksplisit boleh di-skip kalau dirasa scope creep)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 60 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (v1/v2 coexistence, migrasi bertahap per keputusan product owner)
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
