# Task-INFRA-001: Setup SSH Read-Only untuk Hermes (Akses Diagnosa Production)

> Modul prefix: INFRA (infrastruktur/server, bukan kode aplikasi apps/). Dieksekusi langsung oleh user di server production via SSH — BUKAN oleh Claude Code (tidak ada kode aplikasi yang diubah), dan BUKAN oleh Hermes (Hermes dilarang eksekusi shell di production).

---

## 1. Info Eksekusi

**Rekomendasi Model:** Tidak relevan — ini task administrasi server (user manual), bukan task coding.
**Tingkat Effort:** Tidak relevan.
**Alasan pemilihan:** Task ini murni perintah shell administratif di server production, dijalankan langsung oleh user (root/sudo access), bukan tugas yang didelegasikan ke AI coding agent.

## 2. Konteks & Tujuan Utama

Hermes butuh kemampuan mendiagnosa masalah nyata (data, log, status service) di server production AbsenSI TANPA membuka celah tulis/eksekusi berbahaya — konsisten dengan aturan "Hermes hanya baca+diskusi" yang berlaku juga untuk dev/dan sekarang diperluas ke production.

**Prinsip keamanan**: user Linux terpisah, **tanpa** hak `sudo`, **tanpa** akses tulis ke direktori aplikasi (`AbsenSI-production/`), **tanpa** akses ke file `.env`/kredensial. Bisa baca log, baca status service (`systemctl status`, read-only), dan `SELECT` read-only ke MySQL.

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

### A. Buat user Linux baru, read-only
```bash
sudo useradd -m -s /bin/bash hermes-readonly
sudo passwd -l hermes-readonly   # kunci password login — hanya via SSH key
```

### B. Generate SSH keypair KHUSUS untuk Hermes (jangan reuse key lain)
Dijalankan di sisi Hermes/user meminta, key PRIVATE tidak pernah dikirim ke server:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/hermes_readonly -C "hermes-analysis-readonly" -N ""
# hermes_readonly.pub -> dikirim ke server
# hermes_readonly (private) -> disimpan di sisi Hermes/Windows, TIDAK PERNAH di server
```

### C. Pasang public key dengan restriction eksplisit di `authorized_keys`
```bash
sudo mkdir -p /home/hermes-readonly/.ssh
sudo tee /home/hermes-readonly/.ssh/authorized_keys << 'EOF'
no-agent-forwarding,no-X11-forwarding,no-port-forwarding,no-pty ssh-ed25519 AAAA...GANTI_DENGAN_PUBLIC_KEY_ASLI hermes-analysis-readonly
EOF
sudo chown -R hermes-readonly:hermes-readonly /home/hermes-readonly/.ssh
sudo chmod 700 /home/hermes-readonly/.ssh
sudo chmod 600 /home/hermes-readonly/.ssh/authorized_keys
```
**Catatan penting**: `no-pty` di atas MENCEGAH shell interaktif penuh. Kalau user ingin Hermes tetap bisa jalankan perintah spesifik read-only (`journalctl`, `systemctl status`, `mysql -e "SELECT..."`), GANTI baris di atas dengan versi `command=` restricted (lihat opsi D di bawah) — **lebih aman, direkomendasikan** dibanding shell penuh meski read-only.

### D. (Direkomendasikan) Command-restricted, bukan shell bebas
```
command="/home/hermes-readonly/allowed-commands.sh",no-agent-forwarding,no-X11-forwarding,no-port-forwarding ssh-ed25519 AAAA...GANTI hermes-analysis-readonly
```
Buat `/home/hermes-readonly/allowed-commands.sh` berisi whitelist perintah (mis. hanya izinkan `journalctl -u absensi-api -n *`, `systemctl status absensi-*`, `mysql -u hermes_readonly -e "SELECT ..."` — tolak selain itu).

### E. Batasi filesystem read-only untuk direktori aplikasi (opsional tapi disarankan)
```bash
# JANGAN tambahkan hermes-readonly ke grup yang punya akses tulis ke AbsenSI-production/
# Verifikasi: user ini TIDAK bisa baca .env (permission 600, owner beda)
sudo -u hermes-readonly cat /path/to/AbsenSI-production/apps/api/.env   # HARUS "Permission denied"
```

### F. Buat MySQL user read-only terpisah
```sql
CREATE USER 'hermes_readonly'@'localhost' IDENTIFIED BY '[password random kuat, generate baru]';
GRANT SELECT ON absensi_db.* TO 'hermes_readonly'@'localhost';
FLUSH PRIVILEGES;
```

### G. Serahkan ke Hermes
Setelah setup selesai, berikan ke Hermes: (1) IP/hostname server, (2) username `hermes-readonly`, (3) port SSH, (4) private key TERPISAH dari key production lain — Hermes akan simpan sebagai kredensial khusus sesi, bukan ditulis ke memory permanen.

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Buat:** user Linux baru, MySQL user baru, `authorized_keys` untuk user baru — TIDAK menyentuh konfigurasi user/service yang sudah ada.
- **Jangan sentuh:** `.env`, direktori `AbsenSI-production/` (permission, bukan konten), user/key SSH yang sudah dipakai untuk deploy/production existing.

**Dilarang dilakukan:**
- Jangan reuse SSH key yang sama dengan yang dipakai deploy/CI/production admin.
- Jangan beri `sudo` ke user `hermes-readonly` dalam kondisi apa pun.
- Jangan beri MySQL user ini hak selain `SELECT`.

**Skenario kegagalan yang WAJIB ditangani:**
- Private key bocor/ter-commit ke git → dampak dibatasi HANYA read-only (bukan seperti kebocoran key admin) — tapi tetap harus di-rotate (hapus public key dari `authorized_keys`, generate baru) begitu diketahui.
- Hermes mencoba perintah di luar whitelist (kalau pakai opsi D) → harus ditolak oleh `allowed-commands.sh`, bukan silently berhasil.
- User `hermes-readonly` somehow bisa baca `.env` → berarti permission salah, HARUS diperbaiki sebelum key diserahkan (lihat langkah E verifikasi).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] User `hermes-readonly` dibuat, password login terkunci (`passwd -l`), hanya bisa masuk via SSH key.
- [ ] `authorized_keys` memakai `no-pty` atau `command=` restricted (BUKAN shell bebas tanpa batasan).
- [ ] Verifikasi: `hermes-readonly` TIDAK BISA baca `.env` aplikasi (`Permission denied` dikonfirmasi).
- [ ] Verifikasi: `hermes-readonly` TIDAK BISA `sudo` apa pun.
- [ ] MySQL user `hermes_readonly` dibuat dengan HANYA `GRANT SELECT`, dikonfirmasi tidak bisa `INSERT`/`UPDATE`/`DELETE` (test 1 query tulis, harus ditolak).
- [ ] Private key SSH disimpan terpisah dari key production lain, diserahkan ke Hermes lewat kanal aman (bukan ditempel di chat biasa kalau memungkinkan).

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar — task ini murni administrasi akses, bukan perubahan aplikasi
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: tidak ada
