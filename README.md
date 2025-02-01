# Hardenig Ubuntu

---

### **1. Document Host Information**  
**Tujuan**: Meminimalkan informasi sistem yang diekspos ke publik untuk mencegah serangan informasi.  

#### **Langkah Konfigurasi**:  
1. **Atur Nama Host**  
   - Perintah:  
     ```bash
     sudo hostnamectl set-hostname "nama-server-anda"  # Contoh: webserver01
     ```  
   - Verifikasi:  
     ```bash
     hostnamectl
     ```  

2. **Hapus Informasi Sensitif dari Banner Login**  
   - Edit file `/etc/issue` dan `/etc/issue.net`:  
     ```bash
     sudo nano /etc/issue
     sudo nano /etc/issue.net
     ```  
   - Hapus semua detail sistem (seperti versi OS/kernel) dan ganti dengan pesan keamanan:  
     ```plaintext
     Authorized Access Only. Unauthorized use is prohibited.
     ```  

3. **Nonaktifkan Motd (Message of the Day)**  
   - Edit file `/etc/pam.d/sshd`:  
     ```bash
     sudo nano /etc/pam.d/sshd
     ```  
   - Komentari baris berikut dengan menambahkan `#`:  
     ```plaintext
     # session    optional     pam_motd.so
     ```  

**Pengujian**:  
- Login via SSH atau konsol fisik.  
- Pastikan tidak ada informasi sistem yang muncul selain pesan keamanan.

---

### **2. Hardisk Encription**  
**Tujuan**: Melindungi data dari akses fisik dengan enkripsi partisi.  

#### **Langkah Konfigurasi**:  
**A. Enkripsi Partisi dengan LUKS (Untuk Partisi Baru)**  
1. Format partisi (misal: `/dev/sdb1`):  
   ```bash
   sudo cryptsetup luksFormat /dev/sdb1
   ```  
   - **Catatan**: Partisi akan diformat ulang. Backup data sebelumnya!  

2. Buka partisi terenkripsi:  
   ```bash
   sudo cryptsetup luksOpen /dev/sdb1 encrypted_drive
   ```  

3. Buat filesystem dan mount:  
   ```bash
   sudo mkfs.ext4 /dev/mapper/encrypted_drive
   sudo mount /dev/mapper/encrypted_drive /mnt
   ```  

**B. Enkripsi Home Directory dengan ecryptfs (Untuk Sistem yang Sudah Terinstal)**  
1. Instal paket:  
   ```bash
   sudo apt install ecryptfs-utils
   ```  

2. Migrasi home directory pengguna:  
   ```bash
   sudo ecryptfs-migrate-home -u username_anda
   ```  
   - Ikuti petunjuk untuk membuat passphrase.  

3. Restart dan login ulang.  

**Pengujian**:  
- Matikan server, lepas harddisk, dan coba akses data di sistem lain.  
- Data harus tidak terbaca tanpa passphrase LUKS/ecryptfs.

---

### **3. Disk Protection**  
**Tujuan**: Membatasi eksekusi file berbahaya di partisi tertentu.  

#### **Langkah Konfigurasi**:  
1. Edit file `/etc/fstab`:  
   ```bash
   sudo nano /etc/fstab
   ```  

2. Tambahkan opsi `noexec,nosuid,nodev` pada partisi yang tidak memerlukan eksekusi:  
   - Contoh untuk `/tmp`:  
     ```plaintext
     /dev/sda3  /tmp  ext4  defaults,noexec,nosuid,nodev  0 0
     ```  
   - **Penjelasan Opsi**:  
     - `noexec`: Blok eksekusi file.  
     - `nosuid`: Blok SUID/SGID.  
     - `nodev`: Blok device files.  

3. Remount partisi:  
   ```bash
   sudo mount -o remount /tmp
   ```  

**Pengujian**:  
- Coba jalankan script di `/tmp`:  
  ```bash
  echo '#!/bin/bash' > /tmp/test.sh && chmod +x /tmp/test.sh && /tmp/test.sh
  ```  
  - Seharusnya muncul error: **Permission denied**.

---

### **4. Boot Directory Lock**  
**Tujuan**: Mencegah modifikasi tidak sah pada direktori boot.  

#### **Langkah Konfigurasi**:  
1. Ubah kepemilikan direktori `/boot`:  
   ```bash
   sudo chown root:root /boot
   ```  

2. Ubah hak akses:  
   ```bash
   sudo chmod 700 /boot  # Hanya root yang bisa akses
   ```  

3. **Opsional**: Nonaktifkan akses fisik ke GRUB:  
   - Edit file `/etc/default/grub`:  
     ```bash
     sudo nano /etc/default/grub
     ```  
   - Tambahkan:  
     ```plaintext
     GRUB_DISABLE_OS_PROBER=true
     ```  
   - Update GRUB:  
     ```bash
     sudo update-grub
     ```  

**Pengujian**:  
- Coba akses `/boot` sebagai user biasa:  
  ```bash
  ls /boot
  ```  
  - Seharusnya muncul error: **Permission denied**.

---

### **5. Closed Unusual Open Port**  
**Tujuan**: Menutup port yang tidak digunakan untuk mengurangi vektor serangan.  

#### **Langkah Konfigurasi**:  
1. Aktifkan UFW (Uncomplicated Firewall):  
   ```bash
   sudo ufw enable
   ```  

2. Izinkan port esensial (sesuaikan kebutuhan):  
   ```bash
   sudo ufw allow 22/tcp  # SSH
   sudo ufw allow 80/tcp  # HTTP
   sudo ufw allow 443/tcp # HTTPS
   ```  

3. Tolak semua koneksi masuk lainnya:  
   ```bash
   sudo ufw default deny incoming
   ```  

4. Periksa status:  
   ```bash
   sudo ufw status verbose
   ```  

5. Tutup port tidak perlu dengan `iptables` (opsional):  
   ```bash
   sudo iptables -A INPUT -p tcp --dport 12345 -j DROP  # Contoh blokir port 12345
   sudo netfilter-persistent save
   ```  

**Pengujian**:  
- Scan port dengan `nmap`:  
  ```bash
  sudo apt install nmap
  nmap -sS localhost
  ```  
  - Hanya port 22, 80, 443 yang terbuka.  

---
### **6. Konfigurasi dan Pengujian Hardening Whitelisting SELinux di Ubuntu**  

> **Catatan**: SELinux **tidak diaktifkan secara default di Ubuntu**. Ubuntu lebih sering menggunakan **AppArmor** sebagai sistem keamanan berbasis Mandatory Access Control (MAC). Jika ingin menggunakan SELinux, kita perlu menginstalnya dan mengaktifkannya secara manual.  

---

 **1️⃣ Periksa Apakah SELinux Sudah Terinstal**
Cek apakah SELinux sudah terinstal di sistem:  
```bash
sestatus
```
Jika hasilnya **command not found** atau menunjukkan **SELinux disabled**, berarti SELinux belum diinstal.  

---

 **2️⃣ Install SELinux di Ubuntu**
Jalankan perintah berikut untuk menginstal SELinux di Ubuntu:  
```bash
sudo apt update
sudo apt install selinux-basics selinux-policy-default selinux-utils -y
```

Setelah instalasi selesai, **aktifkan SELinux** dengan perintah berikut:  
```bash
sudo selinux-activate
```
**Reboot sistem** agar perubahan diterapkan:  
```bash
sudo reboot
```

---

 **3️⃣ Verifikasi Status SELinux**
Setelah sistem menyala kembali, periksa apakah SELinux sudah aktif:  
```bash
sestatus
```
Jika hasilnya menunjukkan **SELinux enabled (Enforcing mode)**, maka SELinux sudah berjalan.  

---

 **4️⃣ Mengaktifkan Whitelisting pada SELinux**
Whitelisting dalam SELinux berarti **mengizinkan aplikasi atau proses tertentu untuk berjalan** dengan menyesuaikan kebijakan SELinux.  

**Langkah-langkah:**

 **🔹 4.1 Cek Log AVC (Access Vector Cache) untuk Mengetahui Blokir SELinux**
Jika SELinux memblokir suatu proses atau aplikasi, kita bisa melihat lognya dengan:  
```bash
sudo ausearch -m AVC,USER_AVC -ts recent
```
Atau dengan:
```bash
sudo journalctl | grep AVC
```
Hasilnya akan menunjukkan aplikasi yang diblokir oleh SELinux.

---

**🔹 4.2 Tambahkan Aplikasi atau Proses ke dalam Whitelist**
Untuk mengizinkan suatu proses yang diblokir SELinux, kita bisa membuat kebijakan khusus.  

Contoh: Jika SELinux memblokir **Apache** (`httpd`), kita bisa menambahkan kebijakan whitelist dengan:
```bash
sudo semanage permissive -a httpd_t
```
Atau jika ada aplikasi tertentu yang ingin di-whitelist, gunakan:
```bash
sudo semanage permissive -a <nama_tipe>
```
Untuk melihat daftar tipe SELinux yang ada, jalankan:
```bash
sudo semanage boolean -l
```

---

 **5️⃣ Menguji Kebijakan Whitelisting**
Setelah menambahkan whitelist, uji apakah aplikasi yang sebelumnya diblokir kini dapat berjalan.  

1. Jalankan ulang layanan yang diblokir:
   ```bash
   sudo systemctl restart apache2
   ```
2. Cek apakah masih ada error di log SELinux:
   ```bash
   sudo ausearch -m AVC,USER_AVC -ts recent
   ```
3. Jika masih ada error, perbaiki kebijakan dengan:
   ```bash
   sudo restorecon -Rv /path/to/file_or_directory
   ```
   (Ganti `/path/to/file_or_directory` dengan path file yang diblokir.)

---

 **6️⃣ (Opsional) Mengubah Mode SELinux**
- **Enforcing (default, ketat)**:
  ```bash
  sudo setenforce 1
  ```
- **Permissive (hanya mencatat, tidak memblokir)**:
  ```bash
  sudo setenforce 0
  ```
- **Permanent (agar tetap setelah reboot), edit file `/etc/selinux/config`**:
  ```bash
  sudo nano /etc/selinux/config
  ```
  Ubah:
  ```
  SELINUX=enforcing
  ```
  atau jika ingin permissive:
  ```
  SELINUX=permissive
  ```
  Simpan dan keluar (**Ctrl+X → Y → Enter**), lalu reboot:
  ```bash
  sudo reboot
  ```

---

### **7. CHROOT Shell**  
**Tujuan**: Membatasi pengguna ke direktori tertentu (jail) saat login via SSH.  

#### **Langkah Konfigurasi**:  
1. **Buat Direktori Chroot**:  
   ```bash
   sudo mkdir -p /home/chroot_jail/{bin,dev,etc,lib,usr,lib64}
   ```

2. **Salin File Sistem Esensial**:  
   ```bash
   # Salin binary shell (bash)
   sudo cp /bin/bash /home/chroot_jail/bin/

   # Salin library yang diperlukan
   sudo ldd /bin/bash | grep "=>" | awk '{print $3}' | xargs -I {} cp {} /home/chroot_jail/lib/

   # Salin file konfigurasi
   sudo cp /etc/passwd /etc/group /home/chroot_jail/etc/
   ```

3. **Buat Pengguna Chroot**:  
   ```bash
   sudo useradd -M -d /home/chroot_jail -s /bin/bash chroot_user
   sudo passwd chroot_user
   ```

4. **Konfigurasi SSH untuk Chroot**:  
   - Edit file `/etc/ssh/sshd_config`:  
     ```bash
     sudo nano /etc/ssh/sshd_config
     ```  
   - Tambahkan di akhir:  
     ```plaintext
     Match User chroot_user
         ChrootDirectory /home/chroot_jail
         AllowTCPForwarding no
         X11Forwarding no
     ```

5. **Restart SSH**:  
   ```bash
   sudo systemctl restart ssh
   ```

**Pengujian**:  
- Login sebagai `chroot_user`:  
  ```bash
  ssh chroot_user@localhost
  ```  
- Coba akses direktori di luar `/home/chroot_jail` (seperti `/etc`).  
  - Seharusnya muncul error: **No such file or directory**.

---

### **8. Certificate Shell Login**  
**Tujuan**: Login SSH menggunakan sertifikat, bukan password.  

#### **Langkah Konfigurasi**:  
1. **Generate SSH Key Pair di Client**:  
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/server_access
   ```  
   - Simpan di `~/.ssh/server_access` (tanpa passphrase jika opsional).

2. **Salin Public Key ke Server**:  
   ```bash
   ssh-copy-id -i ~/.ssh/server_access.pub user@server-ip
   ```

3. **Nonaktifkan Login Password di Server**:  
   - Edit file `/etc/ssh/sshd_config`:  
     ```plaintext
     PasswordAuthentication no
     ChallengeResponseAuthentication no
     ```  
   - Restart SSH:  
     ```bash
     sudo systemctl restart ssh
     ```

**Pengujian**:  
- Coba login tanpa sertifikat:  
  ```bash
  ssh user@server-ip
  ```  
  - Seharusnya muncul error: **Permission denied (publickey)**.

---

### **9. Directory Listing**  
**Tujuan**: Menonaktifkan direktori listing di web server (Apache).  

#### **Langkah Konfigurasi**:  
1. **Edit Konfigurasi Apache**:  
   ```bash
   sudo nano /etc/apache2/apache2.conf
   ```  

2. **Tambahkan Parameter di Bagian `<Directory>`**:  
   ```apache
   <Directory /var/www/html>
       Options -Indexes
   </Directory>
   ```  

3. **Restart Apache**:  
   ```bash
   sudo systemctl restart apache2
   ```

**Pengujian**:  
- Buka browser dan akses `http://server-ip/` tanpa file `index.html`.  
  - Seharusnya muncul error **403 Forbidden**, bukan daftar file.

---

### **10. Hardening PHP untuk Mencegah Remote Command Execution** 

#### **Langkah 1: Instalasi PHP di Ubuntu**  
1. **Update Repositori**:  
   ```bash
   sudo apt update
   ```

2. **Instal PHP dan Modul Apache**:  
   ```bash
   sudo apt install php libapache2-mod-php -y
   ```

3. **Verifikasi Instalasi**:  
   ```bash
   php -v  # Harus menampilkan versi PHP (contoh: PHP 8.2.x)
   ```

---

#### **Langkah 2: Temukan File Konfigurasi PHP (`php.ini`)**  
1. **Cari Lokasi `php.ini`**:  
   ```bash
   php --ini | grep "Loaded Configuration File"
   ```  
   Contoh output:  
   ```
   Loaded Configuration File => /etc/php/8.2/apache2/php.ini
   ```  
   *Catatan*: Jika menggunakan PHP-FPM, file bisa berada di `/etc/php/8.2/fpm/php.ini`.

---

#### **Langkah 3: Nonaktifkan Fungsi Berbahaya di PHP**  
1. **Buka File `php.ini` dengan Editor**:  
   ```bash
   sudo nano /etc/php/8.2/apache2/php.ini  # Sesuaikan versi PHP
   ```

2. **Cari Baris `disable_functions`**:  
   - Tekan `Ctrl+W` di nano, lalu ketik `disable_functions` untuk mencari.  
   - Jika tidak ada, tambahkan baris berikut di bagian mana saja:  
     ```ini
     disable_functions = exec,passthru,shell_exec,system,proc_open,popen,pcntl_exec
     ```  
   - Jika sudah ada, tambahkan fungsi-fungsi tersebut ke dalamnya (pisahkan dengan koma).

3. **Simpan Perubahan**:  
   - Tekan `Ctrl+O` → `Enter` → `Ctrl+X`.

---

#### **Langkah 4: Restart Apache**  
```bash
sudo systemctl restart apache2
```

---

#### **Langkah 5: Verifikasi Konfigurasi**  
1. **Buat File PHP untuk Uji Coba**:  
   ```bash
   sudo nano /var/www/html/test.php
   ```  
   Isi dengan kode berikut:  
   ```php
   <?php
   echo "<h3>Uji Fungsi PHP yang Dinonaktifkan:</h3>";
   echo "Hasil shell_exec('ls'): " . shell_exec('ls');
   echo "Hasil exec('whoami'): " . exec('whoami');
   ?>
   ```

2. **Akses File via Browser**:  
   Buka: `http://[IP-server-anda]/test.php`.  
   **Hasil yang Diharapkan**:  
   - Tidak ada output dari `shell_exec()` atau `exec()`.  
   - Pesan error mungkin muncul:  
     ```
     Warning: shell_exec() has been disabled for security reasons...
     ```

3. **Cek Log Error PHP**:  
   ```bash
   sudo tail -f /var/log/apache2/error.log
   ```  
   Pastikan log mencatat error terkait fungsi yang dinonaktifkan.

---

#### **Troubleshooting**  
##### **Jika Fungsi Masih Bekerja**  
1. **Pastikan Anda Mengedit File `php.ini` yang Benar**:  
   - Verifikasi dengan perintah:  
     ```bash
     php -i | grep "disable_functions"
     ```  
   - Output harus menampilkan daftar fungsi yang sudah dinonaktifkan.

2. **Pastikan Apache Di-restart**:  
   ```bash
   sudo systemctl restart apache2
   ```

---

#### **Tambahan: Keamanan Ekstra untuk PHP**  
1. **Aktifkan `open_basedir`**:  
   Batasi PHP hanya bisa mengakses direktori tertentu.  
   Edit `php.ini`:  
   ```ini
   open_basedir = /var/www/html
   ```

2. **Nonaktifkan `eval()`**:  
   Tambahkan `eval` ke `disable_functions`:  
   ```ini
   disable_functions = ...,eval
   ```

3. **Aktifkan ModSecurity**:  
   Pasang Web Application Firewall untuk memfilter serangan:  
   ```bash
   sudo apt install libapache2-mod-security2 -y
   ```

---

#### **Contoh Kasus**  
- **Sebelum Hardening**:  
  ```php
  <?php echo shell_exec('rm -rf /'); ?>  # Bisa menghapus seluruh sistem!
  ```  
  Output: File terhapus.  

- **Setelah Hardening**:  
  ```php
  <?php echo shell_exec('rm -rf /'); ?>  
  ```  
  Output: Error **"shell_exec() has been disabled"**.

---
