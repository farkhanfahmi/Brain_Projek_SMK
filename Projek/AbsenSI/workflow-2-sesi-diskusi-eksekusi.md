---
tags: [absensi, workflow, ai-agent]
created: 2026-08-29
status: aktif
---

# Konvensi 2 Sesi Hermes -- Diskusi vs Eksekusi

> Pelengkap dari `workflow-ai-agent-full-hermes.md`. Menjawab: bagaimana user memisahkan sesi Hermes untuk DISKUSI/PLANNING vs sesi Hermes untuk EKSEKUSI, meniru pola lama Claude Code (sesi diskusi terpisah dari sesi eksekusi).

## Kenapa dipisah jadi 2 chat/session, bukan 1 chat

Beda dari Claude Code CLI yang punya flag `--permission-mode plan` di level proses, Hermes desktop tidak punya toggle mode teknis semacam itu -- pemisahannya dilakukan di level SESSION (chat window), bukan di level flag. User memilih tegas: 2 chat terpisah, supaya benar-benar tidak ada risiko sesi "diskusi" tergelincir eksekusi karena momentum percakapan.

## Struktur 2 Sesi

### Sesi A -- "AbsenSI Planning" (Diskusi/Perencanaan)
- **Tujuan:** brainstorming arsitektur, audit read-only, klarifikasi requirement, evaluasi kritis proposal, menulis task lengkap.
- **ATURAN KERAS:** tidak menulis/mengubah kode aplikasi. Boleh membaca file, browsing, riset, Figma, tapi TIDAK PATCH/WRITE ke `apps/`, `packages/ui`, dst.
- **Output WAJIB:** setiap hasil diskusi yang perlu dieksekusi ditulis sebagai task lengkap di `06-Features/tasks/task-<MODUL>-<NNN>-<slug>.md` MENGIKUTI FORMAT `_task-template.md` yang sudah ada (Assigned, Depends on, Objective, Context, Spec Detail, Business Rules, Edge Cases, Files, Acceptance Criteria) -- format ini SAMA dengan yang dipakai era Claude Code, supaya kompatibel dibaca sesi eksekusi manapun (Hermes atau Claude Code CLI).
- **Boleh pakai skill `plan`** (`.hermes/plans/`) untuk draft teknis internal SEBELUM ditulis final ke task-template -- plan skill untuk mikir, task-template untuk hand-off resmi.

### Sesi B -- "AbsenSI Eksekusi" (Implementasi)
- **Tujuan:** mengerjakan task yang SUDAH ditulis lengkap oleh Sesi A. Termasuk memanggil Claude Code CLI di belakang layar (lihat `workflow-ai-agent-full-hermes.md`) untuk implementasi kode luas.
- **ATURAN KERAS:** tidak mendiskusikan ulang requirement dari nol -- kalau task tidak jelas/ambigu, STOP dan lempar balik ke user untuk diperjelas di Sesi A dulu, jangan menebak/improvisasi requirement baru di sesi eksekusi.
- **Setiap task selesai:** centang checkbox Acceptance Criteria di file task, update status di tabel progress modul terkait (kebiasaan lama yang sudah berlaku, lihat memory: "update status vault saat task selesai").

## Cara Membuat 2 Sesi Ini di Hermes

1. Buka chat baru di Hermes, beri nama "AbsenSI Planning" -- pakai untuk semua diskusi/audit/perencanaan AbsenSI.
2. Buka chat baru KEDUA, beri nama "AbsenSI Eksekusi" -- pakai HANYA saat mengerjakan task yang file-nya sudah ada di `06-Features/tasks/`.
3. Kalau proyek dianchor sebagai Hermes Project (opsional), kedua sesi tetap bisa berbagi workspace/folder yang sama -- pemisahan ada di LEVEL PERCAKAPAN, bukan level file.

## Alur Kerja End-to-End

```
User buka Sesi A (Planning)
  -> diskusi fitur/audit/evaluasi
  -> Hermes tulis task lengkap ke 06-Features/tasks/task-XXX-NNN.md
  -> User review & approve task tsb

User pindah ke Sesi B (Eksekusi)
  -> "kerjakan task-WEB-014"
  -> Hermes baca file task, eksekusi (patch langsung ATAU panggil Claude Code CLI utk kode luas)
  -> Hermes centang Acceptance Criteria, update status progress
  -> Hermes lapor hasil + biaya (jika via Claude Code CLI) ke user
```

## Kenapa Ini Lebih Baik dari Sekadar "Tanya Diskusi/Eksekusi" di 1 Chat

Pola lama (tanya "diskusi atau eksekusi?" di awal tugas, 1 chat) tetap valid untuk tugas KECIL/cepat. Pemisahan 2 sesi ini dipakai untuk tugas BESAR yang butuh jarak psikologis+teknis lebih tegas antara "mikir" dan "kerja" -- konsisten dengan preferensi user yang sudah lama: untuk pekerjaan multi-fase besar, tampilkan roadmap dulu sebelum eksekusi (lihat memory).

## Kompatibilitas dengan Claude Code

Karena format task sama persis dengan `_task-template.md` era Claude Code, task yang ditulis Sesi A Hermes BISA dikerjakan oleh:
- Sesi B Hermes (baca task -> eksekusi/panggil Claude Code CLI), ATAU
- Claude Code langsung di VS Code (user buka manual, kasih path file task) -- fleksibel, bukan terkunci ke satu alat.
