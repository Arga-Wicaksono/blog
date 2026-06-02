+++
title = "Menguak Bug 20 Tahun di PostgreSQL: Dari pgp_sym_decrypt Hingga RCE di ZeroDay Cloud"
date = 2026-06-02T00:00:00Z
draft = false
tags = ["postgresql", "security", "exploitation", "zeroday-cloud", "bug-hunting"]
cover = "/images/postgres-bug-cover.jpg"
+++

## Pendahuluan

Pada Desember 2025, kompetisi *ZeroDay Cloud* (ZDC) yang diselenggarakan oleh Wiz Research menyaksikan sebuah pengungkapan kerentanan yang mengejutkan dunia database: **bug dalam fungsi `pgp_sym_decrypt` di ekstensi `pgcrypto` PostgreSQL** yang sudah ada sejak lebih dari 20 tahun lalu. Dua peneliti berhasil mengeksploitasi kerentanan ini untuk mendapatkan *remote code execution* (RCE) dari akun pengguna database dengan hak terbatas.

Artikel ini merangkum cerita lengkap di balik penemuan, eksploitasi, dan dampak bug ini.

## Latar Belakang: ZeroDay Cloud dan PostgreSQL

### Apa itu ZeroDay Cloud?

ZeroDay Cloud adalah kompetisi *pwn* yang mirip Pwn2Own, namun khusus untuk target perangkat lunak cloud. Peserta dapat memilih beberapa target yang diumumkan dua bulan sebelumnya, dan jika berhasil mengeksploitasi target itu di atas panggung, mereka mendapatkan imbalan finansial.

### Kenapa PostgreSQL?

PostgreSQL adalah salah satu sistem database paling populer di dunia, ditulis dalam C dengan commit terdokumentasi sejak 1996. Arsitekturnya menggunakan *multi-process*: setiap koneksi klien dilayani oleh proses backend terpisah yang mengeksekusi SQL.

- **Pre-auth attack surface**: Sangat sempit — hanya perintah autentikasi yang bisa dicapai.
- **Post-auth attack surface**: Luas banget. Pengguna biasa bisa membuat tabel, menulis, membaca, dan — yang paling penting — memuat ekstensi.

Dalam skenario ZDC, peneliti mendapatkan akun pengguna **low-privileged default** tanpa hak khusus.

## Metodologi: Membaca Kode, Bukan Fuzzing

Temuan ini tidak ditemukan dengan fuzzer canggih atau AI. Malahan, itu cukup **membaca kode sumber PostgreSQL** — metode yang sederhana tapi jarang dipakai.

> "We discovered the bug with our own eyes and not with AI."

Kunci temuan adalah ekstensi **`pgcrypto`**. PostgreSQL memiliki 26 ekstensi *terpercaya* (*trusted*) yang bisa dimuat oleh pengguna biasa tanpa hak superuser. Ekstensi ini menjadi *attack surface* emas.

### Ekstensi `pgcrypto`

Ekstensi ini menyediakan fungsi kriptografi: AES, hash, *random source*, dan — yang menjadi titik masalah — **implementasi PGP sendiri** di dalam database.

Fungsi `pgp_sym_decrypt` bertugas mendekripsi *ciphertext* PGP dengan kunci simetris. Dan di sinilah letak bug-nya.

## Memahami Bug `pgp_sym_decrypt`

### PGP Format dan Format Byte

Format PGP memiliki *header* sebelum data sebenarnya. Di header ada **format byte** yang menjelaskan bagaimana plaintext harus ditampilkan:

- `b` = binary
- `u` = Unicode (UTF-8)
- `t` = ASCII text

Dalam implementasi yang buggy, setelah dekripsi, PostgreSQL mengecek **Unicode flag** dari PGP packet. **Jika flag-nya `u` (Unicode)**, maka hasil dekripsi dikonversi kembali ke UTF-8. **Jika bukan `u`**, hasil dekripsi dianggap sudah valid UTF-8 tanpa dicek.

### Masalahnya Apa?

1. Peneliti bisa mengirim *ciphertext* buatan sendiri dengan format byte = `u` (Unicode) yang sebenarnya berisi **byte arbitrer** (misal byte `0xF0` yang merupakan invalid UTF-8).
2. Karena flag Unicode aktif, PostgreSQL mencoba konversi dari UTF-8 — tapi data palsu lolos karena memang melewati *decryption* terlebih dahulu.
3. Akibatnya: **byte arbitrer bisa masuk ke database sebagai objek teks**, melanggar aturan *database encoding* yang seharusnya hanya menerima UTF-8 valid.

Fungsi inilah yang disebut **"dirty string"** — string yang berisi byte sewenang-wenang yang bisa "mencuri" masuk lewat celah konversi encoding.

## Membangun Eksploitasi: Dari Primitif ke RCE

Dengan "dirty string" di tangan, langkah selanjutnya adalah mencari fungsi-fungsi lain yang **memiliki asumsi invalid tentang panjang string UTF-8**.

### Langkah 1: `pg_utf_mblen` dan Discrepansi Panjang

PostgreSQL memiliki fungsi `pg_utf_mblen` untuk menghitung panjang karakter multi-byte UTF-8. Sayangnya, fungsi ini hanya mengandalkan **byte pertama** untuk menentukan jumlah byte karakter. Dengan "dirty string", peneliti bisa membuat fungsi ini melaporkan panjang yang **lebih besar** dari seharusnya.

```
strlen("0xF0")        = 1 byte
pg_utf_mblen("0xF0")  = 4 bytes
```

### Langkah 2: `text_reverse` dan Out-of-Bounds Write

Fungsi `text_reverse` untuk membalik string memiliki loop yang bergantung pada `pg_mblen`. Ketika string "kotor" dengan karakter seolah 4-byte tapi sebenarnya 1-byte dipakai, pointer tujuan reverse melompat **di luar buffer**, menulis ke memori yang seharusnya tidak boleh diakses.

### Langkah 3: Heap Shaping dan Arbitrary Write

PostgreSQL menggunakan AllocSet allocator yang mempartisi memori dalam *chunk* berukuran pangkat dua. Peneliti menemukan bahwa:

- Mereka bisa memaksa allocator menempatkan objek di lokasi memori yang diinginkan.
- Dengan membangun format string dan memanipulasi panjang chunk, mereka bisa menulis ke **block header** allocator.
- Dengan menulis ulang *free pointer*, mereka bisa mengalokasikan memori di alamat arbitrer — **arbitrary write primitif**.

### Langkah 4: Code Execution

Target akhir adalah mengeksekusi perintah sistem. Di PostgreSQL ada fungsi `execute_recovery_command` yang memanggil `system()` untuk menjalankan perintah shell. Namun, untuk memanggil fungsi ini perlu memanipulasi *config string struct*.

Peneliti menulis payload ke alamat kontrol, dan saat pengaturan `search_path` dimodifikasi, *hook* assign dipanggil yang pada akhirnya mengeksekusi perintah melalui `system()`. Hasilnya: **reverse shell** dari proses PostgreSQL.

## Dampak dan Implikasi Nyata

Bug ini bukan cuma teori di atas panggung. Beberapa skenario nyata:

1. **Default deployment PostgreSQL** — 99% instalasi production menggunakan akun default tanpa stripping hak CREATE. Jika akun tersebut disusupi, RCE bisa terjadi dalam hitungan detik.
2. **Managed database services** (AWS RDS, GCP Cloud SQL, dsb.) — Infrastruktur *multi-tenant* berarti satu database yang disusupi bisa menuju host OS bersama, membahayakan penyewa lain.
3. **SQL Injection** — Jika aplikasi terkena SQL injection, penyerang bisa langsung mengupgrade menjadi RCE menggunakan primitive di atas, tanpa perlu upload file atau eksploitasi lain.

## Patch dan Perbaikan

Kerentanan ini segera dilaporkan ke Wiz Research dan ditransfer ke tim maintainer PostgreSQL. Perbaikan arrived di:

- **PostgreSQL 18.2**
- **Semua versi keamanan yang masih didukung** (*backported*)

Perbaikan yang dilakukan:
1. `pgp_sym_decrypt` sekarang memverifikasi bahwa plaintext hasil dekripsi **benar-benar konform dengan database encoding** — tidak lagi mempercayai format byte dari PGP packet.
2. `pg_utf_mblen` di-*harden* dengan pengecekan batas yang lebih ketat.
3. Alokator memory tidak diperketat terlalu banyak untuk menghindari dampak performa.

## Pelajaran dari Perjalanan Eksploitasi

1. **Baca kode sumber** — Pendekatan "old school" membaca kode manual bisa mengungkap bug yang tidak akan ditemukan fuzzer modern.
2. **Ekstensi adalah attack surface baru** — Fitur yang dianggap "aman" seperti loading ekstensi trusted bisa menjadi pintu masuk.
3. **Small bug, big impact** — Satu kesalahan validasi encoding mengarah ke primitive info leak, arbitrary write, hingga RCE.
4. **Kompetisi seperti ZDC bernilai tinggi** — Target yang transparan memaksa peneliti berpikir kreatif, bukan mengandalkan *luck*.

## Referensi

- Sumber inspirasi dan rujukan teknis terkait eksploitasi ini dapat dilacak melalui materi *ZeroDay Cloud* (ZDC) oleh Wiz Research dan presentasi terkait bug `pgp_sym_decrypt` pada Offensive Con 2026.
