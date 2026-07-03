---
tags:
  - task
  - bugfix
  - frontend
created: 2026-06-06
status: ready
---

# Task-010: Fix Login Routing Panitia — Keliling Tidak Diarahkan ke Halaman Keliling

## Objective
Perbaiki discriminator login di `frontend/src/App.jsx` agar panitia (termasuk keliling) diarahkan ke halaman yang benar, bukan ke halaman pengawas.

## Context
- **File:** `frontend/src/App.jsx` — satu-satunya file yang diubah
- **Jangan sentuh:** Backend, Keliling.jsx, komponen lain
- **Jangan refactor** di luar 3 baris yang dispesifikasikan

---

## Root Cause

`Panitia` model memiliki `HasApiTokens`. Akibatnya backend `/login-niy` mengembalikan **token untuk SEMUA user** (baris 227 `PresensiService`):

```php
'token' => $user->createToken($role . '-session')->plainTextToken,
// $user bisa Pengawas ATAU Panitia — keduanya dapat token
```

Frontend memakai **ada/tidaknya `token`** sebagai discriminator:

```jsx
// App.jsx baris 110 — SALAH
if (res.data.token) {
    setUserType('pengawas');  // ← panitia juga masuk sini karena juga dapat token!
} else if (res.data.user?.type === 'panitia' || res.data.type === 'panitia') {
    // ← tidak pernah tercapai
    if (res.data.user?.is_keliling) setUserType('keliling');  // ← tidak pernah jalan
}
```

Padahal backend sudah mengembalikan `role: 'panitia'` atau `role: 'pengawas'` (dari `$role = $panitia ? 'panitia' : 'pengawas'` di PresensiService baris 218). **Gunakan `res.data.role` bukan keberadaan `token`.**

---

## Fix

Di `frontend/src/App.jsx`, ubah kondisi discriminator login (sekitar baris 110–127):

```jsx
// SEBELUM — SALAH
if (res.data.token) {
    // === PENGAWAS LOGIN ===
    localStorage.setItem('pengawas_token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUserType('pengawas');
    setFormData(prev => ({ ...prev, pengawas_id: res.data.user.id, panitia_id: '' }));
} else if (res.data.user?.type === 'panitia' || res.data.type === 'panitia') {
    // === PANITIA LOGIN ===
    const panitiaData = { ...res.data.user, userType: 'panitia' };
    localStorage.setItem('panitia_session', JSON.stringify(panitiaData));
    if (res.data.user?.is_keliling) {
        setUserType('keliling');
    } else {
        setUserType('panitia');
    }
    setFormData(prev => ({ ...prev, panitia_id: res.data.user.id, pengawas_id: '' }));
}

// SESUDAH — BENAR
if (res.data.role === 'panitia') {
    // === PANITIA LOGIN ===
    const panitiaData = { ...res.data.user, userType: 'panitia' };
    localStorage.setItem('panitia_session', JSON.stringify(panitiaData));
    if (res.data.user?.is_keliling) {
        setUserType('keliling');
    } else {
        setUserType('panitia');
    }
    setFormData(prev => ({ ...prev, panitia_id: res.data.user.id, pengawas_id: '' }));
} else {
    // === PENGAWAS LOGIN ===
    localStorage.setItem('pengawas_token', res.data.token);
    axios.defaults.headers.common['Authorization'] = `Bearer ${res.data.token}`;
    setUserType('pengawas');
    setFormData(prev => ({ ...prev, pengawas_id: res.data.user.id, panitia_id: '' }));
}
```

**Yang berubah:**
1. Kondisi utama: `if (res.data.token)` → `if (res.data.role === 'panitia')`
2. Urutan dibalik: panitia dicek dulu, pengawas jadi `else`
3. Isi kedua branch tidak berubah sama sekali

---

## Files yang Diubah

| File | Perubahan |
|------|-----------|
| `frontend/src/App.jsx` | Ubah discriminator login: `token` → `role === 'panitia'` |

**Tidak ada file lain yang perlu diubah.**

---

## Acceptance Criteria

- [ ] Scan NIY panitia biasa (`can_scan = true`, `is_keliling = false`, `is_pds = false`) → masuk halaman scan panitia (bukan pengawas)
- [ ] Scan NIY panitia keliling (`is_keliling = true`) → masuk halaman `/keliling` (bukan pengawas, bukan scan panitia)
- [ ] Scan NIY panitia PDS (`is_pds = true`) → masuk halaman scan panitia (sama seperti panitia biasa, tapi dengan modal keterangan saat scan siswa)
- [ ] Scan NIY pengawas → masuk halaman pengawas (tidak berubah)
- [ ] Setelah logout dan scan ulang → routing masih benar
- [ ] Session restore dari localStorage (`panitia_session`) → masih benar (pakai kode baris 62-73, tidak diubah task ini)
- [ ] Update CONTEXT.md setelah selesai

## Validasi Claudian
- [x] Root cause verified: `Panitia` model punya `HasApiTokens` → createToken berhasil → `res.data.token` selalu truthy
- [x] Fix verified: backend sudah mengembalikan `role: 'panitia'` (PresensiService baris 218 + 225) — tidak perlu ubah backend
- [x] Session restore (`panitia_session` di localStorage) sudah benar untuk keliling (baris 62-66 App.jsx) — tidak perlu diubah
- [x] Scope: 1 file, 1 kondisi if-else, konten kedua branch tidak berubah
