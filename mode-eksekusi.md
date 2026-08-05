---
tags: [workflow, mode, meta]
updated: 2026-08-05
---

# MODE: EKSEKUSI

> User memanggil file ini di awal sesi untuk menandai: sesi ini boleh LANGSUNG
> coding tanpa tanya "diskusi dulu atau eksekusi?" di setiap permintaan baru.

## Aturan wajib

1. **Boleh langsung eksekusi** (edit kode, migration, dsb) untuk permintaan fitur/
   perbaikan tanpa perlu tanya ulang "mau didiskusikan dulu atau eksekusi sekarang?"
   — itu sudah dijawab oleh pemanggilan file ini.
2. **Tetap tanya untuk keputusan yang ambigu/berisiko**, ini TIDAK berubah dari mode
   manapun — AskUserQuestion tetap wajib dipakai kalau ada beberapa pilihan desain
   yang sama-sama masuk akal, atau kalau ada operasi destruktif/berisiko tinggi
   (lihat aturan destructive-action di system prompt & memory insiden database
   wipe — itu SELALU berlaku, tidak pernah di-skip oleh mode eksekusi ini).
3. **Verifikasi tetap wajib** — `tsc --noEmit`, build, dan testing manual/Playwright
   sebelum melaporkan selesai. Mode eksekusi mempercepat IZIN untuk mulai kerja,
   BUKAN alasan untuk melewati langkah verifikasi.
4. **Boleh push ke production** kalau workflow proyek memang begitu (auto-deploy
   via git hook, sesuai `10-Environment.md`) — tapi tetap ikuti kehati-hatian
   standar untuk operasi database (backup dulu, cek migration state, dst).
5. Task yang ditulis di mode-diskusi sebelumnya (`06-Features/tasks/T0XX-*.md`)
   boleh langsung dieksekusi di sini tanpa perlu didiskusikan ulang — baca task-nya,
   ikuti detailnya, tanya HANYA kalau task itu sendiri menandai ada open question
   yang belum terjawab.
6. Setelah eksekusi selesai, **update status task** (checklist/tabel progress) di
   file task terkait dan di `STATUS.md` — jangan cuma lapor selesai di chat.

## Kalau user minta sesuatu yang sangat ambigu/besar

Mode eksekusi bukan berarti tebak-tebak sendiri untuk keputusan besar yang belum
jelas arahnya — tetap pakai AskUserQuestion untuk itu. Mode ini cuma menghilangkan
1 pertanyaan spesifik ("eksekusi atau diskusi?"), bukan semua pertanyaan klarifikasi.
