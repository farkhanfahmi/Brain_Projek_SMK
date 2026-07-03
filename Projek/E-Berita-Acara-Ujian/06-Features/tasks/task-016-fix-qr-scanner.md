---
tags:
  - task
  - bugfix
  - frontend
created: 2026-06-07
status: ready
---

# Task-016: Fix QRScanner — Layar Hitam + Camera Leak + No Feedback

## Objective
Rewrite `frontend/src/components/QRScanner.jsx` untuk memperbaiki 3 bug.
Tidak perlu ganti library (`html5-qrcode` tetap digunakan).
Tidak ada perubahan backend atau file lain.

---

## Bug yang Diperbaiki

### Bug 1 — Stream tidak di-release saat `isScanning = false`
Cleanup hanya berjalan jika `isScanning = true`. Tapi saat kamera sedang inisialisasi
(permission dialog muncul) `isScanning` masih `false`. Unmount di fase ini → stream bocor
→ buka kembali → kamera sudah "dipakai" → layar hitam.

### Bug 2 — `handleFileUpload` membuat instance baru pada elemen `#reader`
`new Html5Qrcode("reader")` kedua menghancurkan referensi instance pertama → stream lama
tidak bisa di-stop → leak permanen sampai clear history.

### Bug 3 — Tidak ada feedback ke user saat kamera gagal
User melihat layar hitam. Tidak ada pesan error, tidak ada tombol retry.

---

## Implementasi Baru

Ganti SELURUH isi `QRScanner.jsx` dengan implementasi berikut:

```jsx
import { useEffect, useRef, useState } from 'react';
import { Html5Qrcode } from 'html5-qrcode';

/**
 * Force-stop semua video track yang terkait elemen #reader.
 * Dipanggil SELALU saat cleanup, sebagai safety net di atas html5-qrcode.
 */
function forceStopVideoTracks() {
    try {
        const videos = document.querySelectorAll('#reader video');
        videos.forEach(v => {
            if (v.srcObject) {
                v.srcObject.getTracks().forEach(t => t.stop());
                v.srcObject = null;
            }
        });
    } catch (_) {}
}

const QRScanner = ({ onScan, onClose }) => {
    const scannerRef   = useRef(null);
    const onScanRef    = useRef(onScan);
    const mountedRef   = useRef(true);  // track apakah komponen masih mounted
    const [camError, setCamError]     = useState(null);   // pesan error kamera
    const [isRestarting, setIsRestarting] = useState(false);

    // Keep callback ref in sync
    useEffect(() => { onScanRef.current = onScan; }, [onScan]);

    // ─── Cleanup helper ───────────────────────────────────────────────────────
    const stopScanner = async () => {
        const scanner = scannerRef.current;
        if (!scanner) return;
        try {
            if (scanner.isScanning) await scanner.stop();
        } catch (_) {}
        try {
            scanner.clear();
        } catch (_) {}
        forceStopVideoTracks(); // selalu force-stop sebagai safety net
        scannerRef.current = null;
    };

    // ─── Start scanner ────────────────────────────────────────────────────────
    const startScanner = async () => {
        if (!mountedRef.current) return;
        if (!document.getElementById('reader')) return;

        // Bersihkan instance lama jika ada
        await stopScanner();

        setCamError(null);
        setIsRestarting(false);

        const html5QrCode = new Html5Qrcode('reader');
        scannerRef.current = html5QrCode;

        try {
            await html5QrCode.start(
                { facingMode: 'environment' },
                { fps: 10 },
                (decodedText) => {
                    if (!mountedRef.current) return;
                    if (html5QrCode.isScanning) {
                        html5QrCode.stop()
                            .catch(() => {})
                            .finally(() => {
                                forceStopVideoTracks();
                                if (mountedRef.current) onScanRef.current(decodedText);
                            });
                    }
                }
            );
        } catch (err) {
            console.error('Camera start failed:', err);
            forceStopVideoTracks(); // cleanup stream yang mungkin sudah di-acquire
            if (!mountedRef.current) return;

            // Pesan error yang informatif
            if (err?.toString?.().includes('Permission')) {
                setCamError('Izin kamera ditolak. Buka pengaturan browser dan izinkan akses kamera.');
            } else if (err?.toString?.().includes('NotFound') || err?.toString?.().includes('DevicesNotFound')) {
                setCamError('Kamera tidak ditemukan. Pastikan kamera tersedia dan tidak digunakan aplikasi lain.');
            } else {
                setCamError('Kamera tidak dapat diakses. Coba tekan "Muat Ulang Kamera" di bawah.');
            }
        }
    };

    // ─── Mount / Unmount ─────────────────────────────────────────────────────
    useEffect(() => {
        mountedRef.current = true;
        startScanner();

        return () => {
            mountedRef.current = false;
            stopScanner(); // cleanup SELALU dipanggil, tidak bergantung isScanning
        };
    }, []); // eslint-disable-line

    // ─── Restart handler ──────────────────────────────────────────────────────
    const handleRestart = async () => {
        setIsRestarting(true);
        await stopScanner();
        // Beri jeda 500ms agar browser benar-benar melepas kamera
        setTimeout(() => {
            if (mountedRef.current) startScanner();
        }, 500);
    };

    // ─── File upload ─────────────────────────────────────────────────────────
    // Gunakan instance terpisah (bukan #reader) untuk scan file agar tidak konflik
    const handleFileUpload = async (e) => {
        const file = e.target.files[0];
        if (!file) return;

        // Pakai elemen reader-file-scan yang terpisah dari kamera
        const tempId = 'reader-file-scan';
        let tempDiv = document.getElementById(tempId);
        if (!tempDiv) {
            tempDiv = document.createElement('div');
            tempDiv.id = tempId;
            tempDiv.style.display = 'none';
            document.body.appendChild(tempDiv);
        }

        const fileScanner = new Html5Qrcode(tempId);
        try {
            const decoded = await fileScanner.scanFile(file, true);
            onScanRef.current(decoded);
        } catch (_) {
            alert('Gagal membaca QR Code dari file. Pastikan gambar jelas dan tidak buram.');
        } finally {
            try { fileScanner.clear(); } catch (_) {}
            tempDiv.remove();
        }
    };

    // ─── Render ───────────────────────────────────────────────────────────────
    return (
        <div className="fixed inset-0 bg-black/90 flex items-center justify-center z-[100] backdrop-blur-sm p-4">
            <div className="bg-white rounded-3xl w-full max-w-md shadow-2xl overflow-hidden border border-slate-200">
                {/* Header */}
                <div className="bg-slate-50 p-4 border-b border-slate-100 flex justify-between items-center px-6">
                    <h3 className="font-bold text-slate-800 flex items-center gap-2">
                        <svg className="w-5 h-5 text-indigo-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2"
                                d="M12 4v1m6 11h2m-6 0h-2v4m0-11v3m0 0h.01M12 12h4.01M16 20h4M4 12h4m12 0h.01M5 8h2a1 1 0 001-1V5a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1zm12 0h2a1 1 0 001-1V5a1 1 0 00-1-1h-2a1 1 0 00-1 1v2a1 1 0 001 1zM5 20h2a1 1 0 001-1v-2a1 1 0 00-1-1H5a1 1 0 00-1 1v2a1 1 0 001 1z" />
                        </svg>
                        Scan QR Code
                    </h3>
                    <button onClick={onClose} className="p-1 hover:bg-slate-200 rounded-lg transition-colors text-slate-400">
                        <svg className="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M6 18L18 6M6 6l12 12" />
                        </svg>
                    </button>
                </div>

                <div className="p-6 space-y-4">
                    {/* Viewfinder */}
                    <div className="relative aspect-square w-full rounded-2xl overflow-hidden bg-slate-900 shadow-inner">
                        <div id="reader" className="w-full h-full" />

                        {/* Error overlay */}
                        {camError && (
                            <div className="absolute inset-0 bg-slate-900/95 flex flex-col items-center justify-center p-6 text-center gap-3">
                                <svg className="w-10 h-10 text-red-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2"
                                        d="M15 10l4.553-2.069A1 1 0 0121 8.87v6.26a1 1 0 01-1.447.894L15 14M3 8a2 2 0 012-2h10a2 2 0 012 2v8a2 2 0 01-2 2H5a2 2 0 01-2-2V8z" />
                                </svg>
                                <p className="text-white text-sm font-medium">{camError}</p>
                            </div>
                        )}

                        {/* Scan overlay (hanya jika tidak error) */}
                        {!camError && (
                            <div className="absolute inset-0 border-2 border-indigo-500/30 pointer-events-none">
                                <div className="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 w-48 h-48 border-2 border-indigo-500 rounded-2xl">
                                    <div className="absolute -top-1 -left-1 w-4 h-4 border-t-4 border-l-4 border-indigo-400 rounded-tl-md" />
                                    <div className="absolute -top-1 -right-1 w-4 h-4 border-t-4 border-r-4 border-indigo-400 rounded-tr-md" />
                                    <div className="absolute -bottom-1 -left-1 w-4 h-4 border-b-4 border-l-4 border-indigo-400 rounded-bl-md" />
                                    <div className="absolute -bottom-1 -right-1 w-4 h-4 border-b-4 border-r-4 border-indigo-400 rounded-br-md" />
                                </div>
                                <div className="absolute top-0 left-0 w-full h-1 bg-gradient-to-r from-transparent via-indigo-400 to-transparent animate-scan-line" />
                            </div>
                        )}
                    </div>

                    {/* Tombol-tombol */}
                    <div className="flex flex-col gap-3">
                        {/* Muat Ulang Kamera — selalu tampil sebagai safety net */}
                        <button
                            onClick={handleRestart}
                            disabled={isRestarting}
                            className="w-full py-3 bg-amber-50 text-amber-700 font-bold rounded-xl border-2 border-amber-100 hover:bg-amber-100 transition flex items-center justify-center gap-2 disabled:opacity-50"
                        >
                            <svg className={`w-5 h-5 ${isRestarting ? 'animate-spin' : ''}`} fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2"
                                    d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
                            </svg>
                            {isRestarting ? 'Memuat ulang...' : '🔄 Kamera gelap? Muat Ulang'}
                        </button>

                        {/* Upload dari file */}
                        <input
                            type="file"
                            id="qr-upload"
                            accept="image/*"
                            className="hidden"
                            onChange={handleFileUpload}
                        />
                        <button
                            onClick={() => document.getElementById('qr-upload').click()}
                            className="w-full py-3 bg-indigo-50 text-indigo-700 font-bold rounded-xl border-2 border-indigo-100 hover:bg-indigo-100 transition flex items-center justify-center gap-2"
                        >
                            <svg className="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2"
                                    d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12" />
                            </svg>
                            Pilih Gambar QR dari File
                        </button>

                        <button
                            onClick={onClose}
                            className="w-full py-3 bg-slate-900 text-white font-bold rounded-xl shadow-md hover:bg-slate-800 transition active:scale-[0.98]"
                        >
                            Tutup Kamera
                        </button>
                    </div>

                    {/* Tips */}
                    <div className="bg-amber-50 rounded-xl p-4 border border-amber-100">
                        <div className="flex gap-2">
                            <svg className="h-5 w-5 text-amber-400 flex-shrink-0 mt-0.5" fill="currentColor" viewBox="0 0 20 20">
                                <path fillRule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z" clipRule="evenodd" />
                            </svg>
                            <div className="text-xs text-amber-800 space-y-1">
                                <p className="font-bold">Tips:</p>
                                <ul className="list-disc pl-4 space-y-0.5 opacity-80">
                                    <li>Jika kamera gelap, tekan "Muat Ulang" di atas.</li>
                                    <li>Jika tetap tidak bisa, gunakan tombol "Pilih Gambar" sebagai alternatif.</li>
                                    <li>Pastikan izin kamera sudah diaktifkan di browser.</li>
                                </ul>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    );
};

export default QRScanner;
```

---

## Yang Berubah dari Versi Lama

| Aspek | Sebelum | Sesudah |
|---|---|---|
| Cleanup on unmount | Hanya jika `isScanning = true` | **Selalu**, + force-stop video tracks |
| File scan | Instance baru di `#reader` (konflik) | Instance terpisah di elemen hidden |
| Error handling | `console.error` saja | Overlay error + pesan user-friendly |
| Retry | Tidak ada | Tombol "🔄 Muat Ulang Kamera" |
| Restart delay | Tidak ada | 500ms delay agar browser release kamera |
| Mounted guard | Tidak ada | `mountedRef` cegah state update setelah unmount |

---

## File yang Diubah

| File | Aksi |
|------|------|
| `frontend/src/components/QRScanner.jsx` | REWRITE — cleanup fix + retry button + error UI |

**Tidak ada file lain yang diubah. Tidak ada backend changes.**

---

## Acceptance Criteria

- [ ] Buka scanner → scan berhasil → tutup → buka lagi → kamera langsung aktif (tidak hitam)
- [ ] Buka scanner → tutup sebelum kamera aktif → buka lagi → kamera langsung aktif
- [ ] Scanner dari dua tab berbeda → tutup satu → tab lain tetap bisa scan
- [ ] Kamera gagal → muncul pesan error (bukan layar hitam diam)
- [ ] Tombol "🔄 Muat Ulang Kamera" merestart scanner tanpa perlu close & open
- [ ] "Pilih Gambar QR" tetap berfungsi saat kamera aktif
- [ ] Update CONTEXT.md setelah selesai
