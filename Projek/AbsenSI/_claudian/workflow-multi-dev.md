---
tags: [absensi, workflow, claudian, multi-dev]
updated: 2026-06-25
---

# Workflow Claudian — AbsenSI (Tim 3 Developer)

← [[Projek/AbsenSI/00-INDEX|Index]]

> Ini adalah **varian** dari [[Projek/AbsenSI/00-INDEX|00-INDEX]] global, disesuaikan untuk tim 3 developer yang bekerja paralel di modul berbeda dalam satu monorepo. Prinsip Layer 1/2/3 (Think → Brief → Build) tetap sama. Yang berubah: bagaimana Layer 2 (CONTEXT.md) dikelola supaya tidak collision antar developer.

---

## Perbedaan Utama dari Workflow Solo

| Aspek | Workflow Solo (lama) | Workflow Multi-Dev (AbsenSI) |
|---|---|---|
| CONTEXT.md | 1 file di root project | **1 file per app**: `apps/api/CONTEXT.md`, `apps/web/CONTEXT.md`, `apps/kiosk/CONTEXT.md` |
| Task ID | `task-001-xxx` | **Prefix modul**: `task-CORE-001`, `task-WEB-001`, `task-KIOSK-001` — langsung kelihatan ownership-nya |
| Task brief | Field bebas | **Wajib ada field `Assigned: Dev [1/2/3]`** |
| 12-Status.md | List linear | **Board per modul** (3 kolom: Core / Web / Kiosk) |
| Review | Tidak wajib | **Wajib 1 approval** untuk PR yang sentuh `apps/api` (modul Core) atau `packages/types` — karena breaking change di sini berdampak ke semua app. PR yang scoped murni ke modul sendiri (`apps/web` saja, `apps/kiosk` saja) boleh self-merge setelah CI hijau. |

---

## Kenapa CONTEXT.md Dipisah per App

Kalau 3 developer jalankan Claude Code session secara bersamaan di modul masing-masing, dan semua menulis ke **satu** `CONTEXT.md` di root — sesi Developer 1 bisa overwrite/conflict dengan brief yang baru ditulis Developer 2. Dengan CONTEXT.md per app:

```
absensi-rfid/
├── CLAUDE.md                ← shared, jarang berubah (stack, aturan wajib semua modul)
├── apps/
│   ├── api/CONTEXT.md       ← working memory Developer 3
│   ├── web/CONTEXT.md       ← working memory Developer 1
│   └── kiosk/CONTEXT.md     ← working memory Developer 2
```

Setiap Claude Code session dijalankan **dari folder app masing-masing** atau diberi instruksi eksplisit "baca apps/[nama]/CONTEXT.md" — supaya tidak baca/tulis context modul orang lain.

---

## Workflow per Task (Multi-Dev)

```
STEP 1 — DISKUSI & RANCANGAN [Obsidian + Claudian]
────────────────────────────────────────────────────
Sama seperti workflow solo. Hasil: spec final di 06-Features/[modul].md
Kalau spec menyentuh >1 modul (misal: kiosk butuh endpoint baru di api),
Claudian WAJIB tandai di spec mana bagian Developer 2 (kiosk) dan
mana bagian Developer 3 (api) — supaya saat pecah jadi task, jelas
siapa ngerjain apa dan urutan dependency-nya (api duluan baru kiosk,
biasanya).

STEP 2 — TULIS TASK BRIEF [Claudian]
─────────────────────────────────────
Task ID pakai prefix modul: task-CORE-XXX / task-WEB-XXX / task-KIOSK-XXX
Field WAJIB tambahan: "Assigned: Dev [nama]"
Kalau task punya dependency ke task lain (misal task-KIOSK-003 butuh
task-CORE-005 selesai dulu) — tulis eksplisit "Depends on: task-CORE-005"
di task brief. JANGAN biarkan Claude Code mulai kerja sebelum dependency
itu selesai (akan dapat API yang belum ada).

STEP 3 — SYNC KE CONTEXT.md APP YANG SESUAI [Claudian]
────────────────────────────────────────────────────────
Claudian tulis brief minimal ke apps/[modul]/CONTEXT.md — BUKAN ke
root CONTEXT.md. Pastikan brief mencantumkan wikilink ke task file
Obsidian dan (jika ada) wikilink ke task dependency-nya.

STEP 4 — EKSEKUSI [Claude Code, per developer]
─────────────────────────────────────────────────
Developer buka Claude Code di folder app miliknya sendiri. Claude Code
baca CLAUDE.md (root, shared) + apps/[modul]/CONTEXT.md (miliknya saja).
Kalau task butuh ubah packages/types (shared) — Claude Code WAJIB
berhenti dan minta konfirmasi user dulu sebelum ubah, karena ini
berdampak ke app lain.

STEP 5 — WRITE HASIL KE OBSIDIAN [Claude Code]
─────────────────────────────────────────────────
Sama seperti workflow solo — append ke 14-Debug-Log.md, update
12-Status.md (BAGIAN KOLOM MODUL YANG SESUAI, jangan ubah kolom modul
lain), update 11-Decisions.md kalau ada keputusan teknis baru.

STEP 6 — PR & REVIEW
──────────────────────
- PR scoped ke 1 modul saja (apps/web ATAU apps/kiosk) → boleh self-merge
  setelah CI hijau, tidak perlu approval dev lain
- PR yang sentuh apps/api ATAU packages/types → WAJIB approval minimal
  1 developer lain sebelum merge ke main
- Branch naming: feat/[modul]-[deskripsi-singkat], contoh:
  feat/kiosk-offline-buffer, feat/core-jadwal-mengajar

STEP 7 — EVALUATE [Claudian]
───────────────────────────────
User kembali ke Claudian, baca 14-Debug-Log + 12-Status seperti biasa.
Karena ada 3 developer, Claudian boleh diminta evaluasi progress LINTAS
modul sekaligus (misal: "apakah task-KIOSK-003 sudah bisa start, dependency
task-CORE-005 sudah selesai?") — Claudian cek 12-Status.md board untuk jawab.
```

---

## Template Task Brief (Multi-Dev)

Salin dari [[Projek/AbsenSI/06-Features/tasks/_task-template|_task-template.md]] — sudah include field `Assigned` dan `Depends on`.

---

## Hal yang TIDAK Berubah dari Workflow Solo

- Layer 1 (Obsidian) tetap satu-satunya source of truth untuk keputusan
- Prinsip zero-ambiguity tetap berlaku — spec harus jelas siapa pun yang baca
- Claude Code tetap tidak boleh buat keputusan arsitektur sendiri
- Model selection guide (Haiku/Sonnet/Opus) tetap sama, lihat [[Projek/AbsenSI/00-INDEX|00-INDEX]] global

