# T008 — Card Import: Mode A (Bulk CSV)

## Depends on
T007 (CardService.createCard() harus sudah ada dan bisa dipakai ulang)

## Objective
Buat fitur import massal pemetaan kartu RFID via CSV untuk rollout awal (UID sudah diketahui dari vendor).

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `cards`
- **Role akses:** `super_admin` + `card_admin`
- **Ref:** `Projek/AbsenSI/06-Features/import-data-master.md` — bagian "Mode A"

## Spec Detail

### CSV Format:
```
nisn_atau_nip,uid
1234567890,A1B2C3D4
0987654321,E5F6G7H8
```

Kolom:
- `nisn_atau_nip`: sistem auto-detect — coba cari di `students.nisn` dulu, kalau tidak ada cari di `teachers.nip`
- `uid`: UID kartu yang akan dipasangkan

### API `POST /import/cards`** (multipart/form-data, field: `file`):
- Parse CSV baris per baris
- Per baris:
  1. Cari siswa/guru berdasarkan `nisn_atau_nip`
  2. Kalau tidak ditemukan → baris gagal: "NISN/NIP tidak ditemukan"
  3. Kalau ditemukan tapi sudah punya kartu active → baris gagal: "Sudah punya kartu aktif"
  4. Kalau UID sudah pernah ada di DB → baris gagal: "UID sudah pernah terdaftar"
  5. Kalau valid → panggil `CardService.createCard()` (reuse dari T007)
- Partial commit — sama seperti T006
- Response format sama persis dengan T006 (konsistensi UX)

### Admin UI:
Tambahkan **tab "Pemetaan Kartu (CSV)"** ke halaman import yang sudah ada (`/admin/import`) — **jangan buat halaman baru**.

Tab ini berisi:
- Tombol download template CSV (kolom: nisn_atau_nip, uid)
- Upload area
- Hasil import (format sama dengan tab Siswa/Guru)

## JANGAN
- ❌ JANGAN buat halaman import terpisah — tambahkan tab ke `/admin/import` yang sudah ada di T006
- ❌ JANGAN tulis ulang logika validasi kartu — reuse `CardService.createCard()` dari T007
- ❌ JANGAN izinkan format `nisn,uid` dan `nip,uid` sebagai 2 kolom terpisah — cukup 1 kolom `nisn_atau_nip` yang auto-detect

## Files
- **Modifikasi:** `apps/api/src/import/import.service.ts` — tambah method `importCards()`
- **Modifikasi:** `apps/api/src/import/import.controller.ts` — tambah endpoint `POST /import/cards`
- **Modifikasi:** `apps/web/app/(admin)/import/page.tsx` — tambah tab ketiga

## Acceptance Criteria
- [ ] CSV 50 baris valid → 50 kartu terdaftar
- [ ] Baris dengan NISN tidak ada → gagal dengan pesan yang jelas
- [ ] Baris dengan UID duplikat → gagal, baris lain tetap masuk
- [ ] `card_admin` bisa akses endpoint ini (bukan hanya `super_admin`)
- [ ] Tab baru muncul di halaman import yang sudah ada

## Handoff ke T009
T009 adalah Mode B (tap-to-assign) — untuk kartu yang UID-nya tidak diketahui dari CSV. Keduanya adalah alur berbeda di UI yang sama.
