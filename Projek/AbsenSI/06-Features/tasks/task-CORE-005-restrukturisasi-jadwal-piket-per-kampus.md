# Task-CORE-005 / WEB-002: Restrukturisasi Jadwal Piket Per-Kampus

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> Task ini menyentuh backend (CORE) DAN frontend (WEB) sekaligus — dipertahankan 1 file karena perubahannya saling terkait erat (skema query + UI grid berubah bersamaan), TIDAK dipecah 2 file terpisah kecuali nanti terbukti terlalu besar saat eksekusi.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Restrukturisasi query + UI grid dari 1 grid gabungan jadi per-kampus. Tidak butuh migrasi skema baru (field `User.kampusId` untuk `guru_piket` SUDAH ADA, pola existing "T052 — Wali Kelas, pola sama seperti guru_piket.kampus_id"), tapi tetap perlu ketelitian karena mengubah struktur tampilan+query yang sudah dipakai user secara rutin.

## 2. Konteks & Tujuan Utama

Audit menu Jadwal (sesi diskusi 2026-09-02) menemukan `PiketSchedulesService`/`JadwalPiketView` (grid assign guru piket per hari) **tidak di-scope per kampus sama sekali** — 1 grid tunggal untuk SELURUH sekolah, dropdown assign menampilkan SEMUA `guru_piket` lintas kampus tanpa pengelompokan. Ini janggal karena:

1. Sekolah ini punya **2 kampus fisik** (dikonfirmasi `dashboard-piket.md`: *"Sekolah punya 2 kampus fisik (Kampus 1, Kampus 2), masing-masing dengan guru piket sendiri yang punya komputer/dashboard sendiri"*).
2. Dashboard Piket (fitur lain, sudah ada) SUDAH di-scope per kampus (`attendance:kampus:{id}` channel realtime, data siswa discope kampus).
3. Setiap akun `guru_piket` SUDAH punya `User.kampusId` terisi (field existing, bukan field baru) — tapi jadwal piket (assignment "guru mana piket hari apa") sama sekali tidak memanfaatkan field ini.

**Keputusan user:** restrukturisasi penuh — 1 grid per kampus (bukan sekadar UI grouping/label). Dropdown assign guru HARUS terfilter ke guru piket kampus yang sama dengan grid yang sedang dilihat admin.

**Depends on:** Tidak ada — independen dari task lain di batch ini.

## 3. Langkah Eksekusi Detail

### Backend (`apps/api/src/piket-schedules/`)

1. **Verifikasi constraint unique existing** — `PiketSchedule` punya `@@unique([hari, userId])` (dari controller: `where: { hari_userId: { hari, userId } }`). Constraint ini TIDAK perlu diubah — 1 user tetap cuma bisa di-assign 1x per hari, terlepas dari kampus mana yang sedang dilihat. Pastikan tidak ada asumsi baru yang butuh constraint tambahan.

2. **Tambah filter kampus di `findAll()`** (`piket-schedules.service.ts` baris 15-20) — ubah signature jadi menerima parameter opsional `kampusId`:
   ```ts
   async findAll(kampusId?: number) {
     return this.prisma.piketSchedule.findMany({
       where: kampusId ? { user: { kampusId } } : undefined,
       include: SCHEDULE_INCLUDE,
       orderBy: [{ hari: "asc" }, { createdAt: "asc" }],
     });
   }
   ```
   Pertimbangkan juga endpoint terpisah `GET /piket-schedules?kampusId=X` di controller (query param, konsisten pola endpoint lain di codebase seperti `opsi-jadwal?semesterId=X`).

3. **Validasi di `assign()`** (baris 22-51) — tambahkan pengecekan: guru (`dto.userId`) yang di-assign HARUS punya `kampusId` yang SAMA dengan kampus grid yang sedang diisi. Ini butuh `dto` menerima `kampusId` eksplisit (atau di-derive dari guru yang dipilih — didiskusikan mana yang lebih aman: **kirim `kampusId` dari FE eksplisit lalu validasi backend cocok dengan `user.kampusId` guru terpilih**, supaya backend jadi source of truth, bukan sekadar filter dropdown FE yang bisa dilewati lewat API langsung).
   ```ts
   if (user.kampusId !== dto.kampusId) {
     throw new BadRequestException(
       `Guru ${user.username} terdaftar di kampus lain — hanya guru piket kampus yang sama yang bisa di-assign ke grid ini.`,
     );
   }
   ```
   **Tentukan dulu**: apakah `dto.kampusId` perlu ditambah ke `AssignPiketScheduleDto`, atau backend cukup derive dari `user.kampusId` guru yang dipilih tanpa perlu FE kirim kampusId eksplisit (lebih simpel, FE cuma kirim `userId`+`hari`, backend otomatis tahu kampusnya dari data guru). **Rekomendasi: opsi kedua (derive dari user) lebih simpel dan tidak butuh field baru di DTO** — kecuali ada alasan kuat guru piket tanpa kampusId (`kampusId` nullable di schema) perlu ditangani khusus, lihat skenario kegagalan di bagian 4.

4. **Cek `isBertugasHariIni()`** (baris 66-72, dipakai gate read-only dashboard piket) — method ini TIDAK perlu berubah (sudah benar, scope by `userId` individual, bukan butuh tahu kampus di titik ini).

### Frontend (`apps/web/src/app/(admin)/jadwal-piket/`)

5. **Redesain `JadwalPiketView`** — ubah dari 1 grid 6-hari tunggal jadi **Tab/Selector Kampus** di atas, grid 6-hari di bawahnya berubah sesuai kampus yang dipilih. Pola referensi: mirip `Tabs` yang sudah dipakai di `WorkspaceView` (jadwal-pelajaran) untuk konsistensi visual — bukan komponen baru dari nol.
   - Ambil daftar kampus dari endpoint `Kampus` existing (`apps/api/src/core/kampus/`) — kemungkinan sudah ada `GET /kampus` yang dipakai halaman lain, reuse.
   - State `kampusAktif` di level `JadwalPiketView`, grid + dialog assign HANYA menampilkan/mengoperasikan data kampus yang sedang aktif.

6. **Filter dropdown guru di `HariDialog`** (baris 104-156) — `guruPiket` yang di-pass ke komponen ini harus SUDAH terfilter ke kampus aktif (dari parent), bukan semua guru piket lintas kampus seperti sekarang. Verifikasi prop `guruPiket` dari halaman induk (`page.tsx` — server component yang fetch data awal) sudah query dengan filter kampus, atau filter di client state setelah kampus dipilih.

7. **Pastikan halaman `page.tsx`** (server-side data fetching untuk halaman ini) menyesuaikan — kemungkinan perlu fetch SEMUA piket schedules + SEMUA guru piket (semua kampus) sekaligus di initial load, lalu filter di client saat ganti tab kampus (hindari round-trip network tiap ganti tab, kecuali datanya besar — untuk 2 kampus dan jumlah guru piket yang realistis kecil, fetch-semua-sekaligus lebih simpel).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/piket-schedules/piket-schedules.service.ts` — `findAll()`, `assign()`
- **Modifikasi:** `apps/api/src/piket-schedules/piket-schedules.controller.ts` — tambah query param kampusId (kalau didesain begitu)
- **Modifikasi:** `apps/api/src/piket-schedules/dto/assign-piket-schedule.dto.ts` — HANYA kalau diputuskan `kampusId` dikirim eksplisit dari FE (lihat langkah 3)
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal-piket/jadwal-piket-view.tsx` — restrukturisasi total ke Tab per Kampus
- **Modifikasi:** `apps/web/src/app/(admin)/jadwal-piket/page.tsx` — sesuaikan data fetching awal
- **Jangan sentuh:** Dashboard Piket (fitur terpisah, sudah benar scope kampus), `isBertugasHariIni()`

**Dilarang dilakukan:**
- Jangan hapus data `PiketSchedule` existing — restrukturisasi ini murni UI+query, data assignment yang sudah ada TETAP VALID (asosiasi user→kampus sudah implisit lewat `user.kampusId`, tidak perlu migrasi data).
- Jangan ubah constraint unique `@@unique([hari, userId])` tanpa analisa dampak — kalau ternyata perlu jadi `@@unique([hari, userId, kampusId])` (skenario 1 guru piket di 2 kampus berbeda hari yang sama — TIDAK REALISTIS karena guru piket biasanya 1 kampus tetap, tapi verifikasi asumsi ini eksplisit sebelum ubah skema).

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: ada `guru_piket` existing dengan `kampusId: null` (belum pernah diisi) → **[VERIFIKASI SELESAI 2026-09-02, oleh Hermes via SSH read-only ke production]** Query `SELECT id, username, kampus_id FROM users WHERE role='guru_piket'` dijalankan langsung ke database production (`10.10.10.198`, tabel `users`, 20 baris total) — **HASIL: TIDAK ADA satu pun baris dengan `kampus_id` NULL.** Semua 20 akun `guru_piket` sudah punya `kampus_id` terisi (10 akun `kampus_id=1`, 10 akun `kampus_id=2` — rincian di bawah). **Kesimpulan: skenario edge case ini TIDAK PERLU ditangani khusus untuk eksekusi task ini** — tidak ada guru piket "yatim" tanpa kampus di data real saat ini. Tetap boleh implementasi validasi/handling untuk `kampusId: null` sebagai defensive coding (best practice, skema tetap `nullable`), TAPI tidak perlu treat sebagai blocker atau minta keputusan user dulu — datanya sudah bersih.

  Detail 20 baris (`id | username | kampus_id`): 5|hilma|2, 19|brizan|1, 20|aditia|1, 21|soelasmi|2, 22|niko stiawan|2, 23|fitriani|1, 24|ratih|1, 25|wahyu|2, 26|arif|2, 27|lanang|1, 28|atik|1, 29|iie|2, 30|wahyuningtyas|2, 33|alfin|1, 34|alyaa|1, 35|gita|2, 36|nanang|2, 38|siska ari|1, 163|ivrada|1, 164|denny|1.
- Kondisi: assignment `PiketSchedule` LAMA (sebelum restrukturisasi) untuk guru yang kampusId-nya SUDAH BERUBAH sejak assignment dibuat (mis. guru pindah kampus) → data lama tetap tampil di kampus BARU guru itu (karena scope-nya ikut `user.kampusId` real-time, bukan snapshot saat assign) — pastikan behavior ini disengaja dan dikomunikasikan (bukan bug), karena artinya histori assignment "pindah kampus" bersama gurunya, bukan tertinggal di kampus lama.
- Kondisi: admin_jurnal atau role lain (bukan super_admin) mencoba assign guru piket dari kampus berbeda lewat API langsung (bypass UI) → validasi backend (langkah 3) WAJIB menolak, bukan cuma andalkan filter UI.

**Edge case:**
- Sekolah nanti tambah kampus ke-3 → desain Tab per Kampus harus scalable (render dari daftar kampus dinamis, BUKAN hardcode "Kampus 1"/"Kampus 2").

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Halaman Jadwal Piket menampilkan grid TERPISAH per kampus (Tab/Selector), bukan 1 grid gabungan
- [ ] Dropdown assign guru piket HANYA menampilkan guru dari kampus yang sedang aktif dilihat
- [ ] Backend menolak (400) percobaan assign guru piket lintas kampus meski dipanggil langsung via API
- [ ] Data `PiketSchedule` existing tetap utuh, otomatis tampil di kampus yang benar berdasarkan `user.kampusId` guru terkait
- [x] ~~Kasus `guru_piket` dengan `kampusId: null`~~ — **tidak relevan**, terverifikasi 2026-09-02 tidak ada baris NULL di production (lihat detail di bagian 4). Boleh tetap ditangani sebagai defensive coding, bukan syarat wajib.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; kalau FE grid redesign ternyata > itu, pertimbangkan pecah CORE-005 backend dan WEB-002 FE jadi 2 file terpisah saat eksekusi)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (konsisten pola scope-per-kampus di Dashboard Piket)
- [x] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency; verifikasi data production (`guru_piket` dengan kampusId null) **SUDAH DILAKUKAN 2026-09-02 via SSH read-only, hasil: 0 baris NULL** (lihat bagian 4) — task ini SIAP dieksekusi tanpa blocker data.
