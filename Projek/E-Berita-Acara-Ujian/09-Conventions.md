---
tags:
  - project
  - conventions
created: 2026-06-04
updated: 2026-06-04
---

# Conventions — E-Berita Acara Ujian

## Backend (Laravel)

### Naming
- **Controller:** `NamaController.php` — PascalCase, suffix `Controller`
- **Model:** `NamaModel.php` — PascalCase singular
- **Migration:** `YYYY_MM_DD_HHMMSS_deskripsi_snake_case.php`
- **Service:** `NamaService.php` — untuk business logic yang kompleks

### Database
- **Tabel:** `snake_case` plural — contoh: `jadwal_ujians`, `presensi_pesertas`
- **Kolom:** `snake_case` — contoh: `mulai_ujian`, `total_expected`
- **FK kolom:** `nama_tabel_singular_id` — contoh: `ujian_id`, `pengawas_id`
- **Timestamps:** selalu `created_at` + `updated_at` (Laravel default)
- **Tidak ada soft delete** — semua hard delete (`onDelete('cascade')`)

### API Response Format
```php
// Sukses
return response()->json($data, 200);
return response()->json(['message' => 'Berhasil', 'data' => $data], 201);

// Error
return response()->json(['message' => 'Deskripsi error'], 422);
return response()->json(['message' => 'Not found'], 404);
```

### Service Pattern
Business logic yang kompleks dipisah ke `app/Services/`:
- `PresensiService` — logika scan peserta, login NIY
- `AssignmentService` — logika get jadwal pengawas + daftar peserta
- `DashboardService` — logika agregasi data dashboard

Controller hanya memanggil service dan return response.

---

## Frontend (React — Semua)

### Naming
- **Komponen:** `PascalCase.jsx` — contoh: `QRScanner.jsx`, `PanitiaDashboard.jsx`
- **Hooks:** `useNamaHook.js` — prefix `use`
- **Context:** `NamaContext.jsx`
- **Pages (admin):** `NamaPage.jsx` — PascalCase di folder `pages/`
- **Variabel state:** `camelCase`

### API Calls
- Semua via `axios`
- Base URL dari konstanta: `const API_BASE = '/api'`
- Error handling via `.catch()` atau try/catch + SweetAlert2

### State Management
- Hanya React built-in (`useState`, `useEffect`, `useCallback`, `useRef`)
- Tidak ada Redux/Zustand
- Context hanya untuk auth (`AuthContext`) dan tema (`ThemeContext`) di admin

---

## Pola Khusus yang WAJIB Diikuti

### 1. Scan Processing Guard
```jsx
const isProcessingScan = useRef(false);
// Setiap handler scan:
if (isProcessingScan.current) return;
isProcessingScan.current = true;
// ... logika scan
setTimeout(() => { isProcessingScan.current = false; }, 1500);
```
**Alasan:** Mencegah double-processing jika kamera scan terlalu cepat.

### 2. Dual-Mode Auth Check (frontend/)
```jsx
// Selalu cek userType sebelum kirim request
if (userType === 'panitia') {
  params.panitia_id = formData.panitia_id;
} else {
  params.pengawas_id = formData.pengawas_id;
}
```

### 3. `manualPresensiBulk` Transaksi
Backend WAJIB gunakan `DB::transaction()` untuk `POST /manual-presensi-bulk`.
Sudah diimplementasi — jangan diubah tanpa alasan.

### 4. Signature Upload
Signature dikirim sebagai PNG blob via `FormData`, bukan base64.
```jsx
const sigBlob = await new Promise(resolve =>
  sigCanvas.getCanvas().toBlob(resolve, 'image/png')
);
data.append('signature', sigBlob, 'signature.png');
```

### 5. Pengawas NIY — Multiple Records
Satu pengawas dengan NIY yang sama bisa punya multiple record di tabel `pengawas` (karena ujian berbeda). Query **selalu** cari by NIY, bukan by ID tunggal:
```php
$pengawasIds = Pengawas::where('niy', $niy)->pluck('id')->toArray();
```

---

## Yang DILARANG

1. ❌ Jangan buat keputusan arsitektur baru di Claude Code — tulis di CONTEXT.md "Masalah Ditemukan"
2. ❌ Jangan install package baru tanpa konfirmasi eksplisit
3. ❌ Jangan refactor kode yang tidak diminta
4. ❌ Jangan ubah struktur migration yang sudah ada — buat migration baru
5. ❌ Jangan hard-code string NIY atau ID di kode
6. ❌ Jangan commit file `.sql` ke git
