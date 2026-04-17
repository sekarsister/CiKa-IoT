
Penjelasan Singkat Setiap Tahapan
Tahap	Langkah Kunci	Perintah/Tools
1. Persiapan Watch	Aktifkan mode developer di watch: Settings > About > Tap Build Number 7x. Lalu Developer Options > ADB Debugging dan Debug over Bluetooth.	-
2. Build APK	Di Android Studio, pilih modul wear lalu Build > Build Bundle(s) / APK > Build APK.	Android Studio
3. Koneksi Fisik	Untuk kabel USB: colokkan watch ke PC. Untuk Bluetooth: jalankan adb forward tcp:4444 localabstract:/adb-hub lalu adb connect 127.0.0.1:4444 (Wear OS 2) atau gunakan opsi "Pair new device" di Wear OS 3+.	ADB command line
4. Instalasi	Gunakan adb install -r path/to/wear.apk atau langsung Run dari Android Studio.	ADB / Android Studio
5. Debugging	Pantau log dengan adb logcat -s Wearable atau filter di Android Studio.	Logcat
6. Rilis	Buat keystore, tanda tangani AAB, unggah ke Play Console dengan kategori Wear OS.	Google Play Console
Catatan Penting untuk Deployment Wear OS
Emulator vs Fisik: Emulator bagus untuk pengembangan awal, tetapi uji coba koneksi Bluetooth/WiFi dan battery life harus dilakukan di perangkat fisik.

Ukuran APK: Pastikan ukuran sekecil mungkin karena storage watch terbatas.

Izin: Jika aplikasi watch mengakses internet langsung, tambahkan <uses-permission android:name="android.permission.INTERNET" /> di AndroidManifest.xml modul wear.

Companion App: Untuk distribusi ke Play Store, aplikasi watch harus di-upload bersama aplikasi ponsel dalam satu listing yang sama (multi-APK) atau terpisah namun ditautkan.
