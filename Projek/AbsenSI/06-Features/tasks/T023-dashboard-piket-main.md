# T023 — Dashboard Piket: Halaman Utama Realtime

## Depends on
T017 (Socket.IO harus broadcast per kampus), T022 (permits API harus ada)

## Objective
Buat halaman utama Dashboard Piket yang menampilkan daftar siswa per kampus secara realtime beserta tombol quick action [Izin] dan [Sakit].

## Context
- **App:** `apps/web`
- **Route:** `/piket`
- **Role:** `guru_piket`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-piket.md` — Fungsi 1 & 2

## Spec Detail

### API endpoint baru:
`GET /attendance/piket-today?kampus_id=X`
- Akses: `guru_piket` (kampus_id dari JWT, bukan dari query param)
- Return: semua siswa di kampus tersebut + status kehadiran hari ini
```json
[
  {
    "student_id": "xxx",
    "nama": "Budi Santoso",
    "kelas": "XI-RPL-1",
    "status_hari_ini": "hadir",      // dari attendance_records
    "waktu_masuk": "07:15:00",
    "waktu_pulang": null,
    "locked": false,
    "permit_hari_ini": null          // null atau { jenis, alasan_kategori }
  }
]
```

### Layout `/piket`:
- Buat `apps/web/app/piket/layout.tsx` — sidebar minimal: Home, Izin Keluar, Logout
- Header: "Dashboard Piket — Kampus X | [tanggal hari ini]"

### Tabel utama:
| Nama | Kelas | Status | Masuk | Pulang | Aksi |
- Badge status: Hadir (hijau), Terlambat (kuning), Izin (biru), Sakit (ungu), Belum (abu), Terkunci (merah)
- Kolom Aksi: tombol **[Izin]** dan **[Sakit]** — muncul hanya jika status belum ada permit hari ini
- Kalau sudah ada permit → tampilkan badge status saja, tidak ada tombol

### Tombol [Izin] / [Sakit]:
Klik → modal kecil:
```
Siswa: Budi Santoso
Alasan: [Sakit ✓] (pre-filled dari tombol yang diklik)
Keterangan: [input text, wajib diisi]
[Batal] [Simpan]
```
Submit → `POST /permits` → tutup modal → row siswa update status-nya

### Realtime update:
- Subscribe Socket.IO `attendance:kampus:{kampus_id}` (dari JWT)
- Saat ada event masuk: cari baris siswa yang sesuai, update status + waktu tanpa reload halaman
- Baru tap masuk: row highlight sebentar (transisi warna) supaya piket tahu ada update

### Scope kampus:
- `kampus_id` diambil dari JWT (`req.user.kampus_id`), bukan dari query param
- UI tidak perlu dropdown pilih kampus — piket sudah terikat ke 1 kampus

## JANGAN
- ❌ JANGAN tampilkan siswa dari kampus lain — scope dari JWT
- ❌ JANGAN buat tombol izin/sakit untuk siswa yang sudah punya permit hari ini
- ❌ JANGAN buat tombol lock/unlock di halaman ini — itu di section terpisah T026
- ❌ JANGAN reload seluruh halaman saat ada update Socket.IO — update baris yang relevan saja

## Files
- **Buat:** `apps/web/app/piket/layout.tsx`
- **Buat:** `apps/web/app/piket/page.tsx`
- **Buat:** `apps/web/app/piket/components/StudentRow.tsx`
- **Buat:** `apps/web/app/piket/components/IzinModal.tsx`
- **Modifikasi:** `apps/api/src/attendance/attendance.controller.ts` — tambah `GET /attendance/piket-today`

## Acceptance Criteria
- [ ] Login sebagai `guru_piket` kampus 1 → hanya melihat siswa kampus 1
- [ ] Tap kartu siswa di kiosk → baris siswa di dashboard piket update status dalam < 2 detik
- [ ] Klik [Sakit] → modal muncul dengan "Sakit" pre-selected → isi keterangan → simpan → badge row berubah jadi "Sakit"
- [ ] Siswa yang sudah punya permit hari ini tidak tampilkan tombol [Izin]/[Sakit]
- [ ] Siswa terkunci (`locked_at IS NOT NULL`) tampilkan badge merah "Terkunci"
