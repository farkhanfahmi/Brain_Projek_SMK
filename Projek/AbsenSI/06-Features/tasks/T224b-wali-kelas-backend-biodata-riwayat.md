# T224b — API: Wali Kelas — Endpoint Biodata + Riwayat Catatan Siswa (Scoped)

## Depends on
Tidak ada dependency teknis ke T224a/T224c (independen satu sama lain). **Bagian 2 dari 4** rangkaian pemecahan T224 asli. WAJIB selesai sebelum T224d (frontend).

## Konteks — Riwayat Catatan SUDAH ADA untuk Admin (dikonfirmasi via riset 2026-08-19)

Fitur "Riwayat Catatan" (gabungan terlambat/izin/sakit/dispen/alfa/pkl/terkunci/dibuka untuk 1 siswa) **SUDAH ADA LENGKAP**:
- Method: `AttendanceReportService.riwayatCatatan(studentId)` — `apps/api/src/attendance/attendance-report.service.ts:496-619`.
- Endpoint existing: `GET /attendance/students/:id/riwayat-catatan` (`attendance.controller.ts:148-157`) — `@Roles(super_admin, card_admin, guru_piket)`, TANPA validasi kepemilikan (role-role itu memang lintas-kampus by design, T076).
- Type `RiwayatCatatanEntry` (union 5 varian by `jenis`) — `attendance-report.service.ts:8-31` (backend) dan `core-types.ts:400-419` (frontend, duplikat manual).

**JANGAN tulis ulang logic riwayat** — task ini REUSE method `riwayatCatatan()` yang sudah ada, HANYA tambah lapisan akses baru untuk wali kelas dengan validasi kepemilikan.

## Keputusan Dikonfirmasi User (2026-08-19)

1. Klik nama siswa (di Daftar Siswa, T224a) → tampilkan biodata + Riwayat Catatan.
2. **Riwayat kartu (`Card[]`) TIDAK ditampilkan** di halaman wali kelas — harus dikecualikan dari response biodata secara eksplisit di level query backend, bukan cuma disembunyikan di render frontend.

## Spec Detail

### 1. Endpoint Biodata Siswa (scoped)

- Endpoint baru: `GET /journal/kelas-wali-siswa/:id` (KONSISTEN pola `journal-kelas-wali.controller.ts` — base path `/journal`, guard guru).
- Query `Student.findUnique({ where: { id } })` — **VALIDASI kepemilikan WAJIB**: siswa yang diminta harus punya `kelasId === user.kelasIdWali` — kalau tidak (siswa dari kelas lain, atau siswa nonaktif yang `kelasId`-nya sudah null tapi `kelasTerakhirNama` cocok kelas wali — VERIFIKASI SAAT IMPLEMENTASI perlakuan siswa nonaktif: gunakan `kelasTerakhirNama` ATAU histori lain untuk validasi kepemilikan siswa yang sudah nonaktif, JANGAN tolak begitu saja siswa nonaktif yang MEMANG dulunya di kelas ini) → 403 dengan pesan jelas ("Anda hanya bisa melihat data siswa di kelas yang Anda wali-i").
- **Response field**: SEMUA biodata `Student` YANG SUDAH ADA (T028: `tempatLahir`, `tanggalLahir`, `jenisKelamin`, `agama`, `alamat`, `rtRw`, `namaAyah`, `namaIbu`, `foto`, `noHpSiswa`, `noHpAyah`, `noHpIbu`) — via `select` Prisma eksplisit (BUKAN `include` generik yang bisa tidak sengaja membawa relasi `cards`).
- **EXCLUDE eksplisit `Card[]`/relasi kartu** — pastikan `select` Prisma TIDAK menyertakan field `cards` sama sekali (defense in depth: data tidak boleh ada di response meski FE tidak merendernya).

### 2. Endpoint Riwayat Catatan (scoped, REUSE logic existing)

- Endpoint baru: `GET /journal/kelas-wali-siswa/:id/riwayat-catatan` (endpoint TERPISAH dari yang admin pakai — KONSISTEN pola `journal-kelas-wali.controller.ts` yang selalu derive scope dari JWT, BUKAN menambah role `guru` langsung ke `@Roles()` endpoint admin `attendance.controller.ts:148-157` — itu akan butuh if-check kepemilikan manual per-request yang riskan lupa, sementara endpoint terpisah di controller yang SUDAH selalu scoped otomatis lebih aman by design).
- **VALIDASI kepemilikan SAMA seperti poin 1** — 403 kalau siswa bukan di kelas wali.
- **Setelah validasi lolos**, panggil LANGSUNG `AttendanceReportService.riwayatCatatan(studentId)` (method existing, TIDAK diubah/diduplikasi) — inject `AttendanceReportService` ke controller/service wali kelas.
- Response shape SAMA PERSIS `RiwayatCatatanEntry[]` (existing), supaya FE (T224d) bisa reuse komponen render tabel yang sama dengan yang admin pakai.

## Edge Cases

- **Siswa nonaktif diklik** — biodata + Riwayat Catatan TETAP bisa dibuka (histori tetap relevan meski siswa sudah keluar) — validasi kepemilikan HARUS tetap mengizinkan ini (lihat catatan `kelasTerakhirNama` di atas), JANGAN blanket-reject semua siswa nonaktif.
- **Siswa yang TIDAK PERNAH ada di kelas wali** (nonaktif dari kelas lain, atau aktif di kelas lain) — 403, TIDAK ADA pengecualian.

## Files
- **Modifikasi:** `apps/api/src/journal/journal-kelas-wali.controller.ts` (2 endpoint baru), service pendamping (inject `AttendanceReportService`, method validasi kepemilikan baru — REKOMENDASI: 1 helper `ensureSiswaMilikKelasWali(studentId, user)` dipakai di 2 endpoint sekaligus, hindari duplikasi validasi).
- **Jangan sentuh:** `AttendanceReportService.riwayatCatatan()` (method existing, TIDAK diubah sama sekali — cukup dipanggil), endpoint admin `GET /attendance/students/:id/riwayat-catatan` (TIDAK diubah, role admin tetap seperti sebelumnya).

## Acceptance Criteria
- [x] `GET /journal/kelas-wali-siswa/:id` — biodata lengkap TANPA field `cards`/riwayat kartu.
- [x] `GET /journal/kelas-wali-siswa/:id/riwayat-catatan` — response identik shape dengan endpoint admin, data sama untuk siswa yang sama.
- [x] Siswa DI LUAR kelas wali (aktif maupun nonaktif dari kelas lain) — 403 di KEDUA endpoint.
- [x] Siswa nonaktif YANG DULUNYA di kelas wali ini — TETAP bisa diakses (biodata+riwayat), tidak ditolak.
- [x] Build + type-check hijau, jest baru: siswa milik kelas (berhasil), siswa kelas lain (403), siswa nonaktif eks-kelas-ini (berhasil), response tidak mengandung field `cards`.

## Validasi Claudian
- [x] Konfirmasi `Card[]`/riwayat kartu di-EXCLUDE di level `select` Prisma (bukan cuma disembunyikan di render FE) — `select` eksplisit di `siswaDetail()` TIDAK menyebut `cards` sama sekali, diverifikasi test `expect.not.objectContaining({ cards: true })`.
- [x] Konfirmasi `AttendanceReportService.riwayatCatatan()` di-REUSE apa adanya (dipanggil, bukan diduplikasi/ditulis ulang) — `siswaRiwayatCatatan()` HANYA panggil `this.attendanceReportService.riwayatCatatan(id)` setelah validasi, method itu sendiri TIDAK disentuh sama sekali.
- [x] Konfirmasi validasi kepemilikan mengizinkan siswa nonaktif eks-kelas-ini, TIDAK blanket-reject semua status nonaktif — diverifikasi test eksplisit (`kelasId:null, kelasTerakhirNama match` → diizinkan).

## Implementasi (2026-08-20)

**Keputusan desain — validasi di controller, BUKAN di `JournalService`**: spec merekomendasikan helper `ensureSiswaMilikKelasWali()` "di service pendamping". Diimplementasikan di CONTROLLER (bukan `JournalService`) — `JournalService` sudah punya constructor dengan 7 call site test existing (`new JournalService(prisma, activityLog, academicPeriod)`), menambah dependency baru ke situ akan memaksa update SEMUA test lama tanpa manfaat tambahan. `PrismaService` (global module, tidak perlu import eksplisit) + `AttendanceReportService` (di-export `AttendanceModule`, ditambahkan ke `imports: []` `JournalModule`) di-inject LANGSUNG ke `JournalKelasWaliController` — pola yang sama sekali tidak mengubah `JournalService` yang sudah stabil.

**Backend**:
- `journal.module.ts` — tambah `AttendanceModule` ke `imports` (tidak perlu `forwardRef`, dikonfirmasi TIDAK ada circular dependency — `AttendanceModule` tidak pernah mengimpor `JournalModule` langsung/tidak langsung).
- `journal-kelas-wali.controller.ts` — constructor tambah `PrismaService`+`AttendanceReportService`. `siswaDetail()`: `select` Prisma eksplisit 18 field biodata (T028) + relasi `kelas`, SENGAJA TIDAK menyebut `cards` sama sekali. `siswaRiwayatCatatan()`: validasi lalu panggil `attendanceReportService.riwayatCatatan(id)` langsung. `ensureSiswaMilikKelasWali()` (private, baru) — 1 helper dipakai KEDUA endpoint, cek `kelasId === kelasWali` ATAU `kelasTerakhirNama === nama kelas wali` (pola OR SAMA PERSIS T224a, konsisten solusi konflik T220 yang sudah ditemukan sebelumnya).

**Verifikasi**: `nest application successfully started` dikonfirmasi via boot manual (`node dist/main.js`) — membuktikan graph DI modul baru (`JournalModule`→`AttendanceModule`) resolve tanpa circular dependency runtime (tsc/nest build tidak menangkap kelas error ini). 7 test baru (5 `siswaDetail`, 2 `siswaRiwayatCatatan`) + rewrite file spec existing (constructor 3-arg). tsc bersih, `nest build` bersih, 55/55 test journal module lulus.
