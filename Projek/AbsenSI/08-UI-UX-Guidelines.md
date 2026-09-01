---
tags: [absensi, ui-ux]
updated: 2026-08-31
---

# 08 — UI/UX Guidelines

← Index (00-INDEX AbsenSI.md)

> **[2026-08-31] Dokumen ini SUPERSEDED** — sebelumnya placeholder murni dari 2026-06-25 ("belum dirancang detail"), padahal Design System v2 lengkap sudah dibangun (audit 4-tahap, 28→komponen v1, lalu revamp total v2 dengan 7 fase: token architecture, foundation rules, component contracts, data-dense patterns, Figma build, governance). Isi lengkap sekarang ada di 3 lapis terpisah — file ini HANYA index pointer, jangan tulis detail visual di sini lagi.

## Ke Mana Harus Pergi untuk Apa

| Kebutuhan | Lokasi |
|---|---|
| **Token desain (warna, spacing, radius, typography) — source of truth mesin** | `packages/design-tokens/` (repo kode) — `primitive.tokens.json`, `semantic.tokens.json`, `foundation-rules.json` |
| **Kontrak komponen (Button, Input, DataTable, dll — 14 total)** | `packages/design-tokens/src/component-contracts/*.json` |
| **Cermin visual di Figma** | File "AbsenSI", page **"Design System v2"** — 7 Variable Collections + 4 Effect Styles + 9 Text Styles + 14 komponen |
| **Narasi & rasional keputusan desain (kenapa, bukan apa)** | Vault `06-Features/design-system-v2/00-INDEX.md` (peta 7 fase lengkap) |
| **Governance & migrasi bertahap (linter, escape hatch komponen baru)** | `06-Features/design-system-v2/07-fase-6-governance-migration.md`, `DESIGN_SYSTEM_AGENT.md` (root repo) |
| **Design system v1 lama (arsip, sebagian sudah revamp jadi v2)** | `06-Features/design-system/MASTER.md` — masih relevan untuk komponen yang BELUM dimigrasi ke v2 |

## Warna Terkunci (Ringkasan Cepat)

Terracotta `#C1452C` (primary) + Cream `#FFFEE0` (background) — **tidak boleh diubah lagi** tanpa diskusi ulang eksplisit. Detail lengkap ramp warna di `packages/design-tokens/src/primitive.tokens.json`.

## Prinsip UI Tetap Berlaku (dari versi awal, masih valid)

- **Kiosk gerbang:** UI sangat sederhana, kontras tinggi, font besar — dilihat sekilas oleh siswa yang tap sambil jalan.
- **Dashboard TV:** dilihat dari jarak jauh, update realtime tanpa interaksi.
- **Dashboard admin/guru/piket:** mobile-first (base Tailwind = mobile, baru `sm:/md:/lg:` untuk desktop), tabel wajib search+sort+kolom "No".

## Component Library

**shadcn/ui** — komponen di-copy langsung ke `packages/ui` (monorepo, bukan dependency eksternal), berbasis Tailwind CSS + Radix UI primitives. Tailwind config di `packages/config/tailwind.config.ts` — tapi nilai warna/spacing SEKARANG bersumber dari `packages/design-tokens/`, bukan hardcode di config.

## Status Migrasi

Belum semua modul kode sudah dimigrasi ke token v2 — linter `absensi_ds_v2_linter.py` (di `hermes/scripts/`) mendeteksi violation nyata (warna hardcode, dll) di beberapa file (`kampus-map.tsx`, `rekap-view.tsx`, `rekap-guru-view.tsx`). Migrasi dilakukan bertahap per modul, lihat `06-Features/design-system-v2/07-fase-6-governance-migration.md` untuk proses & urutan.
