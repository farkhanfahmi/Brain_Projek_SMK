---
tags: [workflow, mode, meta]
updated: 2026-08-05
---

# MODE: DISKUSI

> User memanggil file ini di awal sesi untuk menandai: sesi ini KHUSUS diskusi,
> BUKAN eksekusi/coding. Baca ini SEBELUM merespons permintaan apa pun di sesi ini.

## Aturan wajib

1. **JANGAN mengubah kode apa pun** — tidak ada Edit, Write, atau Bash yang menulis/
   menghapus file kode, menjalankan migration, mengubah database, atau commit/push git.
2. **Riset read-only BOLEH dan didorong** — baca kode, grep, jalankan query read-only
   (`SELECT`, bukan `INSERT`/`UPDATE`/`DELETE`), pakai Explore agent untuk verifikasi
   klaim sebelum menulis task, cek graphify kalau relevan.
3. **Klarifikasi requirement lewat AskUserQuestion** untuk keputusan yang ambigu —
   sama seperti biasa, tapi tujuannya untuk MENULIS task yang lengkap, bukan untuk
   langsung eksekusi.
4. **Hasil akhir diskusi WAJIB ditulis jadi task lengkap** di
   `Projek/AbsenSI/06-Features/tasks/T0XX-*.md` (nomor task berikutnya, cek
   `STATUS.md` untuk nomor terakhir yang dipakai) — ikuti format `_task-template.md`
   kalau ada. Task harus cukup detail supaya SESI LAIN bisa eksekusi tanpa perlu
   riset ulang dari nol: sebutkan file yang relevan, keputusan yang sudah dibuat,
   dan keputusan yang masih terbuka (kalau ada).
5. Setelah task ditulis, **update `STATUS.md`** — tambahkan baris task baru di
   tabel yang sesuai (kalau statusnya "belum dikerjakan") supaya sesi eksekusi nanti
   langsung tahu ada task baru menunggu.
6. Jangan cuma bilang "sudah saya diskusikan" di chat tanpa file tertulis — sesi lain
   TIDAK punya akses ke percakapan ini, mereka hanya baca file.

## Kalau user tiba-tiba minta eksekusi di tengah sesi mode ini

Ingatkan singkat: *"Sesi ini sedang mode diskusi (mode-diskusi.md) — mau saya tetap
tulis ini jadi task untuk sesi lain, atau Anda ingin saya eksekusi sekarang juga
(keluar dari mode diskusi)?"* — jangan diam-diam mulai coding hanya karena user
terdengar yakin, konfirmasi eksplisit dulu karena ini pembatalan mode yang sudah
dinyatakan.
