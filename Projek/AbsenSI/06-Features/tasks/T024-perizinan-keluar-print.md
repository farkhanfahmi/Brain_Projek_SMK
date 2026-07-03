# T024 — Dashboard Piket: Perizinan Keluar + Print Surat

## Depends on
T022 (permits API), T023 (layout piket sudah ada)
⚠️ PRASYARAT NON-CODING: Edit `C:\ProjekSMK\print.php` untuk tambahkan parameter `kode` SEBELUM task ini di-deploy ke production.

## Objective
Buat menu khusus perizinan keluar dengan dua sub-alur (izin keluar sementara dan konfirmasi izin pulang setelah tap), dan integrasi cetak surat via print.php yang sudah ada.

## Context
- **App:** `apps/web`
- **Route:** `/piket/izin-keluar`
- **Role:** `guru_piket`
- **Print server:** `http://10.10.10.100:8800/print.php`
- **Script fisik:** `C:\ProjekSMK\print.php`
- **ADR:** ADR-018 (reuse print.php)
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-piket.md` — Fungsi 3 (Sub-alur A & B)

## Spec Detail

### Halaman `/piket/izin-keluar` — dua section:

**Section A: Izin Keluar Sementara (siswa tidak tap)**

Form:
- Cari siswa (autocomplete — hanya siswa kampus piket yang login)
- Alasan kategori: radio [Sakit] [Izin]
- Keterangan: textarea
- Jam keluar: time picker (default: jam sekarang)
- Jam kembali diharapkan: time picker (optional)

Submit → `POST /permits(jenis: keluar)` → response berisi `kode_verifikasi`

Setelah submit berhasil:
1. Konstruksi URL print:
```typescript
const printUrl = new URL(process.env.NEXT_PUBLIC_PRINT_SERVER_URL);
printUrl.searchParams.set('petugas', namaUserPiket);
printUrl.searchParams.set('tgl', tanggal);
printUrl.searchParams.set('nama', namaSiswa);
printUrl.searchParams.set('kls', kelasSiswa);
printUrl.searchParams.set('alasan', alasanDetail);
printUrl.searchParams.set('ket', alasanKategori);
printUrl.searchParams.set('jamkembali', jamKembaliDiharapkan ?? '');
printUrl.searchParams.set('kode', kodeVerifikasi);
```
2. `window.open(printUrl.toString(), '_blank')` — buka di tab baru
3. Tampilkan pesan: "Surat izin sudah dibuka di tab baru. Klik Print di browser untuk mencetak."

**Section B: Konfirmasi Izin Pulang (setelah siswa tap)**

- Daftar siswa yang tap pulang hari ini (`waktu_pulang IS NOT NULL`, `pulang_via = 'tap'`)
- Piket cari nama siswa
- Tombol **[Tandai Izin Pulang]** → `POST /attendance/confirm-izin-pulang/:record_id`
- Konfirmasi dialog: "Ubah status pulang Budi Santoso menjadi 'Izin Pulang'?"

**Fallback: tidak bisa tap**
- Tombol **[Input Manual Pulang]** → form: cari siswa, input jam pulang, catatan
- `POST /attendance/manual-pulang`

### print.php parameter mapping:
| Parameter | Sumber |
|---|---|
| `petugas` | `req.user.username` atau nama guru piket |
| `tgl` | `permits.tanggal` format DD/MM/YYYY |
| `nama` | `students.nama` |
| `kls` | `kelas.nama` |
| `alasan` | `permits.alasan_detail` |
| `ket` | `permits.alasan_kategori` ("Sakit" atau "Izin") |
| `jamkembali` | `permits.jam_kembali_diharapkan` atau string kosong |
| `kode` | `permits.kode_verifikasi` |

### env variable di `apps/web`:
```
NEXT_PUBLIC_PRINT_SERVER_URL=http://10.10.10.100:8800/print.php
```

## JANGAN
- ❌ JANGAN build mekanisme print baru — hanya konstruksi URL dan `window.open()` (ADR-018)
- ❌ JANGAN auto-print — user harus klik Print manual di browser preview
- ❌ JANGAN buat siswa tap saat keluar — izin keluar tidak memerlukan tap (keputusan eksplisit)
- ❌ JANGAN hardcode URL print.php di kode — gunakan env variable
- ❌ JANGAN lupa `window.open()` hanya berfungsi kalau tidak diblokir popup blocker — tambahkan instruksi di UI: "Pastikan popup tidak diblokir di browser ini"

## Files
- **Buat:** `apps/web/app/piket/izin-keluar/page.tsx`
- **Buat:** `apps/web/app/piket/izin-keluar/components/FormIzinKeluar.tsx`
- **Buat:** `apps/web/app/piket/izin-keluar/components/KonfirmasiIzinPulang.tsx`
- **Modifikasi:** `apps/web/.env.local` — tambah `NEXT_PUBLIC_PRINT_SERVER_URL`

## Acceptance Criteria
- [ ] Isi form izin keluar → submit → tab baru terbuka dengan URL print.php yang benar
- [ ] URL print.php mengandung semua parameter termasuk `kode`
- [ ] Kode verifikasi di URL sesuai dengan yang tersimpan di database
- [ ] Section B: siswa yang tap pulang muncul di daftar
- [ ] Klik [Tandai Izin Pulang] → `attendance_records.pulang_via` berubah jadi `tap_izin_pulang`
- [ ] Input manual pulang berhasil update `attendance_records`

## ⚠️ Sebelum production deployment:
Edit `C:\ProjekSMK\print.php` — tambahkan di template HTML:
```php
<p>Kode: <?= htmlspecialchars($_GET['kode'] ?? '') ?></p>
```
Test print surat di browser untuk verifikasi kode muncul.
