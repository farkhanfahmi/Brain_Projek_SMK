# T011 — Kiosk App: UI Tap Gerbang

## Depends on
T003 (KioskGuard harus sudah ada di API)

## Objective
Buat aplikasi kiosk fullscreen di `apps/kiosk` yang menangkap tap kartu RFID via HID keyboard emulation dan menampilkan feedback kehadiran.

## Context
- **App:** `apps/kiosk` (Next.js, fullscreen, tidak ada navbar)
- **ADR:** ADR-004 (HID keyboard emulation — reader = input field yang capture keystroke)
- **Ref:** `Projek/AbsenSI/06-Features/absensi-gerbang.md`

## Spec Detail

### Env variable yang dibutuhkan:
```
NEXT_PUBLIC_API_URL=http://localhost:3000
KIOSK_DEVICE_TOKEN=xxx   ← dikirim sebagai Bearer di setiap request
KIOSK_ID=gerbang-1       ← dikirim di header X-Kiosk-ID
```

### Halaman utama (`/`) — satu-satunya halaman:

**State: Idle (menunggu tap)**
```
┌──────────────────────────────────────┐
│                                      │
│         [LOGO SEKOLAH]               │
│                                      │
│         07:32:45                     │  ← jam digital realtime
│         Kamis, 3 Juli 2026           │
│                                      │
│    Tempelkan kartu Anda di reader    │
│                                      │
│  [input tersembunyi, auto-focus]     │
└──────────────────────────────────────┘
```

**State: Processing (setelah Enter dari reader, menunggu response API)**
- Tampilkan spinner/loading singkat (< 1 detik idealnya)

**State: Sukses - Hadir**
```
┌──────────────────────────────────────┐
│                                      │
│    ✅  BUDI SANTOSO                  │
│        Kelas XI-RPL-1 · Kampus 1    │
│                                      │
│        MASUK                         │
│        07:32:45                      │
│                                      │
└──────────────────────────────────────┘
```
Background: hijau. Durasi tampil: 3 detik, lalu kembali ke Idle.

**State: Sukses - Terlambat**
Sama seperti Hadir tapi background merah/oranye dan label "TERLAMBAT".

**State: Sukses - Pulang**
Background biru, label "PULANG", tampilkan jam pulang.

**State: Ditolak**
```
┌──────────────────────────────────────┐
│                                      │
│    ❌  KARTU TIDAK DIKENALI          │
│                                      │
│    Hubungi Admin                     │
│                                      │
└──────────────────────────────────────┘
```
Variasi pesan sesuai `result` dari API:
- `rejected_unknown` → "Kartu tidak terdaftar"
- `rejected_inactive` → "Kartu tidak aktif"
- `rejected_locked` → "**Hubungi Guru Piket**" (pesan berbeda, lebih spesifik)
- `rejected_duplicate` → tidak perlu ditampilkan ke user — ini debounce, abaikan saja (kembali ke idle tanpa feedback)

Background: merah gelap. Durasi tampil: 3 detik.

### Logika capture UID:
```typescript
// Input tersembunyi (opacity-0, pointer-events-none, tapi tetap ada di DOM dan focused)
// onKeyDown: akumulasi karakter sampai Enter
// Saat Enter: kirim accumulated string ke API sebagai uid
// Reset accumulated string setelah kirim

// Auto-refocus: kalau user klik sembarangan dan input kehilangan focus,
// document.addEventListener('click', () => inputRef.current?.focus())
```

### Auth header untuk semua request:
```typescript
headers: {
  'Authorization': `Bearer ${process.env.KIOSK_DEVICE_TOKEN}`,
  'X-Kiosk-ID': process.env.KIOSK_ID
}
```

### Indikator status koneksi:
Pojok kiri bawah, font kecil:
- 🟢 Online (saat bisa reach API)
- 🔴 Offline — tap tersimpan lokal (saat API tidak terjangkau)

## JANGAN
- ❌ JANGAN buat navbar, sidebar, atau elemen navigasi apapun — kiosk adalah 1 halaman fullscreen
- ❌ JANGAN buat halaman login untuk kiosk — auth via static device token (tidak ada form login)
- ❌ JANGAN tampilkan data sensitif (NISN, alamat, foto — foto opsional dan bisa ditambah nanti)
- ❌ JANGAN tampilkan feedback untuk tap yang `rejected_duplicate` — cukup kembali ke idle (supaya user tidak bingung)
- ❌ JANGAN buat routing di `apps/kiosk` — satu halaman saja
- ❌ JANGAN implement offline buffer di task ini — itu T014. Task ini hanya: tap → kirim API → tampilkan feedback

## Files
- **Buat:** `apps/kiosk/app/page.tsx` (halaman utama)
- **Buat:** `apps/kiosk/app/layout.tsx` (layout fullscreen, `overflow: hidden`)
- **Buat:** `apps/kiosk/components/TapFeedback.tsx` (komponen feedback)
- **Buat:** `apps/kiosk/components/IdleClock.tsx` (jam digital realtime)
- **Modifikasi:** `apps/kiosk/.env.local` (tambah variabel yang dibutuhkan)

## Acceptance Criteria
- [ ] Halaman tampil fullscreen tanpa scrollbar
- [ ] Input tersembunyi auto-focus saat halaman dimuat
- [ ] Ketik string + Enter di keyboard biasa (simulasi reader) → request dikirim ke API
- [ ] Feedback sukses muncul 3 detik lalu kembali ke idle
- [ ] Feedback "Hubungi Guru Piket" muncul untuk response `rejected_locked`
- [ ] Klik di luar input → focus kembali ke input otomatis
- [ ] Jam digital update setiap detik

## Handoff ke T012
T012 akan membuat endpoint `/attendance/tap` yang dikonsumsi oleh kiosk ini. Sementara T012 belum ada, gunakan mock response untuk testing UI. Setelah T012 selesai, test end-to-end dengan reader RFID fisik.
