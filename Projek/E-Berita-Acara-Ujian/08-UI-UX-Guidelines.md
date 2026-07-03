---
tags:
  - project
  - ui-ux
created: 2026-06-04
updated: 2026-06-04
---

# UI/UX Guidelines — E-Berita Acara Ujian

## Design Principles

1. **Mobile-first** — `frontend/` digunakan dari smartphone saat berdiri di lapangan
2. **Glanceable** — `frontend-tv/` harus terbaca dari jarak 3–5 meter
3. **Minimum tap** — pengawas harus bisa scan tanpa banyak navigasi
4. **Feedback jelas** — setiap aksi harus ada response visual dalam < 1.5 detik

---

## Color System

### frontend/ & frontend-admin/ (TailwindCSS)

| Warna | Penggunaan |
|-------|-----------|
| `indigo-600` | Aksi utama pengawas, primary CTA |
| `amber-500` | Mode panitia, badge panitia, alert |
| `emerald-500/green` | Status hadir, sukses |
| `red-500` | Error, absen alfa, danger action |
| `slate-800/900` | Text utama |
| `slate-50/100` | Background |

### frontend-tv/ (CSS Variables — Dark Theme)

```css
--accent-blue: #38bdf8
--accent-green: #4ade80
--accent-yellow: #facc15
--accent-red: #ef4444
--accent-orange: #fb923c
--text-primary: rgba(255,255,255,0.92)
--text-secondary: rgba(255,255,255,0.6)
--text-muted: rgba(255,255,255,0.35)
```

---

## Komponen Utama

### QR Scanner (`frontend/src/components/QRScanner.jsx`)
- Menggunakan `html5-qrcode`
- Full screen overlay saat aktif
- Auto-close setelah scan berhasil
- Cooldown 1.5 detik antar scan (mencegah double scan)

### Signature Pad (`frontend/src/components/SignaturePad.jsx`)
- Menggunakan `react-signature-canvas`
- Canvas untuk TTD dengan jari/mouse
- Submit sebagai PNG blob ke API

### Barcode Listener (`frontend-tv/src/components/BarcodeListener.jsx`)
- Listen `keydown` event di `window`
- Barcode reader USB = keyboard input cepat
- Trigger jika Enter diterima
- Tidak memerlukan field input aktif

---

## Layout frontend-tv/ (4 Panel)

```
┌─────────────── HEADER ───────────────────┐
│ Logo + Ujian Aktif + Jam Digital         │
├───────────────────────────────────────────┤
│ STATS STRIP                               │
│ Total | Hadir | Belum | % | Pengawas | ...│
├───────────┬───────────┬──────────┬───────┤
│ Panel 1   │ Panel 2   │ Panel 3  │Panel 4│
│ Peserta   │ Per       │ Per      │Pengawas│
│ Belum     │ Kampus    │ Kelas    │&Panitia│
│ Hadir     │ & Ruang   │(progress)│       │
└───────────┴───────────┴──────────┴───────┘
```

---

## Alert / Feedback Patterns

### SweetAlert2 Patterns (frontend/)

| Situasi | Type | Timer | Toast? |
|---------|------|-------|--------|
| Scan berhasil | success | 1500ms | Ya (top-end) |
| Scan gagal / duplikat | warning | 2500ms | Ya (top-end) |
| Submit berhasil | success | - | Tidak (dialog) |
| Submit gagal | error | - | Tidak (dialog) |
| Sesi expired | warning | - | Tidak (dialog + reload) |
| Login gagal | error | - | Tidak (dialog) |
| Validasi form | warning | - | Tidak (dialog) |

---

## Responsive Breakpoints (frontend/)

- Default: mobile (< 640px) — single column
- `sm:` (640px+) — beberapa komponen jadi row
- `lg:` (1024px+) — grid 2 kolom untuk tabel hadir/belum hadir

---

## UX Rules Penting

1. **Auto-select ujian** — jika hanya 1 ujian aktif, langsung dipilih otomatis
2. **Idle timeout countdown** — timer muncul di header, merah + pulse jika < 1 menit
3. **Missing reports alert** — banner merah untuk berita acara yang tertunggak
4. **Pengawas pengganti** — badge khusus ditampilkan di daftar jika pengawas adalah pengganti
5. **Scanner type badge** — peserta yang discan panitia dapat badge "Panitia: [nama]"
