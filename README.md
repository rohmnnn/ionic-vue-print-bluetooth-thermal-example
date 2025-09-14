# Ionic Vue — Bluetooth Thermal Printer Example

Contoh aplikasi Ionic Vue + Capacitor untuk uji cetak ke printer thermal via:
- Classic (SPP) — `cordova-plugin-bluetooth-serial` (Android saja)
- BLE (GATT) — `cordova-plugin-ble-central` (Android & iOS)

> Untuk uji plugin, jalankan di perangkat nyata. Mode web hanya untuk melihat UI.

## Prasyarat
- Node.js LTS (18/20)
- Android Studio + Android SDK (untuk build Android)
- Perangkat Android (USB debugging aktif)
- iOS: Mac + Xcode (opsional, BLE saja)

## Instalasi
```cmd
npm install
```
Plugin Cordova (jika belum terpasang):
```cmd
REM BLE (sudah ada di package.json, jalankan jika belum)
npm i cordova-plugin-ble-central

REM Classic SPP (Android)
npm i cordova-plugin-bluetooth-serial

REM Runtime permission helper (opsional, Android)
npm i cordova-plugin-android-permissions
```
Sinkronkan ke proyek Capacitor:
```cmd
npx cap sync
```

## Jalankan di Web (UI saja)
```cmd
npm run dev
```
Buka URL yang ditampilkan oleh Vite. Fitur Bluetooth tidak berfungsi di web.

## Build & Jalankan di Android
```cmd
npx cap add android
npm run build
npx cap sync
npx cap open android
```
Lalu run dari Android Studio ke perangkat nyata. Pastikan Bluetooth aktif.

Izin Android (Android 12+): aplikasi butuh BLUETOOTH_CONNECT, BLUETOOTH_SCAN, dan lokasi (untuk scanning). Di app ada tombol "Check & Request" untuk meminta izin saat runtime.

## Penggunaan Aplikasi
1. Buka app di perangkat nyata. Pada bagian "Environment", klik "Refresh" untuk deteksi plugin.
2. Klik "Check & Request" untuk memastikan izin Bluetooth & lokasi.
3. Classic (SPP):
   - Enable → aktifkan Bluetooth klasik.
   - Paired → tampilkan perangkat yang sudah dipasangkan (pairing dilakukan di Settings OS).
   - Discover → temukan perangkat baru.
   - Connect → sambung ke printer, lalu gunakan "Print Text", "Print ESC/POS", atau "Print Formatted".
4. BLE:
   - Scan → pindai perangkat BLE, pilih lalu "Connect".
   - Service UUID & Write Char UUID sudah diisi default umum (mis. modul HM-10/serupa). Ubah jika printer Anda berbeda.
   - Gunakan "Print Text", "Print ESC/POS", atau "Print Formatted".
5. Cek bagian "Logs" untuk status dan error.

## Kustomisasi
- Ubah teks default dan resi terformat di `src/views/HomePage.vue`:
  - `printText` (teks cetak sederhana)
  - `buildFormattedReceipt(...)` (konten resi ESC/POS)
- Ubah UUID BLE di kolom input jika printer Anda memakai service/characteristic berbeda.

## Catatan
- iOS tidak mendukung Bluetooth Classic SPP; gunakan BLE.
- Beberapa printer butuh baris kosong atau perintah potong kertas (GS V) agar hasil keluar penuh.
- Jika transfer BLE gagal, coba kurangi ukuran payload (chunking) atau pastikan perangkat dekat.

## Lisensi
Contoh proyek untuk keperluan belajar/demonstrasi.
