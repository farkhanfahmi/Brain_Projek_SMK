# Task-WEB-004: Badge Jam Masuk Tampilkan Ringkasan Jam Langsung (Tanpa Label)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** low
**Alasan pemilihan:** Perubahan tampilan murni (tambah teks ringkasan di 1 komponen tabel), logic kondensasi jam per-hari sederhana (grouping consecutive days dengan jam sama). Tidak ada perubahan API/skema.

## 2. Konteks & Tujuan Utama

Audit menu Jam Masuk Sekolah (sesi diskusi 2026-09-02): di `TingkatJadwalView` (`apps/web/src/app/(admin)/jadwal/tingkat/[tingkat]/tingkat-jadwal-view.tsx`), tabel kelas HANYA menampilkan badge status ("Override khusus" / "Mengikuti tingkat") tanpa isi datanya — admin harus klik pencil dulu (buka dialog `JamMasukForm`) untuk tahu jam berapa yang sedang berlaku untuk kelas itu. Tidak actionable untuk sekadar cek cepat.

**Referensi desain dari user** — aplikasi JurnalePro (pendahulu AbsenSI di sekolah ini, `https://pro.jurnale.id/panduan/mobile`, screenshot "Ceklok KBM"): pola tampilan JAM POLOS (mis. `07:00 - 08:30 WIB`) TANPA label prefix seperti "Jadwal:"/"Jam Berlaku:" — status (Terlaksana/Berlangsung/Berikutnya) ditandai lewat badge warna TERPISAH dari teks jam, bukan digabung jadi 1 kalimat panjang. Ikuti pola ini: **teks jam selalu polos/mentah, badge status terpisah**.

**Keputusan user:** tampilkan ringkasan jam langsung di badge/baris (tanpa perlu klik), TANPA teks label "Jam Berlaku:" atau sejenisnya — cukup angka jam mentah.

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

1. Di `apps/web/src/app/(admin)/jadwal/tingkat/[tingkat]/tingkat-jadwal-view.tsx`, buat helper function **kondensasi jam per-hari** — input `Schedule[]` (6 baris Senin-Sabtu, tiap baris punya `hari`, `jamMulai`, `jamSelesai`), output string ringkas yang menggabungkan hari-hari BERURUTAN dengan jam SAMA jadi 1 rentang label:
   ```ts
   function ringkasJamMasuk(rows: Schedule[]): string {
     if (rows.length === 0) return "-";
     // Urutkan by hari, group consecutive days dengan jamMulai+jamSelesai identik.
     // Contoh output: "Senin-Jumat 07:00-15:00, Sabtu 07:00-12:00"
     // Contoh output (semua sama): "07:00-15:00"
     // ...implementasi grouping...
   }
   ```
   Cek dulu apakah util serupa sudah ada di `apps/web/src/lib/` (kemungkinan ada helper hari/label lain yang bisa direuse polanya) sebelum menulis dari nol.

2. **Terapkan di 2 tempat** yang punya struktur tabel sama (`JamMasukTable`, dipakai di `jadwal-view.tsx` DAN `tingkat-jadwal-view.tsx` — cek `jadwal-view.tsx` baris 124-156 apakah perlu perubahan serupa untuk konsistensi, meski fokus utama audit di halaman Tingkat):
   - **`TingkatJadwalView`** (tabel kelas, kolom "Status Jam Masuk") — tambahkan teks ringkasan jam di BAWAH atau DI SAMPING badge pill existing (`Override khusus`/`Mengikuti tingkat`), format: badge pill TETAP ada (menandakan sumber aturan), teks jam TAMBAHAN di bawahnya polos tanpa label, mis:
     ```
     [Override khusus]
     07:00-15:00
     ```
     BUKAN:
     ```
     [Override khusus]
     Jam Berlaku: 07:00-15:00
     ```
   - Sumber data: untuk baris "Mengikuti tingkat" pakai `defaultRows` (state induk, sudah ada di scope), untuk baris "Override khusus" pakai `kelas.override` (sudah ada di `kelasList[].override`) — TIDAK perlu fetch data baru, semua sudah tersedia di client state.

3. **Styling** — teks jam pakai token tipografi yang sudah ada (`text-caption text-ink-secondary`, konsisten style existing di file ini), JANGAN bikin token warna/ukuran baru.

4. **Verifikasi mobile-first** — pastikan teks ringkasan tidak overflow/wrap buruk di layar sempit (konvensi proyek: mobile-first, test di viewport kecil dulu).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal/tingkat/[tingkat]/tingkat-jadwal-view.tsx` — tambah helper `ringkasJamMasuk()` + render di kolom Status
- **Modifikasi (opsional, untuk konsistensi):** `apps/web/src/app/(admin)/jadwal/jadwal-view.tsx` — HANYA kalau user/Hermes sepakat halaman Default Global juga perlu ringkasan serupa (didiskusikan saat eksekusi kalau relevan, task ini fokus ke halaman Tingkat)
- **Jangan sentuh:** `JamMasukForm` (dialog edit) — tidak berubah, cuma tampilan tabel ringkasan di luar dialog.

**Dilarang dilakukan:**
- Jangan tambahkan teks label seperti "Jam Berlaku:", "Jadwal:", dsb — SESUAI keputusan eksplisit user, jam ditampilkan polos.
- Jangan hilangkan badge pill status existing (Override khusus/Mengikuti tingkat) — itu tetap informasi penting (sumber aturan), teks jam adalah TAMBAHAN bukan pengganti.

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: kelas belum punya jam sama sekali (data kosong, `rows.length === 0`) → tampilkan "-" atau pesan pendek netral, JANGAN crash/string kosong membingungkan.
- Kondisi: 6 hari SEMUA jamnya beda-beda (tidak ada yang bisa dikondensasi) → tampilkan semua, tapi pertimbangkan format ringkas per-hari kalau terlalu panjang untuk 1 baris (mis. wrap ke beberapa baris, bukan 1 baris super panjang horizontal-scroll).

**Edge case:**
- Data jam dengan format inconsistent (mis. `jamMulai: "07:00:00"` vs `"07:00"`) → verifikasi format `Schedule.jamMulai/jamSelesai` konsisten `HH:mm` di seluruh sumber data (cek DTO/type existing), jangan asumsi format tanpa cek.

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Tabel kelas di halaman Jam Masuk Tingkat menampilkan ringkasan jam langsung (tanpa perlu klik pencil)
- [ ] Teks jam TIDAK ada label prefix ("Jam Berlaku:" dsb) — polos angka jam saja
- [ ] Hari berurutan dengan jam sama dikondensasi jadi 1 rentang label (bukan 6 baris terpisah)
- [ ] Badge pill status existing (Override khusus/Mengikuti tingkat) TETAP ada, tidak dihapus
- [ ] Tampilan tetap rapi di viewport mobile (tidak overflow/wrap buruk)

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan — task ini jauh di bawah itu)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
