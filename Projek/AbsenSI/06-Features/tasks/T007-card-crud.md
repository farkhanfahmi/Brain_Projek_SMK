# T007 — Card Module: CRUD Kartu RFID

## Depends on
T005 (students & teachers harus ada sebagai target mapping kartu)

## Objective
Buat API dan UI admin untuk registrasi, nonaktifkan, dan ganti kartu RFID. Termasuk riwayat kartu per orang.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `cards`
- **Role akses:** `super_admin` + `card_admin`
- **ADR:** ADR-010 (dual-FK nullable — `student_id` XOR `teacher_id`)
- **Ref:** `Projek/AbsenSI/06-Features/manajemen-kartu.md`

## Spec Detail

### Aturan bisnis kritis:
1. 1 UID hanya boleh aktif ke 1 orang (`status: active`) pada satu waktu
2. UID yang pernah dipakai lalu dinonaktifkan **tidak boleh** didaftar ulang ke orang lain
3. 1 orang hanya boleh punya 1 kartu `active` sekaligus (tapi bisa punya banyak kartu `inactive` dari masa lalu)
4. Tepat 1 dari `student_id` / `teacher_id` yang terisi (tidak boleh keduanya null atau keduanya isi)

### API Endpoints:

**GET `/cards`**
- Query params: `status?` (active/inactive), `person_type?` (student/teacher), `search?` (uid/nama), pagination
- Include relasi: nama siswa/guru yang terhubung

**POST `/cards`** (registrasi kartu baru)
- Body: `{ uid: string, student_id?: string, teacher_id?: string }`
- Validasi:
  - `uid` belum pernah ada di DB sama sekali (bukan hanya tidak aktif — kalau UID pernah ada, tolak)
  - Tepat 1 dari student_id/teacher_id terisi
  - Orang yang dituju belum punya kartu `active`
- Return: kartu yang baru dibuat

**PATCH `/cards/:id/revoke`** (nonaktifkan kartu)
- Set `status: inactive`, isi `revoked_at`
- Tidak hapus data, riwayat tap tetap tersimpan

**POST `/cards/:id/replace`** (ganti kartu — kartu lama rusak/hilang)
- Body: `{ new_uid: string }`
- Atomik: dalam 1 transaksi — nonaktifkan kartu lama, buat kartu baru dengan UID baru untuk orang yang sama
- `new_uid` harus belum pernah ada di DB

### Admin UI:

**Halaman `/admin/kartu`:**
- Tabel: UID | Pemilik | Tipe (Siswa/Guru) | Status | Tanggal Dibuat | Aksi
- Filter: status, person_type, search
- Tombol **[Nonaktifkan]** per baris (muncul kalau status active)
- Tombol **[Ganti Kartu]** per baris (muncul kalau status active) → modal input UID baru

**Form Registrasi Kartu Baru:**
- Field: UID (input text biasa atau scan dari reader PC admin)
- Radio: Siswa / Guru
- Autocomplete: cari nama siswa atau guru (hanya yang belum punya kartu aktif)
- Tombol Submit

### Input UID dari reader di PC admin:
UID reader di PC admin juga bekerja sebagai HID keyboard — saat fokus di field UID dan kartu di-tap, UID otomatis terisi. Implementasi: field UID dengan `onKeyDown` yang capture Enter setelah kartu di-tap.

## JANGAN
- ❌ JANGAN izinkan reuse UID yang pernah ada — error: "UID ini sudah pernah terdaftar sebelumnya dan tidak dapat didaftarkan ulang"
- ❌ JANGAN buat endpoint DELETE kartu — hanya revoke (nonaktifkan)
- ❌ JANGAN buat kartu bisa dimiliki oleh keduanya (student_id DAN teacher_id terisi sekaligus)
- ❌ JANGAN hilangkan riwayat kartu lama saat replace — kartu lama tetap ada di DB dengan status inactive
- ❌ JANGAN buat UI untuk assign kartu dari halaman detail siswa/guru — semua dari halaman `/admin/kartu`

## Files
- **Buat:** `apps/api/src/card/card.module.ts`
- **Buat:** `apps/api/src/card/card.service.ts`
- **Buat:** `apps/api/src/card/card.controller.ts`
- **Buat:** `apps/web/app/(admin)/kartu/page.tsx`

## Acceptance Criteria
- [ ] Registrasi kartu dengan UID baru + student_id valid → berhasil
- [ ] Registrasi kartu dengan UID yang sudah pernah ada (meski inactive) → error
- [ ] Registrasi kartu untuk siswa yang sudah punya kartu active → error
- [ ] Revoke kartu → status berubah inactive, `revoked_at` terisi
- [ ] Replace kartu → kartu lama inactive, kartu baru active, keduanya untuk orang yang sama
- [ ] `GET /cards` dengan filter `status=active` hanya return kartu aktif
- [ ] Login sebagai `card_admin` bisa akses semua endpoint card

## Handoff ke T008 & T009
T008 (bulk CSV import kartu) dan T009 (tap-to-assign) menggunakan logika validasi yang sama dengan `POST /cards`. Pertimbangkan membuat `CardService.createCard()` yang bisa dipakai oleh ketiga alur: CRUD manual (T007), bulk CSV (T008), dan tap-to-assign (T009).
