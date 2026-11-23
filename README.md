1. Instal PHP, Ekstensi Laravel, dan Composer
Masuk ke terminal SSH EC2 Debian Anda, lalu pasang PHP beserta modul yang dibutuhkan oleh Laravel dan MySQL:

```bash
sudo apt update
sudo apt install -y php libapache2-mod-php php-mysql php-mbstring php-xml php-bcmath php-curl php-zip php-gd unzip curl git
```

Setelah itu, pasang Composer secara global:

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer --version
```

Verifikasi: Perintah 'composer --version' menampilkan versi Composer yang terpasang tanpa galat.

2. Siapkan Proyek Laravel (Clone atau Buat Baru)
Masuk ke direktori /var/www/ lalu ambil proyek Laravel dari repository Git Anda (atau buat proyek baru jika soal meminta instalasi bersih):

```bash
cd /var/www
sudo composer create-project laravel/laravel aplikasi-aws
```

3. Atur Kepemilikan dan Hak Akses Direktori
Ubah kepemilikan folder proyek ke user web server Apache (www-data) agar Laravel dapat menulis file cache dan log:

```bash
sudo chown -R www-data:www-data /var/www/aplikasi-aws
sudo chmod -R 775 /var/www/aplikasi-aws/storage /var/www/aplikasi-aws/bootstrap/cache
```

4. Konfigurasi File .env (Koneksi ke RDS MySQL & S3)
Masuk ke direktori proyek dan salin file .env.example menjadi .env (jika belum ada), lalu generate application key:

```bash
cd /var/www/aplikasi-aws
sudo -u www-data cp .env.example .env
sudo -u www-data php artisan key:generate
```

Buka file .env menggunakan 'sudo nano /var/www/aplikasi-aws/.env', lalu sesuaikan kredensial RDS MySQL dan S3 yang sudah Anda buat pada tahap sebelumnya:

APP_ENV=production
APP_DEBUG=false
APP_URL=http://<PUBLIC_IP_EC2_ANDA>

DB_CONNECTION=mysql
DB_HOST=<ENDPOINT_RDS_ANDA>
DB_PORT=3306
DB_DATABASE=<NAMA_DATABASE_RDS>
DB_USERNAME=admin
DB_PASSWORD=<PASSWORD_RDS_ANDA>

FILESYSTEM_DISK=s3
AWS_DEFAULT_REGION=<REGION_AWS_ANDA>
AWS_BUCKET=<NAMA_BUCKET_S3_ANDA>

(Catatan: Biarkan AWS_ACCESS_KEY_ID dan AWS_SECRET_ACCESS_KEY kosong atau hapus barisnya agar Laravel otomatis mengambil kredensial sementara dari IAM Role yang sudah menempel pada EC2).

Jalankan migrasi tabel ke RDS untuk menguji koneksi database dari Laravel:

```bash
sudo -u www-data php artisan migrate --force
```

5. Konfigurasi Virtual Host Apache ke Folder /public
Laravel wajib diarahkan ke direktori public/ sebagai DocumentRoot. Buat file konfigurasi Virtual Host baru:
```bash
sudo nano /etc/apache2/sites-available/laravel.conf
```

Masukkan konfigurasi berikut:

```bash
<VirtualHost *:80>
    ServerName _
    DocumentRoot /var/www/aplikasi-aws/public

    <Directory /var/www/aplikasi-aws/public>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/laravel-error.log
    CustomLog ${APACHE_LOG_DIR}/laravel-access.log combined
</VirtualHost>
```

Aktifkan modul rewrite, aktifkan konfigurasi laravel.conf, nonaktifkan situs bawaan 000-default.conf, lalu restart Apache:

```bash
sudo a2enmod rewrite
sudo a2dissite 000-default.conf
sudo a2ensite laravel.conf
sudo apache2ctl configtest
sudo systemctl restart apache2
```
