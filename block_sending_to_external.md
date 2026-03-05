# Dokumentasi Pemblokiran Email Eksternal dari Akun root@m3bolt.com

## Tanggal Implementasi
5 Maret 2026

## Latar Belakang
Akun `root@m3bolt.com` perlu diblokir agar tidak dapat mengirim email ke alamat eksternal (di luar domain m3bolt.com) untuk alasan keamanan dan pencegahan penyalahgunaan.

## Tujuan
- Mencegah akun `root@m3bolt.com` mengirim email ke domain eksternal
- Tetap mengizinkan pengiriman email internal dalam domain `m3bolt.com`
- Implementasi dilakukan di level server (Postfix) tanpa mengubah konfigurasi aplikasi

## Akun yang Dibatasi
| Akun | Email Eksternal | Email Internal |
|------|-----------------|----------------|
| `root@m3bolt.com` | ❌ Diblokir | ✅ Diizinkan |
| `admin@m3bolt.com` | ❌ Diblokir | ✅ Diizinkan |

## Metode yang Digunakan
**Sender-Dependent Relayhost Mapping**

Metode ini menggunakan fitur Postfix `sender_dependent_relayhost_maps` untuk mengarahkan email dari akun tertentu ke transport "blackhole" yang akan membuang email tersebut.

## File Konfigurasi

### 1. File Restriction Map
**Lokasi:** `/opt/zimbra/common/conf/sender_relay_restrictions`

**Isi file:**
```
# Route admin@m3bolt.com external emails to a blackhole
admin@m3bolt.com    [blackhole]

# Route root@m3bolt.com external emails to a blackhole
root@m3bolt.com    [blackhole]
```

### 2. Transport Blackhole
**Lokasi:** `/opt/zimbra/common/conf/master.cf.in`

**Entry yang ditambahkan:**
```
blackhole   unix/discard  -       n       n       -       -       discard
```

### 3. Konfigurasi Postfix Main
**Parameter:** `sender_dependent_relayhost_maps`

**Nilai:**
```
sender_dependent_relayhost_maps = lmdb:/opt/zimbra/common/conf/sender_relay_restrictions
```

## Langkah-Langkah Implementasi

### Step 1: Buat/Edit File Restriction Map
```bash
# Tambahkan entry untuk root
echo "root@m3bolt.com    [blackhole]" >> /opt/zimbra/common/conf/sender_relay_restrictions
```

### Step 2: Generate LMDB Database
```bash
su - zimbra -c "/opt/zimbra/common/sbin/postmap /opt/zimbra/common/conf/sender_relay_restrictions"
```

### Step 3: Tambahkan Transport Blackhole ke master.cf.in
```bash
echo 'blackhole   unix/discard  -       n       n       -       -       discard' >> /opt/zimbra/common/conf/master.cf.in
chown zimbra:zimbra /opt/zimbra/common/conf/master.cf.in
```

### Step 4: Konfigurasi Postfix
```bash
su - zimbra -c "postconf -e 'sender_dependent_relayhost_maps = lmdb:/opt/zimbra/common/conf/sender_relay_restrictions'"
```

### Step 5: Restart MTA
```bash
su - zimbra -c "zmmtactl restart"
```

## Verifikasi

### Cek Konfigurasi Aktif
```bash
# Verifikasi parameter Postfix
su - zimbra -c "postconf sender_dependent_relayhost_maps"

# Verifikasi mapping root
su - zimbra -c "postmap -q root@m3bolt.com /opt/zimbra/common/conf/sender_relay_restrictions"

# Verifikasi mapping admin
su - zimbra -c "postmap -q admin@m3bolt.com /opt/zimbra/common/conf/sender_relay_restrictions"
```

### Test Email Eksternal (Harus Gagal)
```bash
echo -e "Subject: Test External\nFrom: root@m3bolt.com\nTo: test@gmail.com\n\nTest" | \
/opt/zimbra/common/sbin/sendmail -f root@m3bolt.com test@gmail.com
```

**Expected result di log:**
```
postfix/smtp[PID]: QUEUE_ID: to=<test@gmail.com>, relay=none, delay=X, 
delays=X/X/X/X, dsn=5.4.4, status=bounced (Host or domain name not found. 
Name service error for name=blackhole type=A: Host not found)
```

### Test Email Internal (Harus Berhasil)
```bash
echo -e "Subject: Test Internal\nFrom: root@m3bolt.com\nTo: admin@m3bolt.com\n\nTest" | \
/opt/zimbra/common/sbin/sendmail -f root@m3bolt.com admin@m3bolt.com
```

**Expected result di log:**
```
postfix/lmtp[PID]: QUEUE_ID: to=<admin@m3bolt.com>, relay=m3bolt.com[IP]:7025, 
delay=X, delays=X/X/X/X, dsn=2.1.5, status=sent (250 2.1.5 Delivery OK)
```

## Monitoring

### Cek Log Email
```bash
# Monitor log secara real-time
tail -f /var/log/zimbra.log | grep -E "root@m3bolt|blackhole"

# Cek email yang diblokir dalam 1 jam terakhir
grep "root@m3bolt.com" /var/log/zimbra.log | grep "status=bounced"
```

### Cek Antrian Email
```bash
su - zimbra -c "mailq"
```

## Cara Menambahkan Akun Lain

Untuk menambahkan akun lain yang ingin diblokir:

```bash
# 1. Tambahkan entry baru ke file restriction
echo "akun.baru@m3bolt.com    [blackhole]" >> /opt/zimbra/common/conf/sender_relay_restrictions

# 2. Regenerate LMDB database
su - zimbra -c "/opt/zimbra/common/sbin/postmap /opt/zimbra/common/conf/sender_relay_restrictions"

# 3. Reload Postfix
su - zimbra -c "postfix reload"
```

## Cara Menghapus Pemblokiran

Untuk menghapus pemblokiran dari sebuah akun:

```bash
# 1. Edit file restriction dan hapus baris akun yang bersangkutan
nano /opt/zimbra/common/conf/sender_relay_restrictions

# 2. Regenerate LMDB database
su - zimbra -c "/opt/zimbra/common/sbin/postmap /opt/zimbra/common/conf/sender_relay_restrictions"

# 3. Reload Postfix
su - zimbra -c "postfix reload"
```

## Troubleshooting

### Email eksternal masih terkirim
1. Pastikan file restriction sudah benar formatnya (tab antara akun dan [blackhole])
2. Regenerate LMDB database: `postmap /opt/zimbra/common/conf/sender_relay_restrictions`
3. Reload Postfix: `postfix reload`
4. Cek log untuk error

### Error "unsupported dictionary type: hash"
Zimbra menggunakan LMDB, bukan hash. Gunakan:
```
sender_dependent_relayhost_maps = lmdb:/opt/zimbra/common/conf/sender_relay_restrictions
```

### Email internal tidak terkirim
1. Cek bahwa akun tujuan masih dalam domain m3bolt.com
2. Cek log untuk detail error
3. Pastikan konfigurasi mydestination masih benar

## Catatan Penting

1. **Format File:** Pastikan menggunakan TAB (bukan spasi) antara alamat email dan `[blackhole]`

2. **LMDB vs Hash:** Zimbra 8.8.15+ menggunakan LMDB untuk database Postfix, bukan hash

3. **Efek Samping:** Email yang diblokir akan menghasilkan bounce message ke sender

4. **Alternatif:** Untuk pemblokiran yang lebih halus (tanpa bounce), dapat menggunakan `smtp_discard_maps`

5. **Backup:** Selalu backup file konfigurasi sebelum melakukan perubahan:
   ```bash
   cp /opt/zimbra/common/conf/sender_relay_restrictions \
      /opt/zimbra/common/conf/sender_relay_restrictions.backup
   ```

## Referensi

- [Postfix Sender-Dependent Relayhost Maps](http://www.postfix.org/postconf.5.html#sender_dependent_relayhost_maps)
- [Postfix Discard Service](http://www.postfix.org/discard.8.html)
- [Zimbra Postfix Configuration](https://wiki.zimbra.com/wiki/Postfix)

## Riwayat Perubahan

| Tanggal | Perubahan | Oleh |
|---------|-----------|------|
| 2026-03-05 | Implementasi awal pemblokiran root@m3bolt.com | System Admin |
| 2026-03-05 | Update dokumentasi lengkap | System Admin |
