---
tags:
  - project
  - user-roles
created: 2026-06-04
updated: 2026-06-04
---

# User Roles — E-Berita Acara Ujian

## Daftar Aktor

### 1. Admin
- **Sumber data:** Tabel `users` (dengan password, login via email/password)
- **Auth:** Sanctum token via form login (frontend-admin)
- **Akses:** Hanya via `frontend-admin`

### 2. Pengawas
- **Sumber data:** Tabel `pengawas` (NIY = kode unik di barcode ID card)
- **Auth:** Scan barcode NIY → dapat Sanctum token → simpan di localStorage
- **Akses:** Hanya via `frontend` (mobile)
- **Scope data:** Dibatasi ke ruangan yang ditugaskan sesuai `jadwal_ujians`

### 3. Panitia
- **Sumber data:** Tabel `panitia` (NIY = kode unik di barcode ID card)
- **Auth:** Scan barcode NIY → **tidak dapat token** → data disimpan di localStorage session
- **Akses:** Hanya via `frontend` (mobile)
- **Scope data:** Akses ke semua ruangan dalam satu ujian

### 4. TV Display (Tanpa Auth)
- **Sumber data:** Semua endpoint `GET /dashboard/*` (public, tidak perlu auth)
- **Auth:** Tidak ada — akses publik terbatas
- **Akses:** `frontend-tv` — hanya baca + barcode listener

---

## Matriks Izin

| Fitur | Admin | Pengawas | Panitia | TV |
|-------|-------|----------|---------|-----|
| Login management | ✅ | ❌ | ❌ | ❌ |
| CRUD Ujian/Event | ✅ | ❌ | ❌ | ❌ |
| CRUD Pengawas | ✅ | ❌ | ❌ | ❌ |
| CRUD Panitia | ✅ | ❌ | ❌ | ❌ |
| CRUD Peserta | ✅ | ❌ | ❌ | ❌ |
| CRUD Jadwal Ujian | ✅ | ❌ | ❌ | ❌ |
| CRUD Ruang | ✅ | ❌ | ❌ | ❌ |
| Import data (Excel/CSV) | ✅ | ❌ | ❌ | ❌ |
| Scan QR peserta | ❌ | ✅ (ruangan sendiri) | ✅ (semua ruang) | ❌ |
| Presensi manual peserta | ❌ | ✅ | ✅ | ❌ |
| Submit Berita Acara (TTD) | ❌ | ✅ | ❌ | ❌ |
| Catat keterangan absen | ❌ | ❌ | ✅ | ❌ |
| Catat catatan pelanggaran | ❌ | ❌ | ✅ | ❌ |
| Dashboard monitor ruangan | ❌ | ❌ | ✅ | ❌ |
| Rekap kehadiran admin | ✅ | ❌ | ✅ | ❌ |
| Rekap jadwal pengawas | ❌ | ✅ | ❌ | ❌ |
| Lihat laporan berita acara | ✅ | ❌ | ❌ | ❌ |
| Monitor TV real-time | ❌ | ❌ | ❌ | ✅ |
| Barcode presensi masuk/pulang | ❌ | ❌ | ❌ | ✅ (via barcode USB) |
| Settings aplikasi | ✅ | ❌ | ❌ | ❌ |

---

## Implementasi Auth

### Pengawas Auth Flow
```
1. Scan ID card di frontend/ (kamera QR) atau frontend-tv/ (barcode reader)
2. POST /login-niy { niy: "12345" }
3. Backend: cari NIY di tabel pengawas
   → Jika found: buat Sanctum token, return { token, user, presensi }
4. Frontend: simpan token di localStorage['pengawas_token']
5. Setiap request: header Authorization: Bearer {token}
6. Auto-logout setelah idle timeout (configurable di Settings)
```

### Panitia Auth Flow
```
1. Scan ID card di frontend/ (kamera QR) atau frontend-tv/ (barcode reader)  
2. POST /login-niy { niy: "67890" }
3. Backend: cari NIY di tabel panitia (cek can_login = true)
   → Jika found: return { user, type: 'panitia' } — TANPA token
4. Frontend: simpan data panitia di localStorage['panitia_session'] sebagai JSON
5. Setiap request: panitia_id dikirim di body request (bukan header)
```

### Admin Auth Flow
```
1. Login via form email + password di frontend-admin/
2. POST /login { email, password }
3. Backend: validate credentials di tabel users
   → Return Sanctum token
4. Frontend-admin: simpan token di AuthContext
```

---

## Catatan Penting

- **Pengawas vs Panitia scan peserta:** Pengawas hanya bisa akses peserta di ruangannya (filter by `jadwal_peserta` pivot). Panitia bisa akses semua peserta.
- **Presensi pengawas/panitia masuk/pulang:** HANYA bisa dilakukan via barcode reader di `frontend-tv`. Jika dilakukan dari `frontend` (smartphone) → ditolak dengan pesan "Akses Ditolak".
- **`can_login` di panitia:** Field boolean. Jika `false`, panitia tidak bisa login meski scan ID card.
