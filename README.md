# OpenBSD Home Server T400 Tutorial alpha 0.1 (Fast Version)

**Autor:** retrobofh – `id3fix@retro-technology.pl`
**Data:** 23.08.2025

---

## Spis treści
- [1. Statyczne IP](#1-statyczne-ip-na-serwerze)
- [2. Instalacja i konfiguracja HTTPD](#2-instalacja-i-konfiguracja-httpd)
- [3. HTTPS z ACME](#3-https-z-acme)
- [4. PF – firewall i ochrona przed DDoS](#4-pf--firewall-i-ochrona-przed-ddos)
- [5. DDNS – dynamiczne IP](#5-ddns--dynamiczne-ip)

---

## 1. Statyczne IP na serwerze

Edytuj `/etc/hostname.em0` (przykład dla interfejsu `em0`):

```sh
inet 192.168.1.100 255.255.255.0 192.168.1.1
```

- `192.168.1.100` → IP serwera  
- `192.168.1.1` → brama (Funbox)  

Restart interfejsu:

```sh
doas sh /etc/netstart em0
```

Sprawdź połączenie:

```sh
ping 192.168.1.1
ping 8.8.8.8
```

Dodaj DNS w `/etc/resolv.conf`:

```sh
nameserver 8.8.8.8
nameserver 8.8.4.4
```

**Funbox:**  
- Zaloguj się na `192.168.1.1` → Sieć → NAT/PAT  
- Dodaj porty: HTTPS (443) i HTTP (80)  
- Dodaj serwer do **DMZ** dopiero po skonfigurowaniu `httpd` i PF

---

## 2. Instalacja i konfiguracja HTTPD

1. Sprawdź, czy `httpd` jest zainstalowany:

```sh
pkg_info | grep httpd
```

2. Włącz usługę:

```sh
doas rcctl enable httpd
doas rcctl start httpd
```

3. Umieść pliki HTML w katalogu:

```
/var/www/htdocs/nazwa_strony
```

Przykład: `/var/www/htdocs/nazwa_strony/index.html`

4. Test w przeglądarce:

```
http://192.168.1.100
```

---

## 3. HTTPS z ACME

1. Zainstaluj `acme-client` - możliwe że jest od razu w OpenBSD:

```sh
doas pkg_add acme-client
```

2. Wygeneruj certyfikat:

```sh
doas acme-client -v twojadomena.ddnsgeek.com
```

3. Przeładuj `httpd`:

```sh
doas rcctl reload httpd
```

4. Automatyczne odnawianie certyfikatu:  
Edytuj crontab roota:

```sh
doas crontab -e
```

Dodaj linię:

```sh
0 3 * * * acme-client twojadomena.ddnsgeek.com && rcctl reload httpd
```

---

## 4. PF – firewall i ochrona przed DDoS

Przykładowy fragment `/etc/pf.conf`:

```pf
ext_if = "em0"
table <abuse> persist

set block-policy return
set skip on lo

block in all
block out all

# SSH tylko z zaufanego IP
trusted_host = "123.123.123.123"
pass in on $ext_if proto tcp from $trusted_host to ($ext_if) port 22 flags S/SA keep state

# HTTP / HTTPS
pass in on $ext_if proto tcp to ($ext_if) port {80,443} flags S/SA synproxy state (max-src-conn 40, max-src-conn-rate 20/5, overload <abuse> flush global)

pass out on $ext_if keep state
```

Test składni:

```sh
doas pfctl -nf /etc/pf.conf
```

Przeładuj reguły:

```sh
doas pfctl -f /etc/pf.conf
```

Sprawdzenie tabeli `abuse`:

```sh
doas pfctl -t abuse -T show
```

Logowanie tabeli abuse (`pf_abuse.sh`):

```sh
#!/bin/sh
LOGFILE="/var/log/pf_abuse.log"
DATE=$(date "+%Y-%m-%d %H:%M:%S")
echo "==== $DATE ====" >> $LOGFILE
doas pfctl -t abuse -T show >> $LOGFILE 2>&1
echo "" >> $LOGFILE
```

---

## 5. DDNS – dynamiczne IP

Konfiguracja wykonuje się na stronie [Dynu](https://www.dynu.com).  
W OpenBSD nie ma `gnudip` w pakietach, więc serwer korzysta tylko z `pkg`.

```sh
# Tutaj instalacja klienta DDNS według strony dostawcy.
```
