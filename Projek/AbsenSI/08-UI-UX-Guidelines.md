---
tags: [absensi, ui-ux]
updated: 2026-06-25
---

# 08 — UI/UX Guidelines

← [[Projek/AbsenSI/00-INDEX|Index]]

> Belum dirancang detail — placeholder. Akan diisi saat Developer 1 & 2 mulai breakdown task `apps/web` dan `apps/kiosk`.

## Prinsip Awal
- **Kiosk gerbang:** UI harus sangat sederhana, kontras tinggi, font besar — dilihat sekilas oleh siswa yang tap sambil jalan, bukan dibaca lama
- **Dashboard TV:** dilihat dari jarak jauh, update realtime tanpa perlu interaksi
- **Dashboard admin:** standar admin panel, prioritas kemudahan filter/rekap data

## ✅ Keputusan Stack UI (2026-07-03)

- [x] **Component library: shadcn/ui** — komponen di-copy langsung ke `packages/ui` dalam monorepo (bukan dependency eksternal), berbasis Tailwind CSS + Radix UI primitives. Komponen yang dibutuhkan (Table, DatePicker, Dialog, Form, Badge, Select) semua tersedia. Developer bisa customisasi bebas tanpa terikat opini library. Referensi: https://ui.shadcn.com
- [ ] **Palet warna & branding sekolah** — diisi Developer 1 saat mulai breakdown task `apps/web`. Tailwind config di `packages/config/tailwind.config.ts` jadi satu sumber kebenaran warna untuk semua apps.

