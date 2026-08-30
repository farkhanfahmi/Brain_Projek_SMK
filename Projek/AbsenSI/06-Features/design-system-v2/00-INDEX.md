---
tags: [absensi, design-system-v2]
created: 2026-08-28
status: in-progress
---

# Design System AbsenSI v2 -- Revamp Total

Dokumen ini adalah hasil kerja "Senior UI/UX Architect & Design System Engineer" untuk merombak total design system AbsenSI, dipicu oleh audit mendalam yang menemukan design system v1 jauh dari lengkap sebagai produk mandiri. Lihat 00-audit-lengkap-v1.md untuk detail temuan yang memicu revamp ini.

## Tujuan

Membangun Design System yang sangat detail, rigid, dan presisi secara teknis, dijadikan single source of truth yang bisa dikonsumsi AI coding agent (Claude Code, Hermes, dll) untuk menulis kode UI tanpa ruang untuk halusinasi atau tebakan kreatif menyimpang.

## Batasan Non-Negotiable dari Product Owner

| Aturan | Nilai |
|---|---|
| Warna Terracotta (primary/brand) | #C1452C -- dikunci exact, tidak boleh diubah |
| Warna Cream (background utama) | #FFFEE0 -- dikunci exact, tidak boleh diubah |
| Warna sekunder/netral/semantik/aksen | Bebas diubah/ditambah untuk mendukung keterbacaan data & harmoni |
| Fokus visual | Data-dense UI: tabel kompleks, dashboard analitik, grafik, visualisasi data |
| Rollout | Incremental (bertahap per modul) -- BUKAN big-bang release |
| Governance | Wajib ada escape hatch resmi untuk pengajuan komponen/varian baru |

## Arsitektur 3 Lapis Pengetahuan

Design system v2 sengaja dipecah jadi TIGA lapis paralel dengan peran berbeda, bukan satu tumpukan campur aduk seperti v1 (yang cuma Figma). Ini krusial untuk akurasi eksekusi AI agent nantinya -- setiap lapis punya SATU tanggung jawab, tidak overlap.

```
                    ┌───────────────────────────────┐
                    │   packages/design-tokens/       │
                    │   (JSON, repo AbsenSI)           │
                    │                                   │
                    │   PERAN: SOURCE OF TRUTH MESIN    │
                    │   Dibaca LANGSUNG oleh AI coding  │
                    │   agent saat menulis kode. Tidak   │
                    │   ada interpretasi visual, murni   │
                    │   data terstruktur presisi.        │
                    └───────────────┬───────────────────┘
                                     │
                     digenerate/disinkron ke 2 arah
                     ┌───────────────┼───────────────┐
                     v                               v
      ┌──────────────────────┐          ┌──────────────────────────┐
      │   Figma               │          │   Obsidian Vault           │
      │   (Design System v2    │          │   (06-Features/            │
      │   page)                 │          │   design-system-v2/)        │
      │                          │          │                              │
      │   PERAN: CERMIN VISUAL   │          │   PERAN: NARASI & RASIONAL   │
      │   Untuk manusia (Anda,   │          │   Kenapa keputusan diambil,   │
      │   desainer) mereview     │          │   histori audit, roadmap      │
      │   apakah token/komponen  │          │   fase, keputusan governance, │
      │   "terasa benar" secara  │          │   status migrasi per modul.   │
      │   visual. BUKAN tempat   │          │   Yang dibaca manusia untuk    │
      │   AI agent menulis kode  │          │   paham KONTEKS, bukan untuk  │
      │   dari sini.             │          │   AI agent mengeksekusi kode.  │
      └──────────────────────────┘          └──────────────────────────────┘
```

### Aturan Pembagian Konten (supaya tidak duplikat/tumpang tindih)

| Jenis Konten | Lokasi | Alasan |
|---|---|---|
| Nilai token mentah (hex, px, ms) | Repo JSON | Presisi mesin, versioned, diffable via git |
| Alias semantik (color.brand.primary, dst) | Repo JSON | Sama -- ini yang dibaca AI agent |
| Aturan wajib untuk AI agent (jangan hardcode, dst) | Repo README.md | Harus hidup dekat kode yang dibaca agent saat kerja di repo |
| Preview visual token (swatch warna, spesimen tipografi) | Figma | Verifikasi visual manusia, tidak perlu presisi mesin |
| Komponen UI (Button, Table, dst) dengan variant/state | Figma | Representasi visual untuk review desain |
| Kenapa warna X dipilih, histori perubahan | Obsidian | Konteks naratif, tidak perlu di kode/Figma |
| Roadmap fase, status tiap fase | Obsidian | Tracking proyek, bukan artefak teknis |
| Keputusan governance (escape hatch, urutan migrasi) | Obsidian | Kebijakan tim, bukan data teknis |
| Hasil audit (temuan bug, kontras gagal) | Obsidian | Dokumentasi historis/evaluatif |

## Roadmap 7 Fase

| Fase | Nama | Status | Dokumen |
|---|---|---|---|
| 0 | Discovery & Alignment | Selesai | 01-fase-0-discovery.md |
| 1 | Token Architecture | Selesai (repo + Figma + Obsidian, 3 lapis lengkap) | 02-fase-1-token-architecture.md |
| 2 | Foundation Rules | Selesai (repo + Figma + Obsidian) | 03-fase-2-foundation-rules.md |
| 3 | Component API Contracts | Selesai (repo + Obsidian; Figma menyusul di Fase 5) | 04-fase-3-component-contracts.md |
| 4 | Data-Dense Pattern Library | Selesai (repo + Obsidian; Figma menyusul di Fase 5) | 05-fase-4-data-dense-patterns.md |
| 5 | Figma Build | Selesai (14/14 komponen, semua terverifikasi visual) | 06-fase-5-figma-build.md |
| 6 | Governance & Migration | Selesai (linter v2, escape hatch, migration checklist, DESIGN_SYSTEM_AGENT.md) | 07-fase-6-governance-migration.md |

**STATUS ROADMAP: 7/7 FASE SELESAI.** Design System v2 AbsenSI sekarang punya fondasi lengkap (token, aturan, kontrak, representasi Figma, linter, proses governance) siap dipakai sebagai *single source of truth* AI coding agent. Eksekusi migrasi modul sesungguhnya adalah keputusan terpisah yang menunggu arahan product owner (lihat 07-fase-6-governance-migration.md section 4).

## Lokasi Artefak per Lapis

- Repo (source of truth): C:\ProjekSMK\AbsenSI\packages\design-tokens\src\*.tokens.json
- Figma (cermin visual): File "AbsenSI" > Page "Design System v2" (terpisah dari page "Design System" v1 lama)
- Obsidian (narasi): C:\Brain\Brain_Projek_SMK\Projek\AbsenSI\06-Features\design-system-v2\*.md

## Relasi dengan Design System v1 (lama)

Dokumen v1 di 06-Features/design-system/*.md dan Figma page "Design System" (v1) TIDAK dihapus -- tetap jadi referensi historis dan tetap berlaku untuk kode yang belum dimigrasi (aturan incremental rollout). Begitu modul selesai dimigrasi ke v2, halaman itu dicatat di 07-fase-6-governance-migration.md.
