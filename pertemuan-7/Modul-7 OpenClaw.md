# Modul 7: OpenClaw — Personal AI Assistant 24/7

Catatan praktik dari Kelas Otomesyen, disusun oleh Rizal Wahyu Pratama.

Dokumen ini berisi langkah instalasi yang benar-benar dijalankan di VPS, lengkap dengan masalah yang ditemui dan cara memperbaikinya. Beberapa command di modul resmi sudah tidak sesuai dengan versi OpenClaw terbaru, jadi dokumen ini juga berfungsi sebagai catatan penyesuaian.


## Infrastruktur yang dipakai

- VPS: Sumopod (Tencent Cloud), Singapore, Ubuntu 22.04 LTS, 2 vCPU / 2GB RAM
- Domain: wican.my.id (Cloudflare, plan Free)
- Reverse proxy: Cloudflare Tunnel (cloudflared)
- Sudah ada n8n berjalan di server yang sama, di subdomain n8n.wican.my.id


## Daftar isi

1. Instalasi Node.js
2. Instalasi OpenClaw
3. Menjalankan gateway sebagai service
4. Menghubungkan bot Telegram
5. Menghubungkan model AI lewat OpenRouter
6. Mengakses web dashboard secara permanen lewat domain sendiri
7. Konfigurasi identitas assistant
8. Ringkasan masalah dan solusi


## 1. Instalasi Node.js

VPS ini awalnya punya Node.js versi 12 bawaan Ubuntu, terlalu lama untuk OpenClaw.

Tambahkan repository Node.js versi 20:

```
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
```

Install Node.js:

```
sudo apt-get install -y nodejs
```

Cek versi:

```
node --version
npm --version
```

Hasil yang didapat: Node v20.20.2 dan npm 10.8.2.

Catatan: belakangan diketahui bahwa OpenClaw versi terbaru butuh Node versi 22 ke atas, bukan 18 seperti yang tertulis di modul. Node versi 22 diinstal belakangan lewat nvm, dijelaskan di bagian masalah nomor 2.


## 2. Instalasi OpenClaw

### Percobaan pertama, gagal

```
sudo npm install -g openclaw
```

Perintah ini berhasil tanpa error, tapi command `openclaw` tidak bisa dipanggil setelahnya. Penyebabnya dijelaskan di bagian ringkasan masalah nomor 1 dan 3 di bawah.

### Setup ulang yang benar

Buat folder npm global khusus milik user, supaya instalasi package global tidak perlu sudo:

```
mkdir ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

Install nvm untuk mengelola beberapa versi Node.js sekaligus:

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
```

Install Node versi 22 dan jadikan default:

```
nvm install 22
nvm use --delete-prefix v22.23.2
nvm alias default 22
```

Install OpenClaw versi terbaru secara eksplisit:

```
npm install -g openclaw@latest
```

Verifikasi:

```
openclaw --version
```

Hasil: OpenClaw 2026.7.1-2 berhasil terpasang dan bisa dipanggil.


## 3. Menjalankan gateway sebagai service

Gateway adalah service utama OpenClaw yang berjalan di background. Tugasnya mengatur koneksi ke Telegram, menjalankan tools, dan mengelola memory.

Command `openclaw gateway start` yang disebut di modul resmi sudah tidak berfungsi seperti itu lagi. Command yang benar untuk versi ini:

Daftarkan gateway sebagai systemd service:

```
openclaw gateway install
```

Jalankan service:

```
systemctl --user start openclaw-gateway.service
```

Set supaya otomatis jalan tiap ada perubahan (opsional, untuk keperluan restart manual setelah ubah config):

```
systemctl --user restart openclaw-gateway.service
```

Cek status:

```
systemctl --user status openclaw-gateway.service
openclaw gateway status
```

Status yang diharapkan: `active (running)`, dan gateway status menunjukkan alamat listening seperti `127.0.0.1:18789`.


## 4. Menghubungkan bot Telegram

Command `openclaw plugins enable telegram` yang tertulis di modul resmi tidak berfungsi lagi. Telegram sekarang dikelola lewat sistem channels, bukan plugins.

### Membuat bot di Telegram

1. Buka Telegram, cari @BotFather
2. Kirim `/newbot`
3. Ikuti wizard, beri nama dan username bot (harus diakhiri `_bot`)
4. Salin API token yang diberikan

### Menghubungkan ke OpenClaw

Jalankan wizard interaktif:

```
openclaw channels add
```

Pilih Telegram (Bot API), pilih akun default, lalu tempelkan token bot saat diminta.

Saat proses berjalan, akan muncul peringatan bahwa siapa saja yang menemukan bot bisa mengirim pairing request. Untuk assistant pribadi, sebaiknya dibatasi dengan allowlist:

- Pilih kebijakan DM: Allowlist (specific users only)
- Masukkan Telegram user ID sendiri (bisa didapat dengan chat ke bot @userinfobot)

### Verifikasi koneksi

```
openclaw channels status
```

Kalau statusnya masih `disconnected`, restart gateway dulu:

```
systemctl --user restart openclaw-gateway.service
```

Tunggu beberapa detik, cek ulang statusnya. Status yang diharapkan adalah `connected`.


## 5. Menghubungkan model AI lewat OpenRouter

Tanpa langkah ini, bot akan merespon dengan error HTTP 401 karena belum ada model AI yang terhubung.

Buat akun di https://openrouter.ai, ambil API key dari menu Keys.

Tambahkan auth profile:

```
openclaw models auth add
```

Saat ditanya provider, pilih custom, lalu ketik `openrouter`. Isi profile id dengan nilai default yang disarankan. Saat ditanya apakah token punya masa berlaku, pilih No. Tempelkan API key saat diminta.

Lihat model yang tersedia:

```
openclaw models list
```

Set model default (contoh memakai model gratis untuk keperluan uji coba):

```
openclaw models set openrouter/inclusionai/ling-3.0-flash-fin:free
```

Restart gateway supaya model baru terpakai:

```
systemctl --user restart openclaw-gateway.service
```

Setelah ini, bot Telegram sudah bisa membalas pesan dengan benar.


## 6. Mengakses web dashboard secara permanen lewat domain sendiri

OpenClaw punya web dashboard mirip tampilan chat, bisa diakses lewat browser, tidak harus lewat terminal atau Telegram terus menerus.

### Menemukan dashboard

```
openclaw dashboard --no-open
```

Perintah ini menampilkan URL dashboard yang berjalan di localhost VPS, misalnya `http://127.0.0.1:18789/`. Alamat ini tidak bisa diakses langsung dari browser laptop karena localhost merujuk ke perangkat itu sendiri, bukan ke VPS.

### Mengekspos dashboard lewat Cloudflare Tunnel

VPS ini sudah menjalankan Cloudflare Tunnel untuk n8n. Kita tambahkan satu entry baru untuk OpenClaw di file konfigurasi yang sama.

Penting: file konfigurasi yang benar-benar dipakai oleh service systemd ada di `/etc/cloudflared/config.yml`, bukan di `~/.cloudflared/config.yml` milik user. Percobaan pertama edit di lokasi yang salah tidak berpengaruh apa-apa sampai file yang benar ditemukan dan diedit ulang.

Edit file yang benar:

```
sudo nano /etc/cloudflared/config.yml
```

Tambahkan entry baru sebelum baris `service: http_status:404`, karena baris tersebut berfungsi sebagai catch-all dan harus tetap berada paling akhir:

```
ingress:
  - hostname: n8n.wican.my.id
    service: http://localhost:5678
  - hostname: openclaw.wican.my.id
    service: http://localhost:18789
  - service: http_status:404
```

Restart cloudflared:

```
sudo systemctl restart cloudflared
```

Daftarkan DNS record untuk subdomain baru:

```
cloudflared tunnel route dns <tunnel-id> openclaw.wican.my.id
```

### Mengizinkan akses dari domain baru

Dashboard OpenClaw akan menolak koneksi dari domain yang belum didaftarkan sebagai allowed origin, walaupun tunnel dan DNS sudah benar. Ini fitur keamanan bawaan.

```
openclaw config set gateway.controlUi.allowedOrigins '["https://openclaw.wican.my.id"]'
systemctl --user restart openclaw-gateway.service
```

### Login pertama kali

Buka `https://openclaw.wican.my.id` dari browser mana saja.

Ambil gateway token untuk login:

```
cat ~/.openclaw/openclaw.json | grep -A 2 '"token"'
```

Tempelkan token tersebut di kolom Gateway Token pada halaman login, lalu klik Connect.

Percobaan pertama biasanya akan ditolak dengan pesan "device pairing required", disertai sebuah request ID. Ini bukan error, melainkan lapisan keamanan tambahan yang meminta persetujuan manual dari sisi server untuk perangkat atau browser baru.

Setujui device tersebut dari SSH:

```
openclaw devices approve <request-id-yang-muncul>
```

Kembali ke browser, klik Connect sekali lagi. Dashboard akan terbuka dan menampilkan riwayat percakapan yang sama dengan yang ada di Telegram.


## 7. Konfigurasi identitas assistant

OpenClaw secara otomatis membuat beberapa file markdown di folder workspace saat pertama kali diinstal. Modul resmi menyebutkan pengguna harus membuat file-file ini dari nol, tapi kenyataannya file-file tersebut sudah tersedia dan tinggal disesuaikan.

Lokasi workspace:

```
~/.openclaw/workspace/
```

Isi folder tersebut antara lain:

- SOUL.md, berisi kepribadian dan batasan perilaku assistant
- IDENTITY.md, berisi nama, jenis makhluk, vibe, dan emoji signature assistant
- AGENTS.md, berisi instruksi teknis dan aturan kerja
- USER.md, berisi informasi tentang penggunanya

### Mengisi IDENTITY.md

```
nano ~/.openclaw/workspace/IDENTITY.md
```

Isi setiap field dengan menghapus teks placeholder dalam kurung dan menggantinya dengan nilai sendiri. Contoh yang dipakai di sesi ini:

```
- Name: Wican-AI
- Creature: AI assistant pribadi yang tinggal di server Rizal
- Vibe: Santai kayak jadi teman, ramah, dan gak kaku
- Emoji: (emoji pilihan sendiri)
```

### Menambahkan aturan komunikasi di SOUL.md

```
nano ~/.openclaw/workspace/SOUL.md
```

Tambahkan bagian baru di bawah bagian Vibe, sebelum bagian Continuity, misalnya:

```
## Identity & Communication Style
- Nama kamu: Wican-AI. Selalu perkenalkan diri sebagai Wican-AI.
- Self-reference: pakai "Wican-AI" atau "aku", jangan "saya".
- Bahasa Indonesia casual sehari-hari, jangan kaku atau terlalu formal.
- To the point, jangan bertele-tele kalau gak perlu.
```

### Catatan penting soal perubahan identitas

Perubahan pada SOUL.md dan IDENTITY.md tidak langsung terasa pada percakapan yang sedang berlangsung. Assistant baru menggunakan identitas yang diperbarui pada sesi atau percakapan yang baru dimulai.


## 8. Ringkasan masalah dan solusi

Bagian ini paling penting untuk dibaca ulang, karena berisi hal-hal yang sempat membuat proses tersendat.

### Masalah 1: npm install -g openclaw menginstal package kosong

Package pertama yang terpasang ternyata versi 0.0.1 berlabel "Empty placeholder package", bukan software OpenClaw yang sebenarnya. Solusinya adalah menginstal ulang dengan menuliskan versi secara eksplisit, yaitu `openclaw@latest`.

### Masalah 2: OpenClaw versi asli butuh Node 22 ke atas

Modul resmi menyebutkan syarat Node 18 ke atas, tapi OpenClaw versi 2026.7.1-2 mensyaratkan Node 22.22.3 ke atas. Solusinya adalah memasang nvm untuk mengelola banyak versi Node sekaligus, lalu memasang dan mengaktifkan Node 22.

### Masalah 3: symlink command tidak terbentuk saat install dengan sudo

Instalasi dengan `sudo npm install -g` di folder sistem `/usr/lib/node_modules` sempat membuat package terpasang tapi command-nya tidak bisa dipanggil, karena symlink ke folder bin tidak terbentuk. Solusinya adalah mengatur folder npm global khusus milik user di `~/.npm-global`, sehingga instalasi tidak lagi membutuhkan sudo.

### Masalah 4: konflik antara prefix npm manual dan nvm

Setelah nvm terpasang, muncul peringatan bahwa setting prefix npm yang dibuat manual sebelumnya bentrok dengan cara kerja nvm. Solusinya menjalankan `nvm use --delete-prefix` untuk menghapus setting prefix lama.

### Masalah 5: command dari modul resmi sudah berubah

Beberapa command yang tertulis di modul sudah tidak berlaku di versi OpenClaw terbaru:

- `openclaw gateway start` sudah berubah menjadi kombinasi `openclaw gateway install` dan `systemctl --user start openclaw-gateway.service`
- `openclaw plugins enable telegram` sudah berubah menjadi `openclaw channels add`, karena Telegram sekarang termasuk kategori channels, bukan plugins

Cara mendiagnosis command yang sudah berubah adalah dengan menjalankan `openclaw --help` atau `openclaw <perintah> --help` untuk melihat daftar command dan opsi yang sebenarnya tersedia di versi yang terpasang.

### Masalah 6: file konfigurasi Cloudflare Tunnel yang salah lokasi

Ada dua file config.yml yang terlihat mirip, yaitu `~/.cloudflared/config.yml` milik user dan `/etc/cloudflared/config.yml` yang benar-benar dipakai oleh service systemd. Edit pada file pertama tidak berpengaruh sama sekali sampai file yang kedua ditemukan dan diedit. Cara memastikan lokasi yang benar adalah dengan menjalankan `systemctl status cloudflared` dan melihat path yang tertulis di baris Loaded.

### Masalah 7: dashboard menolak koneksi dari domain publik

Setelah tunnel dan DNS berhasil disiapkan, dashboard tetap menolak koneksi dengan pesan "Browser origin not allowed". Ini bukan masalah jaringan, melainkan proteksi bawaan OpenClaw yang mengharuskan domain didaftarkan secara eksplisit lewat `gateway.controlUi.allowedOrigins`.

### Masalah 8: device pairing diperlukan untuk browser baru

Setelah token diterima, koneksi tetap ditolak dengan pesan "device pairing required". Ini lapisan keamanan tambahan yang mengharuskan persetujuan manual dari server untuk setiap perangkat atau browser baru yang mencoba login, meskipun token yang dimasukkan sudah benar.


## Hasil akhir

Setelah seluruh proses ini selesai, tersedia dua cara mengakses assistant pribadi:

1. Lewat Telegram, dengan bot yang hanya bisa diakses oleh akun sendiri
2. Lewat web dashboard di https://openclaw.wican.my.id, dengan proteksi token dan device pairing

Assistant sudah dikonfigurasi dengan nama Wican-AI, menggunakan gaya komunikasi santai dan bahasa Indonesia casual, sesuai preferensi pengguna.


## Referensi

- OpenClaw Documentation: https://docs.openclaw.ai
- OpenRouter Models: https://openrouter.ai
- Telegram Bot API Docs: https://core.telegram.org/bots/api
- n8n Workflow Automation: https://n8n.io
