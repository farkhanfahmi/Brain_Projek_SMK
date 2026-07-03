# T009 — Card Import: Mode B (Tap-to-Assign)

## Depends on
T007 (CardService harus sudah ada)

## Objective
Buat UI untuk proses cepat pemetaan kartu via tap langsung di PC admin — untuk kartu yang UID-nya belum diketahui dan tidak bisa di-import via CSV.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `cards`
- **Role akses:** `super_admin` + `card_admin`
- **ADR:** ADR-004 (HID keyboard emulation — reader di PC admin bekerja sama seperti reader kiosk)
- **Ref:** `Projek/AbsenSI/06-Features/import-data-master.md` — bagian "Mode B"

## Spec Detail

### API `GET /cards/unassigned-persons`:
- Return daftar siswa + guru yang belum punya kartu active
- Format:
```json
[
  { "id": "...", "nama": "Budi Santoso", "identifier": "NISN: 1234567890", "type": "student" },
  { "id": "...", "nama": "Pak Rudi", "identifier": "NIP: 0987654321", "type": "teacher" }
]
```
- Support `search` query param (cari nama)

### API `POST /cards/tap-assign`:
- Body: `{ uid: string, person_id: string, person_type: 'student' | 'teacher' }`
- Reuse `CardService.createCard()`
- Response: kartu yang baru dibuat + nama pemilik

### UI Halaman `/admin/kartu/tap-assign`:

**Desain: alur berurutan cepat (seperti antrian)**

```
┌─────────────────────────────────────┐
│  Tap-to-Assign Kartu RFID           │
│                                     │
│  🔍 [Cari nama siswa/guru...]       │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ Budi Santoso                │   │  ← dipilih (highlight)
│  │ NISN: 1234567890 | Kelas XI │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ Citra Dewi                  │   │
│  │ NISN: 0987654321 | Kelas X  │   │
│  └─────────────────────────────┘   │
│                                     │
│  ──────────────────────────────     │
│  Status: Menunggu tap kartu...      │
│  [Input tersembunyi auto-focus]     │
│                                     │
│  ✅ Kartu A1B2C3D4 →               │
│     Budi Santoso (berhasil)         │
└─────────────────────────────────────┘
```

**Flow:**
1. Admin ketik nama di search → pilih orang dari daftar (hanya tampil yang belum punya kartu)
2. Saat orang dipilih → input tersembunyi auto-focus, status berubah "Menunggu tap kartu..."
3. Admin minta orang tersebut tap kartu ke reader yang tersambung di PC admin
4. UID masuk via keystroke → Enter → `POST /cards/tap-assign` → feedback sukses/gagal
5. Sukses → nama orang yang baru selesai hilang dari daftar (sudah punya kartu), input bersih, siap untuk orang berikutnya
6. Gagal → pesan error, UID dikosongkan, menunggu tap ulang

**Penting:** tidak ada tombol Submit manual — proses terjadi otomatis saat Enter dari reader.

## JANGAN
- ❌ JANGAN buat form dengan field UID yang bisa diketik manual — field UID harus tersembunyi dan hanya bisa terisi dari keystroke reader (kalau bisa diketik, ada risiko salah input)
- ❌ JANGAN reload halaman setelah setiap assign — ini harus jadi pengalaman cepat berurutan tanpa reload
- ❌ JANGAN tampilkan orang yang sudah punya kartu aktif di daftar

## Files
- **Buat:** `apps/api/src/card/` — tambah endpoint baru di controller yang sudah ada
- **Buat:** `apps/web/app/(admin)/kartu/tap-assign/page.tsx`

## Acceptance Criteria
- [ ] Pilih siswa → tap kartu → kartu terpasang → siswa hilang dari daftar
- [ ] Tap kartu dengan UID yang sudah ada di DB → error muncul, bisa tap ulang
- [ ] Daftar di-refresh otomatis setelah assign berhasil (tanpa reload halaman)
- [ ] Search berfungsi untuk filter daftar yang panjang
- [ ] Proses 10 assign berturut-turut tanpa perlu reload atau navigasi ulang
