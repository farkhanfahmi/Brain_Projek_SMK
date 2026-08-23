# T143 — Schema+API+Web: Menu Setting Tampilan TV Piket (Toggle Per-Blok Per-Kampus) + Blok Baru "Izin Belum Kembali"

## Depends on
Tidak ada dependency teknis ke T139-T142. Independen.

## Objective
1. Admin (super_admin) bisa mengatur, PER KAMPUS, blok mana yang tampil di layar TV Piket (`/tv-piket/[kampusId]`) dari toggle sederhana — tidak perlu ubah kode untuk sekolah yang tidak butuh blok tertentu.
2. Tambah blok BARU "Siswa Izin Keluar Belum Kembali" (data SUDAH ada, tinggal reuse — lihat Spec Detail).
3. Blok yang AKTIF (toggle on) tapi datanya KOSONG saat itu → otomatis TIDAK dirender (hilang dari layar), sisa blok reflow mengisi ruang.
4. Blok yang isinya melebihi tinggi kapasitasnya → font mengecil otomatis (bukan scroll), independen per blok.

## Context — Keputusan Final (Didiskusikan 2026-08-08, JANGAN Ditafsirkan Ulang)

1. **Scope setting HANYA toggle tampil/sembunyi per blok** — TIDAK ada drag-reorder posisi, TIDAK ada pilih-kolom-per-blok. Urutan/posisi blok tetap seperti didesain di task ini (fixed di kode), admin cuma ON/OFF per blok.
2. **Config PER KAMPUS** — tiap kampus punya setting sendiri-sendiri, independen dari kampus lain.
3. **Auto-hide blok kosong** — kalau toggle AKTIF tapi query hasilnya 0 baris/item saat itu (misal tidak ada siswa izin keluar belum kembali), blok itu TIDAK dirender sama sekali di layar TV saat itu (bukan tampil dengan "tidak ada data" kosong). Ini DINAMIS per refresh data (bisa muncul lalu hilang lagi sepanjang hari tergantung kondisi real-time).
4. **Reflow layout**: kalau sebuah blok hilang (karena toggle off ATAU auto-hide kosong), blok LAIN yang sebaris dengannya melebar mengisi ruang (100%), TIDAK ADA ruang kosong dibiarkan. Kalau SEMUA blok di 1 baris hilang, baris itu sendiri hilang total dari layout.
5. **Auto-shrink teks, BUKAN scroll** — task ini MENGHAPUS mekanisme scroll yang ada sekarang (`max-h-[520px] overflow-y-auto` + auto-scroll animasi 8 detik di blok Siswa Tidak Hadir, dan `overflow-y-auto` statis di `ActivityFeedCard`) dan MENGGANTINYA dengan mekanisme font-size yang mengecil bertahap kalau konten melebihi tinggi blok yang tersedia. TV adalah display pasif (tidak ada yang scroll manual), jadi SEMUA konten HARUS selalu terlihat penuh di layar tanpa terpotong — mengecilkan font adalah caranya, bukan menyembunyikan sebagian konten.
6. **Blok "Izin Belum Kembali" — privasi data**: tampilkan HANYA `alasanKategori` (enum terstruktur, misal "Sakit"/"Keperluan Keluarga"), **JANGAN tampilkan `alasanDetail`** (teks bebas, berisiko berisi info pribadi/medis sensitif — layar TV ini dilihat publik: siswa, orang tua tamu, siapa saja yang lewat lorong).
7. **Ukuran TV 42 inch** — TIDAK ADA preseden kode untuk ukuran layar spesifik ini saat ini (dikonfirmasi riset, murni Tailwind token biasa tanpa breakpoint TV-khusus). Task ini TIDAK PERLU membuat breakpoint/media-query baru khusus 42 inch — cukup pastikan mekanisme auto-shrink (poin 5) membuat konten SELALU muat di viewport yang ada, apa pun ukurannya secara proporsional. Kalau nanti ternyata 42 inch spesifik butuh penyesuaian ukuran dasar (base font/spacing), itu task terpisah, BUKAN scope ini.

## Spec Detail

### 1. Schema (Prisma) — model config BARU, pola "1 baris per kampus" (BELUM ADA preseden di codebase, ini yang PERTAMA)

```prisma
model TvPiketDisplayConfig {
  id                        Int      @id @default(autoincrement())
  kampusId                  Int      @unique @map("kampus_id")
  tampilBarPersentase       Boolean  @default(true) @map("tampil_bar_persentase")
  tampilSiswaTidakHadir     Boolean  @default(true) @map("tampil_siswa_tidak_hadir")
  tampilGuruBelumMulai      Boolean  @default(true) @map("tampil_guru_belum_mulai")
  tampilGuruIzin            Boolean  @default(true) @map("tampil_guru_izin")
  tampilSiswaIzinBelumKembali Boolean @default(true) @map("tampil_siswa_izin_belum_kembali")
  updatedById               Int      @map("updated_by")
  updatedAt                 DateTime @updatedAt @map("updated_at")

  kampus    Kampus @relation(fields: [kampusId], references: [id])
  updatedBy User   @relation(fields: [updatedById], references: [id])

  @@map("tv_piket_display_config")
}
```

- `@@unique([kampusId])` — ENFORCE 1 baris per kampus di level DB (BEDA dari pola singleton existing `AttendanceLockConfig` dkk yang `id` selalu 1 — di sini `kampusId` yang unique, `id` autoincrement biasa).
- Default SEMUA `true` — kampus yang belum pernah di-setting admin otomatis menampilkan SEMUA 5 blok (behavior sama seperti sekarang, sebelum fitur ini ada — regresi nol untuk kampus yang belum disentuh).
- Migration baru.

### 2. Backend — Modul baru `tv-piket-display-config`
Ikuti PERSIS struktur `apps/api/src/attendance-lock-config/` (controller+service+dto+module) sebagai referensi pola:
- `GET /tv-piket-display-config/:kampusId` — return config kampus itu (auto-create dengan default semua `true` kalau belum ada baris, JANGAN error 404 — kampus baru harus otomatis dapat behavior "semua blok tampil").
- `PATCH /tv-piket-display-config/:kampusId` — `@Roles(UserRole.super_admin)` SAJA, `@LogActivity` wajib (snapshot before/after, pola sama `AttendanceLockConfigService`).
- DTO: 5 field boolean opsional (`tampilBarPersentase?`, `tampilSiswaTidakHadir?`, `tampilGuruBelumMulai?`, `tampilGuruIzin?`, `tampilSiswaIzinBelumKembali?`).

### 3. Backend — `TvPiketService` (`apps/api/src/tv-piket/tv-piket.service.ts`) — perluas response
- Inject `TvPiketDisplayConfigService` (config baru dari poin 2) DAN `PermitsService` (untuk reuse `findBelumKembali`, sudah ada method publik `findBelumKembali(kampusId)` di `permits.service.ts:77-88` — PANGGIL LANGSUNG, JANGAN tulis ulang query yang sama).
- Method utama (yang menghasilkan response `GET /tv-piket/data`) sekarang:
  1. Baca config kampus itu (`TvPiketDisplayConfigService.get(kampusId)`).
  2. Untuk SETIAP blok yang togglenya `true` — hitung datanya SEPERTI SEKARANG (persentase/siswaTidakHadir/guruBelumMulai/guruIzin sudah ada logic-nya, TIDAK diubah) DITAMBAH data baru `siswaIzinBelumKembali` dari `PermitsService.findBelumKembali(kampusId)`.
  3. Untuk blok yang togglenya `false` — JANGAN hitung query-nya sama sekali (skip, hemat resource), field itu `null`/tidak ada di response.
  4. Untuk blok yang togglenya `true` TAPI hasil kosong (array `[]` atau untuk persentase kalau total siswa 0) — TETAP kirim di response (array kosong), **frontend yang memutuskan hide** (bukan backend yang menyembunyikan) — supaya frontend tetap tahu blok itu "aktif tapi kosong" vs "memang dimatikan admin", dua kondisi beda yang perlu dibedakan di response.
- **Response type baru**: tambah field `config: { tampilBarPersentase: boolean, tampilSiswaTidakHadir: boolean, tampilGuruBelumMulai: boolean, tampilGuruIzin: boolean, tampilSiswaIzinBelumKembali: boolean }` di response `TvPiketData` — supaya frontend tahu toggle mana yang aktif tanpa fetch terpisah.
- Field `siswaIzinBelumKembali` di response — MAPPING dari hasil `findBelumKembali()` (yang return `Permit[]` lengkap dengan relasi student) KE bentuk ringkas untuk TV (JANGAN kirim raw `Permit` object apa adanya, itu bocor field sensitif `kodeVerifikasi`/`buktiFilePath`):
  ```ts
  {
    studentId: number; nama: string; kelas: string;
    alasanKategori: string; // ENUM saja, BUKAN alasanDetail
    jamKeluar: string; // format jam
    jamKembaliDiharapkan: string; // format jam
    terlambatMenit: number; // now - jamKembaliDiharapkan, dalam menit
  }
  ```

### 4. Frontend — Menu Setting Admin (halaman BARU)
- Lokasi: `apps/web/src/app/(admin)/tv-piket-setting/` (nama folder putuskan konsisten pola existing, cek penamaan folder admin lain dulu) — TAMBAH ke sidebar admin (grup yang relevan, kemungkinan dekat menu Kiosk/TV Session yang sudah ada).
- **Per kampus** — halaman ini butuh SELECTOR kampus dulu (dropdown pilih kampus mana yang mau di-setting, ATAU kalau sudah ada halaman admin lain yang scoped-per-kampus, ikuti pola yang sama — cek `kiosk-view.tsx`/`kampus-view.tsx` untuk pola existing).
- Untuk kampus yang dipilih: tampilkan 5 toggle switch (REUSE komponen toggle custom yang sudah dipakai `pengaturan-absensi-view.tsx`), label jelas per blok: "Bar Persentase Kehadiran", "Siswa Tidak Hadir", "Guru Belum Mulai Mengajar", "Guru Izin Hari Ini", "Siswa Izin Keluar Belum Kembali".
- Setiap toggle langsung `PATCH` saat diubah (tidak perlu tombol Simpan terpisah, pola sama config lain).

### 5. Frontend — Halaman TV Piket (`apps/web/src/app/tv-piket/[kampusId]/page.tsx`) — rombak render + layout

**a. Render kondisional per blok** — bungkus render tiap blok dengan kondisi GANDA: `config.tampilXxx === true` DAN `data tidak kosong` (lihat detail per blok di bawah). Kalau salah satu `false`, blok itu tidak dirender (`null`, bukan `display:none` — supaya benar-benar tidak makan ruang layout).

**b. Reflow layout** (baris tengah 60/40 antara "Siswa Tidak Hadir" dan "Guru Belum Mulai Mengajar"):
- Kedua blok tampil → tetap 60/40 seperti sekarang.
- HANYA "Siswa Tidak Hadir" tampil (Guru Belum Mulai hilang) → "Siswa Tidak Hadir" jadi `w-full` (100%).
- HANYA "Guru Belum Mulai Mengajar" tampil → jadi `w-full` (100%).
- KEDUANYA hilang → baris flex itu sendiri tidak dirender sama sekali (tidak ada gap kosong).
- Blok "Bar Persentase" dan "Guru Izin Hari Ini" (masing-masing full-width sendiri di root flex-column) — kalau hilang, otomatis tidak makan ruang (root sudah `flex-col`, tidak perlu logic tambahan, cukup `null` conditional).
- **Blok baru "Siswa Izin Belum Kembali"** — PUTUSKAN posisinya di layout: REKOMENDASI taruh sebagai baris baru di root flex-column, antara blok "Guru Izin Hari Ini" (bawah) — ATAU gabung ke baris tengah sebagai kolom ke-3 (ubah 60/40 jadi proporsi 3-arah kalau blok ini aktif). **Klarifikasi ke user sebelum implementasi kalau ragu** — task ini TIDAK memutuskan tata letak visual pastinya (cuma prinsip toggle+reflow), karena ini soal selera visual bukan soal ambiguitas fungsional. Kalau harus mengambil keputusan tanpa bisa tanya, REKOMENDASI: baris baru full-width di bagian PALING BAWAH (setelah "Guru Izin Hari Ini"), konsisten pola `ActivityFeedCard` yang sudah ada.

**c. Hapus mekanisme scroll, ganti auto-shrink**:
- HAPUS `useAutoScroll` (`apps/web/src/lib/use-auto-scroll.ts`) dan pemakaiannya di blok Siswa Tidak Hadir — HAPUS `max-h-[520px] overflow-y-auto`.
- HAPUS prop `maxHeight` fixed (`"520px"`/`"240px"`) di pemanggilan `ActivityFeedCard` — GANTI dengan mekanisme baru: setiap blok mengukur tinggi kontainer yang tersedia (via `ResizeObserver` atau `useLayoutEffect` + `ref`, hitung ulang tiap kali `data` berubah) VS tinggi konten aktual, kalau konten lebih tinggi dari kontainer → turunkan `font-size` (dan idealnya `line-height`/`padding` proporsional) dalam beberapa tingkat diskrit (misal 100% → 85% → 70% → 55%, berhenti di batas bawah yang masih terbaca, JANGAN sampai mengecil tak terbatas hingga tidak terbaca — tentukan batas minimum wajar saat implementasi, uji visual).
- Buat 1 hook/komponen REUSABLE untuk mekanisme auto-shrink ini (dipakai oleh SEMUA blok: `DataTableCard` Siswa Tidak Hadir, `ActivityFeedCard` ×3 termasuk blok baru) — JANGAN duplikasi logic resize-observer di setiap blok terpisah.
- `DataTableCard`/`ActivityFeedCard` (`packages/ui/src/components/`) — évaluasi apakah perlu modifikasi komponen shared ini untuk terima prop baru (misal `autoShrink?: boolean`) ATAU cukup wrapper di level halaman TV Piket saja (TIDAK mengubah komponen shared, supaya tidak mempengaruhi pemakaian komponen ini di halaman LAIN yang bukan TV — cek dulu apakah `DataTableCard`/`ActivityFeedCard` dipakai di luar TV Piket, kalau ya WAJIB approach wrapper-only, JANGAN ubah shared component behaviornya).

### 6. Frontend — update type `TvPiketData` di `apps/web/src/lib/use-tv-piket-data.ts`
Sinkronkan dengan response backend baru (field `config`, field `siswaIzinBelumKembali`).

## Edge Cases
- Kampus baru dibuat (belum pernah di-setting admin) → `GET /tv-piket-display-config/:kampusId` auto-create baris default semua `true`, TV kampus itu tetap menampilkan semua 5 blok seperti behavior sebelum fitur ini ada (regresi nol).
- SEMUA toggle di-nonaktifkan admin (kasus ekstrem, mungkin tidak disengaja) → halaman TV kosong total (tidak ada blok apa pun) — TIDAK PERLU validasi "minimal 1 blok aktif" di backend/frontend (biarkan admin bertanggung jawab atas pilihannya, ini pengaturan personalisasi bukan keamanan), TAPI pastikan halaman tidak crash/blank-error, cukup tampilkan container kosong dengan rapi (atau pesan kecil "Tidak ada blok aktif" kalau terasa lebih baik UX-nya).
- Auto-shrink mencapai batas minimum font TAPI konten TETAP melebihi tinggi kontainer (kasus ekstrem, misal 50 siswa tidak hadir sekaligus) → terima kondisi ini (konten terpotong di batas font minimum) DAN jangan sampai mengecilkan tak terbatas — ini keterbatasan wajar sebuah display pasif, bukan bug yang harus diselesaikan sempurna.
- Data `siswaIzinBelumKembali` — pastikan query `findBelumKembali` yang di-reuse SUDAH benar hanya untuk kampus yang diminta (`kampusId` dari `req.tvPiket.kampusId` guard TV, BUKAN dari kampus user piket manapun) — verifikasi tidak ada kebocoran data lintas kampus.

## Files
- **Buat:** migration Prisma baru, modul `apps/api/src/tv-piket-display-config/` (controller+service+dto+module), hook/komponen auto-shrink reusable baru di frontend, halaman admin baru `apps/web/src/app/(admin)/tv-piket-setting/`.
- **Modifikasi:** `apps/api/prisma/schema.prisma` (model baru + relasi `Kampus`/`User`), `apps/api/src/tv-piket/tv-piket.service.ts` (inject config+permits service, response diperluas), `apps/api/src/tv-piket/tv-piket.module.ts` (import module baru), `apps/web/src/app/tv-piket/[kampusId]/page.tsx` (render kondisional+reflow+auto-shrink, HAPUS scroll lama), `apps/web/src/lib/use-tv-piket-data.ts` (type diperluas), sidebar admin (menu baru).
- **Hapus:** `apps/web/src/lib/use-auto-scroll.ts` KALAU tidak dipakai di tempat lain (grep dulu sebelum hapus — kalau dipakai halaman TV lain seperti `/tv`, JANGAN hapus filenya, cukup hapus pemakaiannya di TV Piket).
- **Jangan sentuh:** `apps/web/src/app/tv/` (TV Kepsek, halaman BERBEDA, di luar scope task ini), `PermitsService.findBelumKembali()` (reuse apa adanya, jangan diubah — dipakai juga dashboard piket, breaking change di situ akan merusak fitur lain).

## Acceptance Criteria
- [x] Model `TvPiketDisplayConfig` per-kampus (`@@unique([kampusId])`) dengan 5 toggle, default semua `true`.
- [x] Menu admin baru bisa toggle per-blok per-kampus, `@Roles(super_admin)` saja, log tercatat (lihat catatan `@LogActivity` di Status Eksekusi).
- [x] Blok baru "Siswa Izin Keluar Belum Kembali" tampil data (Nama, Kelas, kategori alasan SAJA bukan detail, jam keluar, jam kembali diharapkan, keterlambatan) — reuse `PermitsService.findBelumKembali()`.
- [x] Toggle OFF → blok itu tidak dirender di TV, ruang direflow (blok sebaris melebar/baris hilang total).
- [x] Toggle ON tapi data kosong saat itu → blok otomatis tidak dirender (dinamis, bisa muncul-hilang sepanjang hari).
- [x] Scroll (auto-scroll animasi + overflow manual) SEPENUHNYA dihapus dari TV Piket, diganti auto-shrink font per blok, mekanisme reusable (1 hook `useAutoShrinkText`, dipakai semua blok).
- [x] Kampus baru (belum di-setting) otomatis semua blok tampil (regresi nol untuk kampus existing yang belum disentuh admin).
- [x] `alasanDetail` TIDAK PERNAH dikirim ke response TV Piket untuk blok Izin Belum Kembali (privasi, diverifikasi PROGRAMATIS lewat script — `JSON.stringify(response).includes("alasanDetail")` = `false`).
- [x] `DataTableCard`/`ActivityFeedCard` shared component — grep konfirmasi HANYA dipakai di TV Piket, jadi diubah langsung (bukan wrapper-only), tidak ada halaman lain yang bisa regresi.
- [x] Build + type-check `apps/api` dan `apps/web` hijau.

## Validasi Claudian
- [x] **JANGAN** tampilkan `alasanDetail` di blok Izin Belum Kembali — cuma `alasanKategori`, diverifikasi programatis (lihat atas).
- [x] **JANGAN** buat scope setting lebih dari toggle on/off — DTO hanya 5 field boolean opsional, tidak ada reorder/pilih-kolom.
- [x] **Posisi visual blok baru "Izin Belum Kembali"** — diklarifikasi ke user 2026-08-08: **gabung baris tengah jadi 3 kolom** (bukan baris baru di bawah). Layout: 3 blok tampil = masing-masing ~34%, 2 blok = 50/50, 1 blok = 100%.
- [x] Cek dulu apakah `DataTableCard`/`ActivityFeedCard` dipakai di halaman LAIN — grep konfirmasi HANYA TV Piket, approach ubah-langsung (bukan wrapper-only).
- [x] Cek dulu apakah `use-auto-scroll.ts` dipakai halaman lain — grep konfirmasi HANYA TV Piket, file dihapus total.
- [ ] Test visual auto-shrink dengan data BANYAK/SEDIKIT — **BELUM diverifikasi visual di browser** (lihat catatan gap di Status Eksekusi).

## Status Eksekusi (2026-08-08)
- Schema: migration `20260808034416_t143_tv_piket_display_config` diterapkan bersih ke dev DB. Model `TvPiketDisplayConfig` + back-relation di `Kampus`/`User`.
- Backend: modul baru `apps/api/src/tv-piket-display-config/` (controller+service+dto+module), pola diikuti dari `attendance-lock-config/` PERSIS kecuali 1 hal: **TIDAK pakai decorator `@LogActivity`**, melainkan panggil `activityLogService.record()` manual di service (sama seperti `AttendanceLockConfigService.update()`) — alasan: `ActivityLogInterceptor.fetchSnapshot()` hardcode `findUnique({ where: { id: Number(paramId) } })`, sedangkan route param di sini adalah `kampusId` (bukan `id` PK config), jadi decorator `idParam: "kampusId"` akan salah fetch snapshot-before. Manual call menghindari mismatch ini sekaligus tetap mencatat log dengan benar (diverifikasi record tersimpan lewat script).
- `TvPiketService.getData()` diperluas: inject `TvPiketDisplayConfigService` + `PermitsService`, tiap blok dihitung HANYA kalau togglenya true (Promise.all kondisional), field jadi `null` kalau blok off. Method baru `hitungSiswaIzinBelumKembali()` reuse `PermitsService.findBelumKembali(kampusId)`, mapping ke bentuk ringkas (alasanKategori SAJA, TIDAK PERNAH alasanDetail).
- Frontend: halaman admin baru `(admin)/tv-piket-setting/` (selector kampus + 5 toggle switch, pola reuse dari `pengaturan-absensi-view.tsx`), menu ditambahkan ke grup sidebar "Kartu & Perangkat". Halaman TV Piket (`tv-piket/[kampusId]/page.tsx`) dirombak total: render kondisional (`config.tampilXxx && data tidak kosong`), reflow baris tengah 2/3 kolom, `useAutoShrinkText` hook baru (`apps/web/src/lib/use-auto-shrink-text.ts`, ResizeObserver-based, 4 level shrink 100%→85%→70%→55%) dipakai di semua 4 blok. `ActivityFeedCard` diubah langsung (hapus prop `maxHeight`/scroll internal, tambah prop `listRef`) — grep konfirmasi hanya dipakai TV Piket, aman. `use-auto-scroll.ts` dihapus total (grep konfirmasi tidak dipakai tempat lain).
- Verifikasi: `tsc --noEmit` bersih `apps/api` & `apps/web`. `jest` 203/203 pass. Live: dev DB tidak punya `TvSession` (token) aktif untuk uji end-to-end via browser/HTTP auth penuh, dan tidak ada kredensial login `super_admin` tersedia di sesi ini — sebagai gantinya, verifikasi dilakukan lewat script `NestFactory.createApplicationContext` yang memanggil `TvPiketService`/Prisma langsung (bypass HTTP), mengonfirmasi: (a) default config semua-true untuk kampus tanpa row, (b) toggle off → field jadi `null` di response, (c) `alasanDetail` tidak pernah muncul di response (cek string match pada JSON lengkap), (d) unique constraint `kampusId` bekerja. Data test dibersihkan (row `tv_piket_display_config` dihapus) setelah verifikasi.
- **GAP yang belum tertutup**: mekanisme auto-shrink (`useAutoShrinkText`) belum diuji visual di browser sungguhan (tidak ada TvSession token untuk akses halaman TV Piket di sesi ini) — logic ResizeObserver+level-shrink sudah benar secara statis/tipe, tapi perilaku visual aktual (apakah transisi antar level terlihat wajar, apakah 55% masih terbaca di TV 42 inch) BELUM divalidasi mata langsung. Rekomendasi: uji manual saat ada akses fisik ke TV Piket atau setelah `TvSession` token dibuat via UI admin.
