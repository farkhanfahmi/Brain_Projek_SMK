# T232 — API+Web: Generate Akun Guru Massal (username=NIY, password default)

## Depends on
Tidak ada dependency teknis. Independen, murni modul `users`/`teachers`.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-21)

**Fitur ini BELUM ADA SAMA SEKALI** — dikonfirmasi tidak ada endpoint, service method, maupun UI untuk "buat akun massal berdasarkan guru yang sudah ada". Yang ADA:
- `UsersService.create()` (`apps/api/src/users/users.service.ts:45-73`) — create 1 akun manual per request, `SALT_ROUNDS = 10` (bcrypt).
- `ImportService.importUsers()` (`apps/api/src/import/import.service.ts:562-671`) — bulk via CSV, TAPI username+password harus diisi MANUAL per baris di file, TIDAK ADA logic "skip guru yang sudah punya akun" atau "auto-generate dari NIY".
- Halaman `(admin)/guru/guru-view.tsx` — CRUD biodata Teacher saja, TIDAK ADA kolom status akun/tombol buat akun.
- Halaman `(admin)/akun/akun-view.tsx` — CRUD User manual + import CSV, TIDAK ADA fitur assign-akun-otomatis-dari-teacher-tanpa-user.

**Relasi `User.teacherId → Teacher.id`**: opsional, **1 Teacher BOLEH punya BANYAK User** (dikonfirmasi komentar eksplisit `users.service.ts:243`: "TIDAK ada constraint 1-guru-1-akun: 1 Teacher boleh punya akun guru DAN guru_piket terpisah") — task ini generate akun dengan **role `guru`** SAJA, TIDAK PERLU cek User lain dengan role berbeda untuk Teacher yang sama.

**`mustChangePassword`**: field ada, default `false`, di-set berdasar `ForcePasswordChangeConfigService.shouldForceFor(role)` (toggle admin per-role, `config.forceGuru`) di `UsersService.create()` — BEDA dari `importUsers()` yang HARDCODE `true` tanpa cek config.

## Keputusan Dikonfirmasi User (2026-08-21)

1. **Target**: guru (`Teacher`) yang punya NIY **valid** (murni digit angka) DAN **belum punya** `User` dengan role `guru` terhubung ke `teacherId`-nya.
2. **Username = NIY**, **password awal = "12345678"** (sama untuk semua akun yang di-generate).
3. **`mustChangePassword`**: IKUTI toggle `ForcePasswordChangeConfig.forceGuru` yang SUDAH ADA (KONSISTEN pola `UsersService.create()` satu-per-satu, BUKAN hardcode `true` seperti `importUsers()`) — TIDAK membuat behavior baru yang berbeda dari cara akun guru manual dibuat.
4. **NIY dummy/placeholder** — pola dikonfirmasi: NIY yang **BUKAN murni digit angka** (contoh nyata: `"TANPA_NIY_19"`, mengandung huruf+underscore) dianggap belum diisi sungguhan — guru dengan NIY seperti ini **DI-SKIP dari generate**, DAN **tampil pesan actionable terpisah** di ringkasan hasil ("NIY guru [nama] masih berupa placeholder [TANPA_NIY_19] — perbaiki dulu NIY di menu Guru sebelum generate akun") supaya admin tahu HARUS memperbaiki data dulu, BUKAN cuma "gagal" tanpa penjelasan.
5. **NIY bentrok dengan username akun lain yang sudah ada** (kasus jarang) — SKIP guru itu, lanjut ke guru lain (partial success, KONSISTEN pola `importUsers()`), error dilaporkan jelas per-guru di ringkasan hasil.

## Spec Detail

### 1. Backend — validasi NIY numerik murni (helper reusable)

- Buat helper `isNiyValidForGenerate(niy: string): boolean` — regex `/^\d+$/` (murni digit, TIDAK ADA huruf/underscore/simbol) — REKOMENDASI taruh di `TeachersService` atau shared util, dipakai method generate (poin 2).

### 2. Backend — method generate akun massal

`UsersService` (atau service baru kalau lebih rapi terpisah, VERIFIKASI SAAT IMPLEMENTASI) — method baru `generateAkunGuruMassal(actorId: number, ipAddress: string | null)`:

1. Query SEMUA `Teacher` yang **TIDAK** punya `User` dengan `role: "guru"` terhubung (`teacherId`) — `Teacher.findMany({ where: { users: { none: { role: "guru" } } } })` (VERIFIKASI SAAT IMPLEMENTASI query Prisma relasi negatif ini benar, `none` filter tepat).
2. Untuk TIAP Teacher hasil query:
   - Kalau NIY **BUKAN** murni digit (poin 1) — masuk kategori `niyBelumValid`, TIDAK diproses generate, catat di laporan dengan pesan actionable (poin 4 Keputusan).
   - Kalau NIY valid — coba `create()` User baru: `username: teacher.niy`, `passwordHash: bcrypt.hash("12345678", 10)` (KONSISTEN `SALT_ROUNDS = 10` existing), `role: "guru"`, `teacherId: teacher.id`, `mustChangePassword: await forcePasswordChangeConfig.shouldForceFor("guru")` (REUSE service yang SAMA dipakai `UsersService.create()`, JANGAN hardcode).
   - **Tangkap `P2002` (unique constraint username)** per-guru — SKIP guru itu (JANGAN gagalkan seluruh proses), catat di laporan kategori `usernameBentrok`.
3. **PARTIAL SUCCESS, BUKAN transactional all-or-nothing** — KONSISTEN pola `importUsers()` (Pola B dari riset), BUKAN pola `kenaikanMassal()` (Pola A, all-or-nothing) — REKOMENDASI KUAT karena task ini SECARA SIFAT sama seperti import: banyak baris independen, 1 gagal TIDAK BOLEH menggagalkan yang lain.
4. Return summary (REPLIKASI shape `ImportReport` yang SUDAH ADA di `packages/types/src/index.ts` KALAU cocok, atau bentuk serupa):
   ```ts
   interface GenerateAkunGuruResult {
     totalDiproses: number;       // total Teacher tanpa akun guru yang ditemukan
     berhasil: number;
     niyBelumValid: { teacherId: number; nama: string; niy: string }[];  // actionable: minta admin perbaiki NIY dulu
     usernameBentrok: { teacherId: number; nama: string; niy: string; alasan: string }[];
   }
   ```
5. **Log aktivitas SEKALI per operasi bulk** (KONSISTEN pola `kenaikanMassal()`/`ForcePasswordChangeConfigService.update()` — `ActivityLogService.record()` manual, BUKAN decorator `@LogActivity` per-row, sesuai aturan CLAUDE.md untuk operasi bulk) — `snapshotAfter` berisi ringkasan hasil (jumlah berhasil/skip, BUKAN daftar password — JANGAN log password meski sudah di-hash, cukup ringkasan angka+nama guru).

### 3. Backend — endpoint

`POST /users/generate-akun-guru` (atau nama serupa, konsisten pola RESTful bulk existing) — `@Roles(UserRole.super_admin)` (operasi sensitif, buat banyak akun sekaligus — BATASI ke role paling tinggi, KONSISTEN prinsip proyek endpoint mutasi sensitif).

### 4. Frontend — tombol + hasil ringkasan

- Halaman `(admin)/guru/guru-view.tsx` ATAU `(admin)/akun/akun-view.tsx` (PUTUSKAN SAAT IMPLEMENTASI mana yang lebih pas secara UX — REKOMENDASI: `akun-view.tsx` karena ini operasi terkait akun, bukan biodata guru) — tombol baru "Generate Akun Guru" (misal di toolbar dekat tombol Import/Tambah).
- Klik tombol → konfirmasi dulu (Dialog, KONSISTEN pola konfirmasi aksi sensitif proyek) — sebutkan **berapa banyak guru** yang akan diproses SEBELUM eksekusi (fetch count dulu, atau tampilkan count di dalam dialog konfirmasi setelah user klik, PUTUSKAN pola persisnya saat implementasi) — supaya admin tidak "buta" klik generate untuk jumlah tidak terduga.
- Setelah selesai — tampilkan **ringkasan hasil** jelas: jumlah berhasil, daftar guru dengan NIY belum valid (pesan actionable persis seperti spec), daftar guru dengan username bentrok — KONSISTEN pola tampilan hasil `ImportDialog` yang SUDAH ADA (REUSE komponen/pola render hasil import kalau bisa, JANGAN desain UI baru dari nol untuk kebutuhan yang mirip).
- **TIDAK PERLU tampilkan password di layar** setelah generate (password SAMA untuk semua = "12345678", sudah diketahui admin dari task ini, TIDAK PERLU echo balik di response/UI — mengurangi permukaan kebocoran meski low-risk).

## Edge Cases

- **Tidak ada Teacher yang perlu diproses** (semua guru sudah punya akun, atau semua NIY tidak valid) — tampilkan pesan jelas "Tidak ada guru yang perlu dibuatkan akun baru", BUKAN tombol yang diam-diam tidak melakukan apa-apa tanpa feedback.
- **Guru dengan NIY valid TAPI usernya SUDAH ADA dengan role LAIN** (misal sudah jadi `guru_piket` dengan username sama NIY-nya sendiri, dari akun lama) — INI BUKAN kasus "belum punya akun guru" secara ketat kalau query poin 2.1 benar (filter `role: "guru"` spesifik) — TAPI create User baru dengan `username` yang SAMA (NIY) akan kena unique constraint `username` (constraint GLOBAL, bukan per-role) — masuk kategori `usernameBentrok`, pesan HARUS jelas menyebutkan situasi ini spesifik ("NIY ini sudah dipakai sebagai username akun lain dengan role berbeda") — BEDA dari bentrok generik biasa, supaya admin paham kenapa (constraint unique `username` global, meski relasi Teacher↔User boleh banyak, `username` sendiri tetap harus unik system-wide).
- **Password default "12345678" terlalu lemah secara umum** — DI LUAR SCOPE task ini untuk diperkuat (user secara eksplisit minta password ini persis) — TIDAK PERLU validasi kekuatan password khusus untuk kasus generate ini, KONSISTEN keputusan user.

## Files
- **Modifikasi:** `apps/api/src/users/users.service.ts` (method `generateAkunGuruMassal()`), `apps/api/src/users/users.controller.ts` (endpoint baru), `apps/web/src/app/(admin)/akun/akun-view.tsx` (tombol+dialog+ringkasan hasil).
- **Jangan sentuh:** `ImportService.importUsers()` (pola CSV manual TETAP ADA terpisah, TIDAK digabung/diganti fitur ini — 2 cara berbeda untuk kebutuhan berbeda), `UsersService.create()` (create manual satu-per-satu TIDAK diubah).

## Eksekusi (2026-08-21)

Type baru `GenerateAkunGuruResult`/`GenerateAkunGuruSkip` ditambah ke `packages/types`
(bukan reuse `ImportReport` — shape beda, kategori `niyBelumValid`/`usernameBentrok`
eksplisit, bukan generic `errors[]` per-baris). `UsersService.generateAkunGuruMassal()` —
query `Teacher.findMany({ where: { users: { none: { role: "guru" } } } })` (relasi negatif
Prisma, dikonfirmasi bekerja benar), loop per-Teacher: validasi `isNiyValidForGenerate()`
(regex `/^\d+$/`, helper module-level di `users.service.ts` — TIDAK ditaruh di
`TeachersService` terpisah karena hanya dipakai flow ini, menghindari coupling modul
tambahan yang tidak perlu), lalu `bcrypt.hash("12345678", SALT_ROUNDS)` (SAMA
`SALT_ROUNDS=10` existing) + `mustChangePassword` dari `forcePasswordChangeConfig.
shouldForceFor(UserRole.guru)` (REUSE persis, bukan hardcode). P2002 ditangkap per-guru
(partial success, `continue` ke guru berikutnya) — error LAIN dilempar ulang (tidak
ditangkap diam-diam). Log aktivitas 1x per operasi via `ActivityLogService.record()`
manual (`ActivityLogModule` ditambah ke `imports` `UsersModule`, sebelumnya belum
terhubung). Endpoint `POST /users/generate-akun-guru`, `@Roles(super_admin)`.

Frontend: komponen baru `generate-akun-guru-dialog.tsx` (BUKAN reuse literal `ImportDialog`
— shape data beda, tidak ada file upload — tapi pola visual StatCard tone success/danger +
tabel detail DIREPLIKASI). Count "berapa guru akan diproses" dihitung CLIENT-SIDE dari
`teachers`+`users` yang SUDAH di-load `AkunView` (tidak perlu endpoint count terpisah) —
`teacherIdsWithGuruAccount` Set dari `users.filter(role==="guru")`, lalu filter `teachers`
yang belum ada di situ. Dialog 2-tahap: konfirmasi (tampilkan count) → generate → ringkasan
hasil. Password TIDAK pernah ditampilkan ulang di UI (sesuai keputusan user).

9 test baru: guru valid berhasil, query filter relasi negatif benar, NIY dummy di-skip+
pesan benar, username bentrok (P2002 asli via `Prisma.PrismaClientKnownRequestError`,
BUKAN mock Error+code manual yang gagal `instanceof` check) di-skip+lanjut proses guru
lain, error BUKAN P2002 dilempar ulang, toggle mustChangePassword KEDUA kondisi, tidak ada
guru diproses (0/0, bukan error), log 1x tanpa password.

**Verifikasi live** (dev DB, 3 teacher test dibuat sementara lalu dibersihkan): skenario
guru valid → akun dibuat, LOGIN BERHASIL dengan `username=NIY, password=12345678`
(dikonfirmasi dapat access token asli). Skenario NIY dummy `"TANPA_NIY_TEST"` → di-skip
dengan pesan benar. Skenario username bentrok (Teacher baru dengan NIY yang SUDAH dipakai
sebagai username akun `guru_piket` lain) → di-skip dengan pesan spesifik menyebut
kemungkinan role berbeda, PERSIS edge case yang dijelaskan di spec. `activity_log`
terverifikasi 1 baris per operasi, `snapshot_after` murni ringkasan angka+nama tanpa
password. **Catatan cleanup**: 1 teacher+user test tersisa di dev DB dalam status
`nonaktif` (bukan terhapus) — user itu sempat login (jadi actor di `activity_log`),
constraint `ON DELETE RESTRICT` mencegah hard-delete (konsisten aturan insert-only
`activity_log`), dinonaktifkan sebagai gantinya, dampak dev-only minimal.

## Acceptance Criteria
- [x] Endpoint generate — HANYA proses guru dengan NIY numerik murni DAN belum punya akun role `guru`.
- [x] Akun ter-generate — `username = NIY`, password `"12345678"` (hash bcrypt benar), `role: "guru"`, `teacherId` terhubung benar. Diverifikasi login live dengan kredensial ini.
- [x] `mustChangePassword` — SESUAI toggle `ForcePasswordChangeConfig.forceGuru` (dites KEDUA kondisi: toggle on dan off).
- [x] NIY dummy (contoh `"TANPA_NIY_19"`) — DI-SKIP, muncul di ringkasan dengan pesan actionable minta admin perbaiki NIY dulu.
- [x] Username bentrok — DI-SKIP (partial success, guru lain TETAP diproses), pesan jelas per-kasus (diverifikasi live, teks menyebut kemungkinan role berbeda).
- [x] Tidak ada Teacher yang perlu diproses — pesan jelas di dialog frontend, bukan diam-diam tanpa feedback.
- [x] Log aktivitas — 1 baris per operasi bulk (bukan per-guru), TIDAK mengandung password (diverifikasi query production/dev langsung).
- [x] Endpoint dibatasi `super_admin`.
- [x] Build + type-check hijau, jest baru (9 test): guru valid berhasil, NIY dummy di-skip+pesan benar, username bentrok di-skip+lanjut proses lain, toggle mustChangePassword kedua kondisi, tidak ada guru yang perlu diproses, PLUS 3 test tambahan (query filter benar, error non-P2002 dilempar ulang, log tanpa password). Full suite 42/610 pass.

## Validasi Claudian
- [x] Konfirmasi partial-success (bukan all-or-nothing) — 1 guru gagal TIDAK menggagalkan generate guru lain dalam batch yang sama (test + verifikasi live keduanya konfirmasi).
- [x] Konfirmasi `mustChangePassword` REUSE `ForcePasswordChangeConfigService.shouldForceFor()` yang SAMA dipakai `UsersService.create()`, BUKAN hardcode true seperti `importUsers()`.
- [x] Konfirmasi log aktivitas TIDAK menyertakan password (meski sudah di-hash) di `snapshotAfter` — diverifikasi test DAN query DB langsung.
- [x] Konfirmasi endpoint dibatasi `super_admin` (operasi bulk sensitif, buat banyak kredensial sekaligus).
