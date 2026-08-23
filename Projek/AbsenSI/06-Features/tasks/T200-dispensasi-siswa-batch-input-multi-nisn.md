# T200 — API+Web: Dispensasi Siswa — Input Batch Banyak NISN Sekaligus (Pola PKL)

## Depends on
**WAJIB setelah T192** (Dispensasi Siswa, SUDAH SELESAI — task ini mengubah/memperluas form yang sudah dibuat, bukan buat baru dari nol).

## Objective
Form Dispensasi Siswa (T192, SAAT INI hanya bisa 1 siswa per submit via search-select tunggal) — GANTI jadi **input batch banyak siswa sekaligus** (paste/ketik banyak NISN, validasi per-baris, submit semua sekaligus) — MENIRU PERSIS pola form "Mulai PKL" yang SUDAH ADA dan terbukti bagus untuk kasus serupa.

## Context — Keputusan User (2026-08-16)

Riset mengonfirmasi form Dispensasi (T192) SAAT INI: `dispen-form.tsx` search-select 1 siswa (`formData.append("studentId", ...)`), backend `CreateDispenDto.studentId: number` (bukan array). User eksplisit: "dispensasi siswa bisa jadi banyak siswa... buat seperti input siswa PKL yang bisa hanya copas banyak NISN".

**Referensi pola PKL** (`(admin)/siswa/pkl/pkl-view.tsx`) yang HARUS DITIRU PERSIS:
1. `<textarea>` — admin paste/ketik banyak NISN (pisah baris baru ATAU koma), `parseNisns()` split+dedupe via `Set`.
2. Debounce 500ms → `POST /students/pkl/validate-nisn-batch` body `{nisns: string[]}` → tampilkan HASIL PER-NISN (valid/tidak ditemukan/kondisi khusus) SEBELUM submit final — supaya admin tahu NISN mana yang bermasalah SEBELUM klik submit.
3. Submit final → `POST /students/pkl/mulai` body `{studentIds: number[], ...field lain}`.

## Spec Detail

### 1. Backend — endpoint validasi batch + create batch

- `PermitsController`/`PermitsService` (domain dispen, T192) — endpoint BARU `POST /permits/dispen/validate-nisn-batch` body `{nisns: string[]}` — REPLIKASI PERSIS pola `validate-nisn-batch` PKL (VERIFIKASI signature/response shape method itu, REUSE strukturnya) — return status per-NISN: ditemukan+valid / tidak ditemukan / SUDAH punya izin/dispen di tanggal yang sama (konflik).
- `CreateDispenDto` — GANTI `studentId: number` jadi `studentIds: number[]` (array, minimal 1 elemen — `@ArrayMinSize(1)`).
- `PermitsService.createDispen()` (nama method VERIFIKASI sesuai implementasi T192) — LOOP `studentIds`, create 1 `Permit` PER siswa (bukan 1 Permit untuk semua siswa sekaligus — setiap siswa punya baris `Permit` sendiri, KONSISTEN struktur data existing) — **best-effort** (KONSISTEN pola T179/T180: 1 siswa gagal — misal sudah ada izin konflik di tanggal sama — TIDAK menggagalkan siswa lain, return report `{successCount, failedCount, errors: [{studentId, reason}]}`).
- Upload bukti (`buktiFilePath`) — SATU FILE untuk SEMUA siswa dalam batch itu (surat tugas lomba biasanya 1 dokumen untuk semua peserta, BUKAN per-siswa) — VERIFIKASI asumsi ini masuk akal, KLARIFIKASI user kalau ragu apakah bukti per-siswa atau 1-untuk-semua.

### 2. Frontend — form batch mengikuti pola PKL

- `dispen-form.tsx` — GANTI search-select tunggal jadi `<textarea>` paste NISN (REPLIKASI struktur `pkl-view.tsx` PERSIS — reuse komponen/pattern kalau bisa diekstrak jadi shared, JANGAN duplikasi kode kalau strukturnya benar-benar identik, PERTIMBANGKAN ekstrak jadi komponen `NisnBatchInput` reusable kalau masuk akal — TAPI TIDAK WAJIB kalau menambah kompleksitas berlebih untuk 2 pemakaian saja).
- Debounce validasi + tampilkan hasil per-NISN SEBELUM submit (REPLIKASI PERSIS UX PKL).
- Field lain (rentang tanggal, alasan, upload bukti) — TETAP SAMA seperti form T192 SAAT INI, HANYA cara PILIH SISWA yang berubah dari single ke batch.
- Setelah submit — tampilkan hasil ringkas (berhasil/gagal per siswa, KONSISTEN pola T179/T180).

## Edge Cases
- NISN yang di-paste TIDAK DITEMUKAN di database — tampil sebagai error per-baris SEBELUM submit (KONSISTEN validasi PKL), TIDAK menggagalkan submit siswa lain yang valid.
- NISN duplikat dalam paste yang sama — dedupe otomatis (KONSISTEN pola PKL `Set`).
- Siswa yang SUDAH punya `Permit` (izin/sakit/dispen lain) di tanggal yang SAMA dengan dispen batch ini — DITOLAK untuk siswa itu SAJA (best-effort), siswa lain dalam batch yang sama TETAP berhasil.

## Files
- **Modifikasi:** `apps/api/src/permits/permits.controller.ts`+`permits.service.ts` (endpoint validate-batch baru, `createDispen()` terima array), `CreateDispenDto` (studentIds array), `apps/web/.../admin-jurnal/dispen/components/dispen-form.tsx` (textarea batch + validasi live + hasil ringkas).
- **Jangan sentuh:** form/endpoint PKL existing (REFERENSI pola saja, TIDAK diubah), board piket yang menampilkan dispen (T192, tampilan tidak berubah — tetap 1 Permit = 1 baris di board, terlepas dari cara input-nya batch atau tunggal).

## Acceptance Criteria
- [x] Form Dispensasi Siswa — textarea paste banyak NISN, validasi live per-baris SEBELUM submit.
- [x] Submit batch berhasil membuat `Permit` dispen untuk SEMUA siswa valid sekaligus (best-effort, 1 gagal tidak menggagalkan lain).
- [x] NISN tidak ditemukan / duplikat / konflik tanggal — ditangani sesuai Edge Cases, pesan jelas per kasus.
- [x] Hasil ringkas (berhasil/gagal per siswa) ditampilkan setelah submit.
- [x] Build + type-check hijau, jest baru untuk skenario batch (campuran valid+invalid+konflik).

## Validasi Claudian
- [x] Konfirmasi pola validate-batch+submit-batch MEREPLIKASI PERSIS struktur PKL — lihat rincian di bawah.
- [x] **Klarifikasi ke user** soal bukti file — dikonfirmasi 1 file untuk SEMUA siswa dalam batch (bukan per-siswa), sesuai rekomendasi awal spec.

## Status Eksekusi (2026-08-16)

**Selesai.**

### Kesamaan/perbedaan vs pola PKL (dikonfirmasi eksplisit)

| Aspek | PKL | Dispensasi (T200) |
|---|---|---|
| Textarea+parseNisns (split newline/koma, dedupe Set) | Ya | SAMA PERSIS (kode direplikasi identik) |
| Debounce 500ms → validate-nisn-batch | Ya | SAMA PERSIS |
| Endpoint validate | `POST /students/pkl/validate-nisn-batch` | `POST /permits/dispen/validate-nisn-batch` (modul BEDA, endpoint baru) |
| Kriteria "valid" | NISN ditemukan + kelas tingkat XII + belum PKL | NISN ditemukan + `status: aktif` (semua tingkat, TIDAK ada syarat kelas — beda semantik disengaja) |
| Submit body | JSON polos (`{studentIds, tempatPkl}`) | **FormData** (ada file upload) — `studentIds` dikirim sebagai 1 field JSON.stringify, di-parse via `@Transform` di DTO (PKL tidak butuh trik ini karena tidak ada file) |
| Response submit | `{started, skippedSudahPkl}` | `{successCount, failedCount, errors[]}` — pola `confirmKembaliBulk` (T179), bukan pola PKL |

### Backend

- `CreateDispenDto.studentId: number` → `studentIds: number[]` (`@ArrayMinSize(1)`), dengan `@Transform` custom untuk parse JSON string dari FormData (try/catch, gagal parse dibiarkan lolos ke `@IsArray` supaya pesan error tetap validasi standar, bukan crash `JSON.parse`).
- `PermitsService.validateNisnBatchDispen()` — method baru, REPLIKASI `validateNisnBatchPkl` tapi kriteria valid beda (aktif, bukan XII-only).
- `PermitsService.createDispen()` — signature tambah param `ipAddress`, return type berubah total dari `Permit` tunggal jadi `{successCount, failedCount, errors}`. Validasi LEVEL-BATCH (tanggalSelesai < tanggal) tetap throw langsung (di luar loop, sebelum proses siswa manapun) — tapi validasi PER-SISWA (siswa tidak ditemukan, sudah tap, permit overlap) masuk `try/catch` di dalam loop, best-effort.
- `@LogActivity` decorator DILEPAS dari `POST /permits/dispen` (tidak cocok untuk N target sekaligus) — log manual `activityLog.record()` di dalam loop, HANYA untuk siswa yang berhasil (pola sama `confirmKembaliBulk` T179).
- `POST /permits/dispen/validate-nisn-batch` — endpoint baru, read-only, tidak ada `@LogActivity`.
- 15 unit test baru/diperbarui (validate-batch: valid/tidak ditemukan/tidak aktif/dedupe; createDispen: best-effort per skenario lama + kasus BARU batch campuran valid+gagal) — 486/486 pass di seluruh suite.

### Frontend

- `dispen-form.tsx` — REWRITE TOTAL dari search-select 1 siswa jadi textarea batch NISN, struktur REPLIKASI `pkl-view.tsx` (textarea, debounce, badge hasil per-NISN, tombol submit menampilkan jumlah valid). `studentIds` dikirim via `FormData.append("studentIds", JSON.stringify(...))` karena body tetap multipart (ada file).
- `dispen-view.tsx` — `onCreated` diubah dari append 1 `Permit` jadi refetch penuh `GET /permits/dispen` (submit sekarang bisa hasilkan banyak Permit sekaligus, tidak ada 1 objek tunggal untuk di-append).
- `dispen/page.tsx` — prop `students: Student[]` DIHAPUS (tidak perlu lagi, validasi sekarang lewat NISN batch server-side, KONSISTEN pola PKL yang juga tidak fetch daftar siswa penuh di page.tsx).
- `DispenTable` (riwayat) TIDAK disentuh — tetap terima `Permit[]` flat, 1 Permit = 1 baris terlepas dari cara input batch/tunggal (sesuai instruksi "Jangan sentuh" board yang menampilkan dispen).

### Verifikasi

- `tsc --noEmit` bersih di `apps/api` dan `apps/web`.
- `jest apps/api`: 486/486 pass, 29/29 suite.
- Live-verify browser: belum dilakukan (konsisten pola T186-T196, verifikasi manual diserahkan ke user).
