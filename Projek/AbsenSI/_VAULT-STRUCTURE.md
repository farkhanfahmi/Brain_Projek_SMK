---
title: Vault Structure Map
tags: [map, structure, navigation]
date: 2026-06-26
---

# 🗺️ Peta Struktur Vault AbsenSI

## 📁 Direktori Fisik

```
Brain_Projek_SMK/
├── .obsidian/               (Obsidian config)
├── .claude/                 (Claude Code config)
├── .claudian/               (Claudian AI notes)
├── .git/                    (Git repository)
└── Projek/
    └── AbsenSI/             ← MAIN PROJECT FOLDER
        ├── 00-INDEX.md          (⭐ Start here)
        ├── 01-Overview.md       
        ├── 02-Tech-Stack.md     
        ├── 03-User-Roles.md     
        ├── 04-Database-Schema.md
        ├── 05-API-Endpoints.md  
        ├── 07-User-Flows.md     
        ├── 08-UI-UX-Guidelines.md
        ├── 09-Conventions.md    
        ├── 10-Environment.md    
        ├── 11-Decisions.md      (ADR — Architecture Decision Records)
        ├── 12-Status.md         (Project status board)
        ├── 13-Backlog.md        
        ├── 14-Debug-Log.md      
        ├── 15-Deployment-Guide.md
        ├── _REPAIR-LOG.md       (Perbaikan vault)
        ├── _VAULT-STRUCTURE.md  (Navigation map)
        │
        ├── 06-Features/         (Feature specifications)
        │   ├── _template.md     (Template for new features)
        │   ├── absensi-gerbang.md
        │   ├── absensi-kelas-mapel.md
        │   ├── akun-guru.md
        │   ├── dashboard-tv.md
        │   ├── import-data-master.md
        │   ├── manajemen-kartu.md
        │   ├── notifikasi-ortu.md
        │   └── tasks/
        │       └── _task-template.md
        │
        └── _claudian/          (Team collaboration files)
            ├── discussion-log.md
            ├── project-context.md
            ├── team.md
            └── workflow-multi-dev.md
```

---

## 🎯 File Navigation

### 📘 START HERE (Untuk pembaca baru)
1. [[Projek/AbsenSI/00-INDEX|00-INDEX.md]] — Overview semua file
2. [[Projek/AbsenSI/01-Overview|01-Overview.md]] — Background & vision
3. [[Projek/AbsenSI/_claudian/project-context|project-context.md]] — Quick context

### 👨‍💻 UNTUK DEVELOPER
1. [[Projek/AbsenSI/09-Conventions|09-Conventions.md]] — Coding standards
2. [[Projek/AbsenSI/_claudian/team|team.md]] — Module assignment
3. [[Projek/AbsenSI/_claudian/workflow-multi-dev|workflow-multi-dev.md]] — Workflow guide

### 🏗️ UNTUK ARCHITECT
1. [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack.md]] — Tech choices
2. [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema.md]] — Data model
3. [[Projek/AbsenSI/11-Decisions|11-Decisions.md]] — All ADR decisions

### 📈 UNTUK PM
1. [[Projek/AbsenSI/01-Overview|01-Overview.md]] — Vision & scope
2. [[Projek/AbsenSI/12-Status|12-Status.md]] — Project board
3. [[Projek/AbsenSI/13-Backlog|13-Backlog.md]] — Roadmap & phases

---

## 🔗 Link Format Reference

### Standard Wiki-Link Format
```markdown
[[Projek/AbsenSI/path/to/file|Display Text]]
```

### Examples
```markdown
← [[Projek/AbsenSI/00-INDEX|Index]]
[[Projek/AbsenSI/06-Features/absensi-gerbang|Absensi Gerbang]]
[[Projek/AbsenSI/_claudian/team|team.md]]
```

---

**Generated:** 2026-06-26  
**Vault Status:** ✅ Healthy
