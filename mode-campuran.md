---
tags: [workflow, mode, meta]
updated: 2026-08-05
---

# MODE: CAMPURAN

> User memanggil file ini di awal sesi untuk menandai: default sesi ini adalah
> TANYA DULU per topik, bukan otomatis diskusi-only maupun otomatis eksekusi.
> Ini mode paling aman kalau user belum tahu di awal sesi mau ngapain — cocok
> untuk sesi yang isinya campuran (kadang diskusi arsitektur, kadang perbaikan
> kecil yang sekalian dieksekusi).

## Aturan wajib

1. **Untuk SETIAP permintaan fitur/perbaikan baru** (bukan pertanyaan informasi biasa,
   bukan riset), tanya dulu lewat AskUserQuestion: *"Ini mau didiskusikan dulu (saya
   tulis jadi task) atau langsung dieksekusi sekarang?"* — sebelum mulai Edit/Write/
   migration apa pun.
2. **Kalau user sudah eksplisit bilang** "langsung kerjakan"/"eksekusi sekarang"/
   "tolong perbaiki" dengan nada instruksi langsung (bukan "bagaimana kalau...") di
   pesan yang sama — itu sudah jawaban, TIDAK perlu tanya lagi untuk permintaan itu.
3. **Task multi-langkah yang sudah disetujui** eksekusinya di awal — lanjutkan tanpa
   tanya ulang di tiap sub-langkah, kecuali ada percabangan keputusan baru yang
   signifikan.
4. Kalau user pilih "diskusi dulu" untuk 1 topik, ikuti aturan `mode-diskusi.md`
   UNTUK TOPIK ITU SAJA — topik lain di sesi yang sama tetap perlu ditanya ulang.
5. Kalau user pilih "eksekusi" untuk 1 topik, ikuti aturan `mode-eksekusi.md`
   UNTUK TOPIK ITU SAJA.
6. **Riset read-only selalu boleh** tanpa tanya, di mode manapun.

## Kenapa mode ini ada

Ini yang paling dekat dengan "default aman" — mencegah kejadian di mana Claude
mengasumsikan salah satu mode (diskusi-only ATAU eksekusi-langsung) tanpa
konfirmasi, padahal preferensi user bisa beda-beda per topik bahkan dalam 1 sesi
yang sama.
