# Upgrade Speak-Hospital dari OVA/OVF Versi Lama

Panduan ini digunakan ketika Speak-Hospital dari image **OVA/OVF versi lama** tidak dapat diperbarui melalui alur normal. Proses upgrade akan mengembalikan repository konfigurasi ke versi dasar yang kompatibel, mengambil pembaruan terbaru, lalu menjalankan instalasi/update ulang.

> **Peringatan:** Perintah `git checkout .` dan `git reset --hard` akan membuang seluruh perubahan lokal pada file yang dilacak Git di `/var/www/speak-hospital/`. Selesaikan langkah persiapan sebelum melanjutkan.

## Sebelum Memulai

Siapkan akses berikut:

| Kebutuhan | Nilai |
|-----------|-------|
| SSH host | `localhost` |
| SSH port | `2222` |
| SSH user | `sa` |
| Direktori aplikasi | `/var/www/speak-hospital/` |
| Commit kompatibilitas | `270688d` |

Pastikan server dapat mengakses Git repository, Docker registry, dan DNS. Daftar jaringan yang perlu diizinkan tersedia di bagian [Prasyarat Jaringan](README.md#prasyarat-jaringan).

### Catat konfigurasi yang sedang digunakan

Sebelum upgrade, catat atau simpan informasi berikut jika server sebelumnya sudah dikonfigurasi:

- Protokol yang digunakan: **HTTP**, **HTTPS**, atau **HTTP + HTTPS**.
- Domain atau IP server.
- Port HTTP dan HTTPS.
- File sertifikat lengkap, biasanya `fullchain.pem`.
- File kunci privat, biasanya `privkey.pem`.
- Pengaturan proteksi IP/whitelist CPanel, jika pernah diaktifkan.

Pastikan kedua file sertifikat tersedia dan merupakan pasangan yang benar. Mode HTTPS memerlukan sertifikat lengkap dan kunci privat.

Jika kondisi server masih memungkinkan, buat backup dari **CPanel System → Pemeliharaan** sebelum memulai maintenance pada server production.

## Langkah Upgrade

### 1. Masuk ke server melalui SSH

Jalankan dari terminal komputer admin:

```bash
ssh -p 2222 sa@localhost
```

### 2. Masuk ke direktori Speak-Hospital

```bash
cd /var/www/speak-hospital/
```

Pastikan lokasi sudah benar:

```bash
pwd
```

Output yang diharapkan:

```text
/var/www/speak-hospital
```

### 3. Kembalikan repository ke versi yang kompatibel

Jalankan perintah berikut satu per satu:

```bash
git checkout .
git reset --hard 270688d
git pull
```

| Perintah | Fungsi |
|----------|--------|
| `git checkout .` | Membuang perubahan lokal pada file yang dilacak Git |
| `git reset --hard 270688d` | Mengembalikan repository ke commit dasar yang kompatibel |
| `git pull` | Mengambil dan menerapkan pembaruan terbaru dari branch yang sedang aktif |

Jika salah satu perintah menampilkan error, **jangan lanjutkan ke `initial -u`**. Periksa koneksi jaringan, akses repository, dan branch Git yang aktif terlebih dahulu.

### 4. Jalankan proses instalasi/update

```bash
initial -u
```

Tunggu sampai proses selesai. Jangan menutup terminal atau me-restart server selama proses masih berjalan.

`initial -u` akan menyiapkan ulang konfigurasi dan memperbarui komponen Speak-Hospital, termasuk database service, memory service, backend, frontend, web server, CPanel Center, dan CPanel System.

## Konsekuensi Setelah Upgrade

### Konfigurasi lokal dapat kembali ke default

Karena repository di-reset secara paksa, perubahan lokal pada file yang dilacak Git akan hilang. Konfigurasi web dapat kembali menggunakan **HTTP** dengan nilai bawaan. Domain, port, HTTPS, dan proteksi IP yang sebelumnya diterapkan perlu diperiksa dan mungkin dikonfigurasi ulang.

### SSL/TLS perlu disiapkan ulang

Jika server sebelumnya memakai HTTPS, lakukan setup ulang melalui **CPanel System → System → Konfigurasi**:

1. Buka CPanel System. Contoh: `http://192.168.4.158/cpanel-system`.
2. Buka menu **System → Konfigurasi**.
3. Pilih **HTTPS** atau **HTTP + HTTPS** sesuai konfigurasi sebelumnya.
4. Isi kembali domain/IP serta port HTTP dan HTTPS.
5. Pada bagian **Sertifikat SSL**, pilih:
   - **Sertifikat lengkap**: `fullchain.pem`
   - **Kunci privat**: `privkey.pem`
6. Klik **Unggah sertifikat** dan tunggu sampai muncul status **Sertifikat siap**.
7. Periksa **Pratinjau konfigurasi web**.
8. Klik **Simpan**, lalu pantau **Log penerapan** sampai selesai.
9. Refresh browser dan uji akses melalui HTTPS.

Mengunggah sertifikat hanya menyimpan file ke server. Konfigurasi baru digunakan setelah tombol **Simpan** ditekan dan proses penerapan selesai.

Panduan lengkap tersedia di [Setup Domain dan SSL/TLS](README.md#setup-domain-dan-ssltls).

### Data aplikasi

Prosedur ini ditujukan untuk memperbaiki kode dan konfigurasi instalasi lama. Perintah di atas **tidak menjalankan reset atau restore database secara langsung**.

## Struktur URL

Path yang diawali `/` merupakan lanjutan dari alamat dasar server, bukan lanjutan dari URL tenant. Contoh untuk server `192.168.4.158`:

| Halaman | URL |
|---------|-----|
| Aplikasi tenant | `http://192.168.4.158/hospitals/restuibuhospital-balikpapan/` |
| Login tenant | `http://192.168.4.158/hospitals/restuibuhospital-balikpapan/login` |
| CPanel Center | `http://192.168.4.158/cpanel` |
| CPanel System | `http://192.168.4.158/cpanel-system` |

CPanel System tidak berada di bawah path tenant. Alamat seperti `/hospitals/restuibuhospital-balikpapan/cpanel-system` tidak digunakan.

## Verifikasi Setelah Upgrade

Periksa seluruh poin berikut:

- Halaman utama `/` dapat diakses.
- CPanel Center `/cpanel` dapat dibuka dan login berhasil.
- CPanel System `/cpanel-system` dapat dibuka.
- Daftar tenant masih tersedia.
- Tenant dapat diakses melalui `/hospitals/{slug}`.
- Domain/IP dan port sesuai konfigurasi rumah sakit.
- Jika menggunakan HTTPS, sertifikat valid dan browser tidak menampilkan error.
- Jika proteksi IP sebelumnya aktif, whitelist sudah diterapkan kembali dan admin tidak terkunci dari CPanel.
- Status container dan log pada dashboard CPanel System tidak menunjukkan error.

## Pemulihan Jika HTTPS Gagal

Jika konfigurasi SSL/TLS membuat situs tidak dapat diakses, masuk kembali melalui SSH lalu jalankan:

```bash
initial -u --only-reset
```

Perintah tersebut mengembalikan konfigurasi web ke HTTP dan me-restart container tanpa menarik image baru. Setelah itu:

1. Akses aplikasi kembali menggunakan `http://...`.
2. Periksa domain, port, `fullchain.pem`, dan `privkey.pem`.
3. Konfigurasikan kembali HTTPS melalui **CPanel System → Konfigurasi**.
4. Klik **Simpan** dan pantau log sampai proses selesai.

## Referensi

- [Dokumentasi utama Speak-Hospital](README.md)
- [Dokumentasi teknis](README%20(technical).md)
- [Repository dokumentasi Speak-Hospital](https://github.com/itlamkprs/speak-hospital-documentation)
