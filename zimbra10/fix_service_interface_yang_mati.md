Zimbra 10.1.13 Ubuntu 20 (Versi Intalio)

# Dokumentasi Perbaikan Layanan Zimbra

Dokumentasi ini menjelaskan langkah-langkah yang dilakukan untuk memperbaiki layanan Zimbra, khususnya masalah dengan SNMP dan Stats yang menunjukkan status "Stopped" di zmcontrol dan antarmuka admin.

## Masalah Awal

- Layanan SNMP dan Stats tidak berjalan meskipun semua layanan lain tampak berjalan
- Antarmuka admin Zimbra menunjukkan tanda 'x' untuk beberapa layanan
- zmcontrol status menunjukkan bahwa SNMP dan Stats dalam status "Stopped"

## Langkah-langkah Perbaikan

### 1. Memperbaiki zmstatuslog

zmstatuslog tidak dijadwalkan dalam crontab, sehingga status layanan tidak diperbarui secara berkala:

```bash
# Menambahkan zmstatuslog ke crontab pengguna zimbra
(crontab -u zimbra -l; echo "*/5 * * * * /opt/zimbra/libexec/zmstatuslog") | crontab -u zimbra -
```

### 2. Memperbaiki Layanan Logger

Layanan logger gagal berjalan karena modul Perl Swatchdog::Actions tidak ditemukan:

```bash
# Install paket swatch yang diperlukan
sudo apt-get install -y swatch

# Start layanan logger
sudo -u zimbra /opt/zimbra/bin/zmloggerctl start
```

### 3. Memperbaiki MySQL Server

MySQL server terus berhenti, menyebabkan layanan lain juga berhenti:

```bash
# Start MySQL server secara manual
sudo -u zimbra /opt/zimbra/bin/mysql.server start
```

### 4. Memperbaiki SNMP dan Stats

Melakukan restart berjenjang untuk memastikan semua layanan berjalan dengan benar:

```bash
# Restart zmconfigd
sudo -u zimbra /opt/zimbra/bin/zmconfigdctl restart

# Start layanan secara berurutan
sudo -u zimbra /opt/zimbra/bin/zmloggerctl start
sudo -u zimbra /opt/zimbra/bin/zmmailboxdctl start

# Start semua layanan
sudo -u zimbra /opt/zimbra/bin/zmcontrol start
```

### 5. Memastikan SNMP Berjalan

Install dan konfigurasi SNMP daemon:

```bash
# Install SNMP packages
sudo apt-get install -y snmpd snmp

# Enable dan start SNMP daemon
sudo systemctl enable snmpd
sudo systemctl start snmpd
```

### 6. Verifikasi Akhir

Restart layanan penting dan verifikasi status:

```bash
# Restart zmconfigd dan zmmailboxd
sudo -u zimbra /opt/zimbra/bin/zmconfigdctl restart
sudo -u zimbra /opt/zimbra/bin/zmmailboxdctl restart

# Jalankan zmstatuslog untuk memperbarui status
sudo -u zimbra /opt/zimbra/libexec/zmstatuslog

# Cek status akhir
sudo -u zimbra /opt/zimbra/bin/zmcontrol status
```

## Hasil Akhir

Setelah semua langkah dilakukan:

- Semua layanan Zimbra menunjukkan status "Running" di zmcontrol
- Layanan SNMP dan Stats sekarang berjalan dengan benar
- zmstatuslog dijadwalkan dalam crontab untuk pembaruan status otomatis
- Antarmuka admin Zimbra seharusnya menampilkan semua layanan dengan status yang benar

## Verifikasi Status Layanan

Gunakan perintah berikut untuk memeriksa status layanan:

```bash
sudo -u zimbra /opt/zimbra/bin/zmcontrol status
```

## Jadwal Pembaruan Status

zmstatuslog dijadwalkan untuk berjalan setiap 5 menit:

```
*/5 * * * * /opt/zimbra/libexec/zmstatuslog
```

Ini memastikan bahwa status layanan terus diperbarui dan tercermin dengan akurat di antarmuka admin Zimbra.

## Troubleshooting Tambahan

Jika SNMP atau Stats kembali menunjukkan status "Stopped":

1. Cek log di `/opt/zimbra/log/`
2. Pastikan zmstatuslog berjalan sesuai jadwal
3. Restart zmconfigd dan zmmailboxd jika diperlukan
4. Jalankan zmstatuslog secara manual untuk pembaruan segera
