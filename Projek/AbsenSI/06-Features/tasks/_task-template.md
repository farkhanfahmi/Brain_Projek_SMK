# Task-[MODUL-XXX]: [Nama Task]

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk). Contoh: task-CORE-001.

## Assigned
**Developer:** [Dev 1 / Dev 2 / Dev 3 — nama]
**Recommended Model:** [Haiku / Sonnet / Opus — lihat panduan di Claudian-Workflow global]

## Depends on
[task-XXX-YYY jika ada dependency. Tulis "Tidak ada" jika tidak ada. JANGAN mulai eksekusi sebelum dependency selesai.]

## Objective
[Satu kalimat: apa yang harus selesai setelah task ini dikerjakan]

## Context
- **Modul:** [Core / Attendance / Card / Schedule / Notification / Web / Kiosk]
- **DB Tables:** [tabel yang terlibat]
- **API Endpoints:** [endpoint yang terlibat]

## Spec Detail
(Salin langsung dari 06-Features/[modul].md — bukan link, tapi teks lengkap)

**Input:**
- field: type, required/optional, validasi

**Output sukses:**
{ contoh response }

**Output gagal:**
{ contoh response + kode HTTP }

## Business Rules
- Rule 1
- Rule 2

## Edge Cases
- Case: [kondisi] → [behavior yang diharapkan]

## Files
- **Buat:** `apps/[app]/[path]`
- **Modifikasi:** `apps/[app]/[path]` — [apa yang diubah]
- **Jangan sentuh:** `apps/[app]/[path]`
- **⚠️ Kalau task ini butuh ubah `packages/types` (shared):** WAJIB stop dan minta konfirmasi user dulu — breaking change ke app lain.

## Acceptance Criteria
- [ ] Kriteria 1
- [ ] Kriteria 2
- [ ] Kriteria 3

## Validasi Claudian
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua edge case sudah tercakup
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan)
- [ ] Tidak ada konflik dengan keputusan di 11-Decisions.md
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign
- [ ] Field "Assigned" sudah diisi nama developer yang benar
