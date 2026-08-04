---
title: Vault Structure Map
tags: [map, structure, navigation]
date: 2026-07-21
---

# 🗺️ Peta Struktur Vault AbsenSI

## 📁 Direktori Fisik

```
Brain_Projek_SMK/
└── Projek/
    └── AbsenSI/                          ← MAIN PROJECT FOLDER
        ├── 00-INDEX AbsenSI.md               (⭐ Start here — satu-satunya file dengan suffix " AbsenSI")
        ├── 01-Overview.md
        ├── 02-Tech-Stack.md
        ├── 03-User-Roles.md
        ├── 04-Database-Schema.md            (mencakup entitas Fase 1 + Piket Schedule T032)
        ├── 05-API-Endpoints.md              (draft awal, belum sinkron detail dengan kode)
        ├── 07-User-Flows.md                 (draft awal)
        ├── 08-UI-UX-Guidelines.md            (draft awal; lihat 06-Features/design-system/ untuk brief final)
        ├── 09-Conventions.md                (draft awal)
        ├── 10-Environment.md                (draft awal)
        ├── 11-Decisions.md                  (ADR-001 s/d ADR-025, terpelihara aktif)
        ├── 12-Status.md                     (ringkasan progres per modul, terkini)
        ├── 13-Backlog.md                    (roadmap Fase 1 selesai + Fase 2/3 rencana)
        ├── 14-Debug-Log.md                  (belum dipakai aktif — bug dicatat inline di TASKS-*.md)
        ├── 15-Deployment-Guide.md            (draft awal, belum ada deployment nyata)
        ├── Catatan Spontan.md                (catatan bebas non-terstruktur)
        ├── TASKS-FASE-1.md                  (32 task, 31/32 selesai — T001-T028e)
        ├── TASKS-POLISH-1.md                (8 task, semua selesai — P001-P008)
        ├── TASKS-POLISH-2.md                (9 task, 8/9 selesai — T029-T037, T035 dilewati)
        ├── _REPAIR-LOG.md                   (riwayat perbaikan struktur/link vault)
        ├── _VAULT-STRUCTURE.md              (file ini)
        │
        ├── 06-Features/                     (spesifikasi fitur — sebagian besar draft pra-coding)
        │   ├── _template.md
        │   ├── absensi-gerbang.md
        │   ├── absensi-kelas-mapel.md        (Fase 2, belum dikerjakan)
        │   ├── akun-guru.md
        │   ├── dashboard-piket.md
        │   ├── dashboard-tv.md
        │   ├── import-data-master.md
        │   ├── kalender-pendidikan.md
        │   ├── manajemen-kartu.md
        │   ├── notifikasi-ortu.md            (Fase 3, belum dikerjakan)
        │   ├── rekap-kehadiran.md
        │   ├── design-system/                (brief visual "EzMart" — dipindah dari session Claude/, 2026-07-21)
        │   │   ├── MASTER.md                 (source of truth, baca duluan)
        │   │   ├── 01-colors.md
        │   │   ├── 02-typography.md
        │   │   ├── 03-components.md
        │   │   ├── 04-layout-spacing.md
        │   │   └── 05-charts-data-viz.md
        │   └── tasks/                        (task breakdown T001-T028, Fase 1)
        │       ├── _task-template.md
        │       └── T001 ... T028-profil-lengkap-foto-kiosk-scan.md
        │
        └── _claudian/                        (rencana kerja tim 3 developer — belum tentu dipakai persis, lihat 00-INDEX)
            ├── discussion-log.md
            ├── project-context.md
            ├── team.md
            └── workflow-multi-dev.md
```

---

## 🎯 File Navigation

### 📘 START HERE (Untuk pembaca baru)
1. [[Projek/AbsenSI/00-INDEX AbsenSI|00-INDEX AbsenSI.md]] — Overview semua file, status terkini
2. [[Projek/AbsenSI/01-Overview|01-Overview.md]] — Background & vision
3. [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2.md]] — Task terakhir yang dikerjakan (paling detail & terkini)

### 👨‍💻 UNTUK DEVELOPER
1. [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema.md]] — Skema data terkini
2. [[Projek/AbsenSI/11-Decisions|11-Decisions.md]] — Semua ADR (keputusan arsitektur mengikat)
3. Kode aktual di `/home/anunnaki/Documents/APP SMK/AbsenSI` — sumber kebenaran final untuk detail implementasi

### 🏗️ UNTUK ARCHITECT
1. [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack.md]] — Tech choices
2. [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema.md]] — Data model
3. [[Projek/AbsenSI/11-Decisions|11-Decisions.md]] — All ADR decisions

### 📈 UNTUK PM / STATUS CEK CEPAT
1. [[Projek/AbsenSI/12-Status|12-Status.md]] — Status per modul, terkini
2. [[Projek/AbsenSI/13-Backlog|13-Backlog.md]] — Roadmap & fase
3. [[Projek/AbsenSI/TASKS-FASE-1|TASKS-FASE-1.md]], [[Projek/AbsenSI/TASKS-POLISH-1|TASKS-POLISH-1.md]], [[Projek/AbsenSI/TASKS-POLISH-2|TASKS-POLISH-2.md]] — detail task-by-task

---

## ⚠️ Catatan Penting Soal Akurasi Dokumen

Banyak dokumen inti (05, 07, 08, 09, 10, semua `06-Features/*.md` kecuali `design-system/`) masih berupa **draft dari fase perencanaan awal** (sebelum coding dimulai) dan belum ditulis ulang detail per baris untuk mencerminkan implementasi aktual. Untuk detail teknis terkini, selalu utamakan urutan ini:
1. **Kode aktual** (`/home/anunnaki/Documents/APP SMK/AbsenSI`)
2. **TASKS-FASE-1.md / TASKS-POLISH-1.md / TASKS-POLISH-2.md** — catatan implementasi per task, paling akurat & terkini
3. **11-Decisions.md (ADR)** — keputusan arsitektur mengikat, terpelihara aktif
4. Dokumen 01-10 & `06-Features/*.md` — konteks desain awal, bisa berbeda detail dari hasil akhir

## 🔗 Link Format Reference

### Standard Wiki-Link Format
```markdown
[[Projek/AbsenSI/path/to/file|Display Text]]
```

### Contoh Nyata
```markdown
← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]
[[Projek/AbsenSI/06-Features/dashboard-piket|Dashboard Piket]]
[[Projek/AbsenSI/06-Features/design-system/MASTER|Design System]]
```

**Perhatian:** hampir semua file di vault ini **TIDAK** punya suffix " AbsenSI" pada nama filenya — kecuali `00-INDEX AbsenSI.md` sendiri. Target wikilink harus persis sama dengan nama file (tanpa ekstensi `.md`), termasuk spasi jika ada.

---

**Terakhir diperbarui:** 2026-07-21
**Vault Status:** ✅ Sehat — broken link `00-INDEX` (33 file) diperbaiki massal, folder `session Claude/` dipindah ke `06-Features/design-system/`, lihat [[Projek/AbsenSI/_REPAIR-LOG|_REPAIR-LOG.md]] untuk detail.
