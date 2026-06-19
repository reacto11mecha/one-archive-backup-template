# 🗄️ One Archive Backup Vault

Repositori ini adalah templat *Continuous Integration* (CI) berbasis GitHub Actions yang dirancang khusus untuk mencadangkan (mem-backup) Database PostgreSQL dan penyimpanan objek SeaweedFS (S3) dari sistem **One Archive** Anda secara otomatis.

Alur kerja (*workflow*) di repositori ini menggunakan **Chisel** untuk menembus jaringan lokal peladen (NAT/Firewall) dengan aman, mengekstrak data terbaru, dan menyimpannya langsung ke dalam repositori ini sebagai arsip riwayat data.

---

## 🚀 Panduan Setup Awal

### 1. Buat Repositori Backup Baru (Gunakan Templat)
Langkah pertama adalah membuat "brankas" pribadi Anda menggunakan templat ini.
1. Klik tombol hijau **`Use this template`** di sudut kanan atas halaman repositori ini, lalu pilih **`Create a new repository`**.
2. Beri nama repositori baru Anda (contoh: `one-archive-backup-vault`).
3. ⚠️ **SANGAT PENTING:** Pastikan Anda menyetel visibilitas repositori menjadi **Private**. Repositori ini akan menyimpan data mentah arsip sekolah dan pangkalan data (*database*) Anda yang bersifat sangat rahasia.
4. Klik **`Create repository`**.

### 2. Konfigurasi Kredensial (GitHub Secrets)
Agar GitHub Actions di peladen *cloud* diizinkan untuk menyedot data dari peladen lokal Anda melalui Chisel, Anda wajib mendaftarkan variabel rahasia.
1. Pada repositori **Private** baru Anda, buka menu **`Settings`** > **`Secrets and variables`** > **`Actions`**.
2. Klik tombol hijau **`New repository secret`**.
3. Tambahkan 7 *secrets* berikut satu per satu sesuai dengan konfigurasi peladen Anda:

| Nama Secret | Contoh Nilai | Keterangan |
| :--- | :--- | :--- |
| `CHISEL_AUTH` | `admin:superrahasia` | Kredensial *User:Password* peladen Chisel Anda. |
| `CHISEL_SERVER` | `https://chisel.domain.com` | URL peladen Chisel publik (*relay*) Anda. |
| `DB_NAME` | `one-archive` | Nama pangkalan data PostgreSQL Anda. |
| `DB_PORT` | `5431` | Port pangkalan data (sesuai *mapping* Docker Compose Anda). |
| `DB_USER_PASS` | `password_db_anda` | Kata sandi pengguna `root` PostgreSQL. |
| `S3_ACCESS_KEY` | `AKIAIOSFODNN7EXAMPLE` | *Access Key* SeaweedFS (Sama dengan `.env` web). |
| `S3_SECRET_KEY` | `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY` | *Secret Key* SeaweedFS (Sama dengan `.env` web). |

### 3. Menjalankan Proses Backup
Secara bawaan, jadwal otomatis (`cron`) pada file `.github/workflows/backup.yml` dinonaktifkan (di-*comment*). Untuk menjalankan *backup*:
1. Buka tab **`Actions`** di repositori Anda.
2. Pada bilah kiri, klik *workflow* **`Backup Database & S3 CI`**.
3. Di sisi kanan, klik **`Run workflow`** > **`Run workflow`**.
4. GitHub akan menjalankan mesin virtual Ubuntu, membuka terowongan Chisel ke peladen lokal Anda, melakukan *dump* data, mengkompresinya, dan melakukan *commit* file-file tersebut ke dalam folder `/backups` repositori Anda.

*(Jika Anda ingin *backup* berjalan otomatis setiap hari, silakan hapus tanda `#` pada baris `schedule` dan `cron` di dalam file `backup.yml`).*

---

## 🛟 Panduan Pemulihan (Disaster Recovery)

Jika terjadi kegagalan sistem (seperti peladen rusak atau *hard drive* korup), Anda dapat memulihkan sistem One Archive ke titik *backup* terakhir dengan langkah-langkah berikut. 

**Prasyarat:**
Pastikan kontainer Docker `one-archive-db` dan klaster `seaweedfs` sudah menyala dalam kondisi kosong di peladen baru Anda, serta Anda sudah mengunduh berkas *backup* (`.sql` dan `.tar.gz`) dari direktori `/backups` repositori ini ke dalam peladen.

### Memulihkan Database (PostgreSQL)
Proses ini akan memasukkan seluruh rekam jejak tabel, klasifikasi, dan data pengguna dari berkas SQL langsung ke dalam kontainer yang berjalan.
1. Buka terminal peladen Anda dan arahkan ke lokasi berkas SQL hasil unduhan.
2. Jalankan perintah berikut (sesuaikan nama berkas `db_tanggal.sql`):

```bash
cat db_tanggal.sql | docker exec -i one-archive-db psql -U root -d one-archive
```

### Memulihkan Berkas Fisik (SeaweedFS S3)

Kita menggunakan AWS CLI untuk mengembalikan seluruh dokumen fisik (PDF/Gambar) ke dalam *bucket* penyimpanan.

1. Ekstrak berkas *backup* S3 (contoh: `s3_tanggal.tar.gz`) untuk memunculkan folder direktori sementara bernama `s3_tmp`:

```bash
tar -xzvf s3_tanggal.tar.gz
```

2. Daftarkan kredensial SeaweedFS Anda ke dalam sesi terminal peladen saat itu juga:

```bash
export AWS_ACCESS_KEY_ID="STORAGE_ACCESS_KEY_ANDA"
export AWS_SECRET_ACCESS_KEY="STORAGE_SECRET_KEY_ANDA"
export AWS_DEFAULT_REGION="us-east-1"
```

3. Sinkronisasikan isi folder hasil ekstrak ke dalam klaster SeaweedFS lokal (Port `8333`):

```bash
aws s3 sync ./s3_tmp s3://onearchive --endpoint-url http://localhost:8333
```

4. Bersihkan file *backup* dan folder sementara untuk menghemat ruang cakram (*disk space*):

```bash
rm -rf ./s3_tmp
rm s3_tanggal.tar.gz db_tanggal.sql
```

🎉 **Selesai!** Peladen One Archive Anda sudah sepenuhnya hidup kembali dengan data dari *backup* terakhir.
