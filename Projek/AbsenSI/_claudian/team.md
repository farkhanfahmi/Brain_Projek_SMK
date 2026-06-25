---
tags: [absensi, team, ownership]
updated: 2026-06-25
---

# Team & Module Ownership

← [[30.Projects/AbsenSI/00-INDEX|Index]]

> Pembagian modul **diusulkan** berdasarkan background yang disampaikan saat diskusi awal. Ini bukan keputusan final — silakan dikoreksi/disesuaikan tim, lalu update file ini.

---

## Anggota Tim

| Slot | Background | Kebiasaan Stack Lama |
|---|---|---|
| **Developer 1** (Anda) | Laravel, lebih sering kerja frontend | MySQL |
| **Developer 2** | Laravel, lebih sering kerja frontend | MySQL |
| **Developer 3** | Fullstack | MySQL |

> Isi nama asli & kontak di sini setelah dikonfirmasi.

---

## Usulan Pembagian Modul

| Modul | Diusulkan untuk | Alasan |
|---|---|---|
| **`apps/web`** (Dashboard Admin, Next.js) | Developer 1 | Leverage kekuatan frontend yang sudah ada, React component model relatif dekat dengan kebiasaan frontend Laravel/Blade+Alpine |
| **`apps/kiosk`** (Halaman tap gerbang + TV, Next.js) | Developer 2 | Juga frontend-leaning. Karena reader HID, kiosk app pada dasarnya web app menangkap keystroke + panggil API — entry point yang gentle ke ekosistem JS, tidak butuh pemahaman backend dalam |
| **`apps/api`** (NestJS — Core, Attendance, Card, Schedule, Notification) | Developer 3 | Modul paling kritis & berisiko tinggi (sumber kebenaran data, logic bisnis utama) — cocok dipegang yang paling fullstack/berpengalaman |
| **`packages/types`, `packages/config`** | Shared — siapa pun boleh ubah, tapi WAJIB diberitahu 2 dev lain sebelum merubah struktur (breaking change ke semua app) | |

**Catatan:** Developer 1 & 2 disarankan tetap belajar dasar NestJS meski tidak owning modul itu — supaya tidak buta total kalau Developer 3 berhalangan. Tidak perlu jadi expert, cukup paham struktur module/DI dasar.

---

## Aturan Kolaborasi

1. **Task selalu punya field "Assigned to"** di task brief — lihat [[30.Projects/AbsenSI/06-Features/tasks/_task-template|_task-template.md]]. Jangan kerjakan task modul orang lain tanpa koordinasi dulu.
2. **CONTEXT.md terpisah per app** (`apps/api/CONTEXT.md`, `apps/web/CONTEXT.md`, `apps/kiosk/CONTEXT.md`) — bukan satu CONTEXT.md di root. Ini supaya 3 Claude Code session paralel tidak rebutan/override file context yang sama. Detail di [[30.Projects/AbsenSI/_claudian/workflow-multi-dev|workflow-multi-dev.md]].
3. **Perubahan ke `packages/types`** (shared) wajib di-flag ke 2 dev lain sebelum merge — karena breaking change di sini berdampak ke semua app.
4. **Modul Core (`apps/api`)** adalah satu-satunya sumber kebenaran data siswa/guru/jadwal. Modul lain TIDAK BOLEH query database langsung ke tabel Core — selalu lewat service/API layer yang disediakan. Ini supaya batas modul tetap bisa dipecah jadi microservice nanti tanpa rewrite besar (lihat ADR-003).
5. **Async-first** — karena semua anggota tim adalah guru aktif dengan waktu terbatas, koordinasi utama lewat update [[30.Projects/AbsenSI/12-Status|12-Status.md]] (board per modul), bukan mengandalkan meeting sinkron.
