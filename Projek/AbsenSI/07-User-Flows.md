---
tags: [absensi, user-flows]
updated: 2026-06-25
---

# 07 — User Flows

← [[Projek/AbsenSI/00-INDEX|Index]]

> Draft awal — detail lengkap menyusul setelah Open Questions di absensi-gerbang.md terjawab.

---

## Flow: Siswa/Guru Tap di Gerbang
```
1. Tap kartu di reader kiosk gerbang
2. Kiosk capture UID (keystroke dari HID)
3. Kiosk kirim ke API (atau buffer lokal jika offline)
4. Sistem validasi UID aktif & terdaftar
5. Tentukan tap ke-1 (masuk) atau tap ke-2+ (pulang)
6. Hitung status terlambat (jika tap masuk) berdasarkan jadwal
7. Simpan attendance_record + dispatch event attendance.recorded
8. Kiosk tampilkan konfirmasi (nama + status) ke layar
9. Dashboard TV update realtime via WebSocket
```

## Flow: Admin Ganti Kartu Hilang
```
1. Siswa/guru lapor kartu hilang ke admin
2. Admin buka menu Kartu di dashboard web
3. Admin nonaktifkan kartu lama (status → inactive)
4. Admin registrasi kartu baru, tap kartu baru di reader yang tersambung PC admin
5. Sistem mapping UID baru ke person yang sama
6. Riwayat kartu lama tetap tersimpan untuk audit
```

## ❓ Belum Dibuat
- Flow koreksi manual data absensi oleh admin
- Flow lihat rekap dengan filter di dashboard

