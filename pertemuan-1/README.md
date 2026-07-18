# Kelas Agentic AI Automation — Kelas Otomesyen

Repositori ini berisi kumpulan tugas dan project dari Kelas Agentic AI Automation oleh Kelas Otomesyen. Setiap folder mewakili satu pertemuan dengan dokumentasi dan file yang relevan.

---

## Struktur Repositori

```
kelas-otomesyen/
├── pertemuan-1/
    └── index.html
    └── README.md
```

---

## Pertemuan 1: Fondasi — Cloud, VPS & Mindset Automation

### Deskripsi

Modul pertama membahas fondasi infrastruktur untuk automation: konsep cloud computing, setup VPS dari nol, penggunaan terminal Linux, dan deployment aplikasi menggunakan Docker.

### Yang Dipelajari

- Konsep VPS dan perbedaannya dengan laptop/komputer biasa
- Setup VPS dari awal: deploy, SSH, dan keamanan dasar
- Penggunaan terminal Linux untuk operasi sehari-hari
- Install dan menjalankan Docker container

### Spesifikasi VPS yang Digunakan

| Komponen | Detail |
|----------|--------|
| Provider | Sumopod (via Tencent Cloud) |
| Region | Singapore |
| OS | Ubuntu Server 22.04 LTS |
| CPU | 2 vCPU |
| RAM | 2 GB |
| Storage | 40 GB SSD |
| IP Publik | 43.156.100.201 |

### Langkah-Langkah yang Dilakukan

#### 1. Pembuatan VPS

VPS dibuat melalui dashboard Sumopod dengan konfigurasi di atas. Setelah aktif, Sumopod menyediakan IP publik, username, dan password untuk akses pertama.

#### 2. Akses Server via SSH

Login ke server menggunakan SSH dari Command Prompt Windows:

```bash
ssh ubuntu@43.156.100.201
```

Catatan: username default Sumopod adalah `ubuntu`, bukan `root`.

#### 3. Update Sistem

```bash
sudo apt update && sudo apt upgrade -y
```

#### 4. Setup Firewall (UFW)

```bash
sudo apt install ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 8080/tcp
sudo ufw enable
```

#### 5. Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

Setelah perintah di atas, logout lalu login kembali agar permission Docker aktif.

#### 6. Upload File ke Server

Perintah berikut dijalankan di Command Prompt laptop (bukan di server):

```bash
scp "C:\Agentic AI\index.html" ubuntu@43.156.100.201:~/mywebsite/
```

#### 7. Deploy Web Server dengan Docker + Nginx

```bash
mkdir -p ~/mywebsite
cd ~/mywebsite
docker run -d --name portofolio -p 8080:80 \
  -v $(pwd)/index.html:/usr/share/nginx/html/index.html \
  nginx
```

Port yang digunakan adalah `8080` karena port `80` diblokir oleh Tencent Cloud secara default.

#### 8. Verifikasi

Website dapat diakses melalui browser di:

```
http://43.156.100.201:8080
```

### Catatan Penting

**Provider VPS:** Oracle Cloud Free Tier sempat dicoba namun gagal karena sistem Oracle menolak pendaftaran dari Indonesia. Sumopod dipilih sebagai alternatif karena proses aktivasi yang cepat.

**Port 80 vs 8080:** Port 80 tidak dapat diakses dari luar meskipun sudah dibuka di UFW, karena Tencent Cloud memblokir port tersebut di level jaringan. Fitur Firewall di dashboard Sumopod belum tersedia untuk paket ini. Solusinya adalah menjalankan Nginx di port 8080.

**SSH Username:** Sumopod menggunakan username `ubuntu`, bukan `root`. Login sebagai root memang tidak disarankan untuk keamanan.

### File PR

| File | Keterangan |
|------|------------|
| `index.html` | Halaman portofolio pribadi dengan tema maroon, putih, dan gold. Dibuat menggunakan HTML, CSS, dan Lucide Icons. Deployed via Docker + Nginx di VPS. |

### Command Docker yang Sering Dipakai

```bash
docker ps                  # Lihat container yang sedang berjalan
docker ps -a               # Lihat semua container
docker stop portofolio     # Stop container
docker start portofolio    # Start container
docker restart portofolio  # Restart container
docker logs portofolio     # Lihat log container
docker rm portofolio       # Hapus container
```

---

## Tentang

**Rizal Wahyu Pratama**
Machine Learning Geospatial Engineer
Mahasiswa S1 Data Science, Telkom University Purwokerto

- Email: rizalwp@student.telkomuniversity.ac.id
- GitHub: github.com/rizaledc
- Website: kodinginaja.biz.id
