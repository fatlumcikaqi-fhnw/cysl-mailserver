# Mailserver-Dokumentation CyberLab

Diese Dokumentation beschreibt den Aufbau eines abgesicherten Mailservers für die Domain `u8.cyberlab.fhnw.ch`. Der Server wurde in einer Debian-13-VM betrieben und schrittweise mit DNS, Postfix, SPF und weiteren Mail-Sicherheitsmechanismen erweitert.

## Ausgangslage

Die VM wurde auf einem MacBook Air M4 mit UTM virtualisiert.

```text
Virtualisierung: UTM
Host-System: MacBook Air M4
Gast-System: Debian 13 Trixie ARM64
ISO: debian-13.4.0-arm64-netinst.iso
RAM: 4 GB
Disk: 30 GB
Netzwerk: CyberLab-Netzwerk / bridged
Benutzer: fatlum
```

Die Debian-Netinstall-ISO ist über die offizielle Debian-Webseite verfügbar:

[https://www.debian.org/releases/trixie/debian-installer/](https://www.debian.org/releases/trixie/debian-installer/)

Nach der Installation wurde die VM gestartet und die IP-Adresse mit `ip a` geprüft, damit der Zugriff per SSH möglich ist.

---

## Inhaltsverzeichnis

- [1 VM und Netzwerk](#1-vm-und-netzwerk)
- [2 DNS (BIND9)](#2-dns-bind9)
- [3 Mailserver (Postfix)](#3-mailserver-postfix)
- [4 SPF](#4-spf)
- [5 DKIM](#5-dkim)
- [6 DMARC](#6-dmarc)
- [7 Virenscan (Amavis, ClamAV)](#7-virenscan-amavis-clamav)
- [8 Greylisting](#8-greylisting)
- [9 Tarpit und Anti-UBE](#9-tarpit-und-anti-ube)
- [10 DNSSEC](#10-dnssec)
- [11 Spam-Filter (Bayes)](#11-spam-filter-bayes)
- [12 Backup](#12-backup)
- [13 Gesamttests und Abnahme](#13-gesamttests-und-abnahme)
- [14 Bewertung und produktiver Betrieb](#14-bewertung-und-produktiver-betrieb)
- [15 Quellen und KI-Nutzung](#15-quellen-und-ki-nutzung)

---

## 1 VM und Netzwerk

In diesem Kapitel wird die grundlegende System- und Netzwerkkonfiguration der VM beschrieben. Ziel ist eine stabile statische IP-Adresse, damit der Server später zuverlässig als DNS- und Mailserver erreichbar ist.

### 1.1 UTM: VM anlegen und Debian installieren

Die VM wurde mit UTM erstellt. Als Installationsmedium wurde die Debian-13-Trixie-ARM64-Netinstall-ISO verwendet. Während der Installation wurden 4 GB RAM, eine 30-GB-Festplatte und das CyberLab-Netzwerk als Netzwerkverbindung gewählt.

Nach der Installation wurde die ISO entfernt, damit die VM von der virtuellen Festplatte startet. Danach wurde mit folgendem Befehl die aktuelle Netzwerkkonfiguration geprüft:

```bash
# command
ip a
```

Die relevante Schnittstelle war in dieser Installation `enp0s1`.

### 1.2 Forward-Zone und Nameserver prüfen

Zuerst wurde geprüft, welche Nameserver für die Domain `u8.cyberlab.fhnw.ch` zuständig sind:

```bash
# command
dig +norecurse u8.cyberlab.fhnw.ch
```

Relevante Ausgabe:

```text
u8.cyberlab.fhnw.ch.      4800 IN NS ns2.u8.cyberlab.fhnw.ch.
u8.cyberlab.fhnw.ch.      4800 IN NS ns1.u8.cyberlab.fhnw.ch.
ns2.u8.cyberlab.fhnw.ch.  4800 IN A 192.168.97.65
ns1.u8.cyberlab.fhnw.ch.  4800 IN A 192.168.97.64
```

Damit ist ersichtlich, dass die Nameserver `ns1.u8.cyberlab.fhnw.ch` und `ns2.u8.cyberlab.fhnw.ch` für diese Zone vorgesehen sind.

Zusätzlich wurde die Reverse-DNS-Delegation geprüft:

```bash
# command
dig +norecurse -x 192.168.97.65
```

Relevante Ausgabe:

```text
65.97.168.192.in-addr.arpa.     IN CNAME 65.64/29.97.168.192.in-addr.arpa.
64/29.97.168.192.in-addr.arpa.  IN NS ns2.u8.cyberlab.fhnw.ch.
64/29.97.168.192.in-addr.arpa.  IN NS ns1.u8.cyberlab.fhnw.ch.
```

Wir verwalten damit die Reverse-Zone `64/29.97.168.192.in-addr.arpa` für das Subnetz `192.168.97.64 - 192.168.97.71`. Die VM wird trotzdem mit `/22` konfiguriert, damit sie im gesamten CyberLab-Netz kommunizieren kann.

### 1.3 Netzwerk und statische IP

Für den stabilen Betrieb als DNS- und Mailserver wurde die VM auf eine statische IP-Adresse umgestellt. Dafür wurde `systemd-networkd` verwendet.

Die Datei `/etc/systemd/network/10-enp0s1.network` wurde erstellt:

```bash
# command
sudo nano /etc/systemd/network/10-enp0s1.network
```

Inhalt:

```ini
[Match]
Name=enp0s1

[Network]
Address=192.168.97.64/22
Gateway=192.168.96.1
DNS=192.168.64.10
DNS=192.168.64.11
```

Danach wurde die alte Netzwerkverwaltung deaktiviert und `systemd-networkd` aktiviert. Dadurch übernimmt `systemd-networkd` die Verwaltung der Netzwerkschnittstelle `enp0s1`.

> **Achtung:** Dieser Schritt kann die bestehende SSH-Verbindung unterbrechen, weil die Netzwerkkonfiguration neu geladen wird. Danach muss man sich erneut über die statische IP-Adresse verbinden.

```bash
# command
sudo systemctl stop networking
sudo systemctl disable networking
sudo systemctl enable systemd-networkd
sudo systemctl restart systemd-networkd
```

Falls zusätzlich noch eine alte DHCP-Adresse vorhanden war, wurde die alte DHCP-Konfiguration in `/etc/network/interfaces` entfernt. Danach blieb dort nur noch die Loopback-Konfiguration:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback
```

Anschliessend wurden alte DHCP-Prozesse beendet und die Netzwerkkonfiguration neu geladen. Dadurch wird verhindert, dass das Interface zusätzlich zur statischen IP-Adresse weiterhin eine alte DHCP-Adresse behält.

> **Achtung:** Auch dieser Schritt kann die SSH-Verbindung kurz unterbrechen, weil alte Netzwerkkonfigurationen beendet und `systemd-networkd` neu gestartet wird.

```bash
# command
sudo systemctl stop ifup@enp0s1.service
sudo pkill dhcpcd
sudo systemctl restart systemd-networkd
```

Die Verbindung kann danach über die statische IP hergestellt werden:

```bash
# command
ssh fatlum@192.168.97.64
```

Für die Namensauflösung wurde zusätzlich `systemd-resolved` verwendet. Nach der Umstellung auf `systemd-networkd` kann es vorkommen, dass die IP-Konfiguration korrekt ist, aber DNS-Abfragen trotzdem nicht funktionieren. Ein typisches Symptom wäre zum Beispiel, dass `ping 8.8.8.8` funktioniert, aber `ping google.com` fehlschlägt.

Falls DNS nicht mehr funktioniert, wird zuerst temporär ein Nameserver in `/etc/resolv.conf` eingetragen. Dadurch kann `apt` wieder Pakete auflösen und `systemd-resolved` installiert werden:

```bash
# command
echo "nameserver 192.168.64.10" | sudo tee /etc/resolv.conf
```

Danach wird systemd-resolved installiert und aktiviert:

```bash
# command
sudo apt update
sudo apt install -y systemd-resolved
sudo systemctl enable --now systemd-resolved
```

Zum Schluss wird /etc/resolv.conf auf die Resolver-Datei von systemd verlinkt:

```bash
# command
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Dadurch verwenden Programme, die DNS über /etc/resolv.conf abfragen, den lokalen Resolver von systemd-resolved. Dieser nutzt die DNS-Server, die in der systemd-networkd-Konfiguration hinterlegt wurden.

Test:

```bash
# command
ping -c 3 google.com
```

Wenn der Ping funktioniert, sind statische IP und Namensauflösung korrekt eingerichtet.

### 1.4 Zertifikate

Für diese Laborumgebung wurden keine zusätzlichen Zertifikate eingerichtet. Die geforderten Sicherungsmassnahmen beziehen sich auf DNS, Postfix, SPF, DKIM, DMARC, Greylisting, DNSSEC sowie Viren- und Spamfilterung.

---

## 2 DNS (BIND9)

In diesem Kapitel wird BIND9 als autoritativer DNS-Server für `u8.cyberlab.fhnw.ch` eingerichtet. Dafür werden eine Forward-Zone, eine Reverse-Zone und mehrere Tests erstellt.

### 2.1 Installation und Basis

Zuerst wurde geprüft, ob BIND9 bereits installiert ist:

```bash
# command
dpkg -l | grep -E "bind9|dnsutils"
```

Da der eigentliche BIND9-Server noch fehlte, wurde er installiert:

```bash
# command
sudo apt install -y bind9
```

Der Dienst wurde danach geprüft:

```bash
# command
systemctl is-active bind9

# response
active
```

Zusätzlich wurde die lokale Host-Auflösung angepasst, damit `mail.u8.cyberlab.fhnw.ch` auf die statische Server-IP zeigt:

```bash
# command
sudo nano /etc/hosts
```

Inhalt:

```text
127.0.0.1 localhost
192.168.97.64 mail.u8.cyberlab.fhnw.ch mail

::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### 2.2 Zonenreferenz und Dateipfade

Für die Zonendateien wurde ein eigenes Verzeichnis erstellt:

```bash
# command
sudo mkdir -p /var/lib/bind/u8.cyberlab.fhnw.ch
sudo chown -R bind:bind /var/lib/bind/u8.cyberlab.fhnw.ch
sudo touch /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db /var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db
sudo chown bind:bind /var/lib/bind/u8.cyberlab.fhnw.ch/*.db
```

Danach wurden die Zonen in `/etc/bind/named.conf.local` eingetragen:

```bash
# command
sudo nano /etc/bind/named.conf.local
```

Inhalt:

```conf
zone "u8.cyberlab.fhnw.ch" {
    type master;
    file "/var/lib/bind/u8.cyberlab.fhnw.ch/forward.db";
};

zone "64/29.97.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db";
};
```

Die BIND-Konfiguration wurde geprüft:

```bash
# command
sudo named-checkconf
```

Keine Ausgabe bedeutet, dass die Konfiguration syntaktisch korrekt ist.

### 2.3 Forward-Zone

Die Forward-Zone bildet Hostnamen auf IP-Adressen ab.

Da die Delegation in Kapitel 1.2 sowohl `ns1` als auch `ns2` vorsieht, werden beide Nameserver in der Zone eingetragen. Zusätzlich erhält `ns2` einen eigenen A-Record auf `192.168.97.65`.

```bash
# command
sudo nano /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db
```

Inhalt:

```dns
$TTL 86400
@   IN SOA ns1.u8.cyberlab.fhnw.ch. admin.u8.cyberlab.fhnw.ch. (
        2026051601 ; serial
        3600       ; refresh
        1800       ; retry
        604800     ; expire
        86400      ; negative cache
)

; Nameserver
@    IN NS ns1.u8.cyberlab.fhnw.ch.
@    IN NS ns2.u8.cyberlab.fhnw.ch.

; Hosts
ns1  IN A 192.168.97.64
ns2  IN A 192.168.97.65
mail IN A 192.168.97.64

; Mail exchange
@    IN MX 10 mail.u8.cyberlab.fhnw.ch.
```

Prüfung:

```bash
# command
sudo named-checkzone u8.cyberlab.fhnw.ch /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
zone u8.cyberlab.fhnw.ch/IN: loaded serial 2026051601
OK
```

### 2.4 Reverse-Zone

Die Reverse-Zone bildet die IP-Adresse zurück auf den Hostnamen ab.

Auch in der Reverse-Zone werden beide autoritativen Nameserver eingetragen, damit Forward- und Reverse-Zone dieselbe Nameserver-Struktur veröffentlichen.

```bash
# command
sudo nano /var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db
```

Inhalt:

```dns
$TTL 86400
@   IN SOA ns1.u8.cyberlab.fhnw.ch. admin.u8.cyberlab.fhnw.ch. (
        2026051601 ; serial
        3600       ; refresh
        1800       ; retry
        604800     ; expire
        86400      ; negative cache
)

; Nameserver
@   IN NS ns1.u8.cyberlab.fhnw.ch.
@   IN NS ns2.u8.cyberlab.fhnw.ch.

; PTR Records
64  IN PTR mail.u8.cyberlab.fhnw.ch.
```

Prüfung:

```bash
# command
sudo named-checkzone 64/29.97.168.192.in-addr.arpa /var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db

# response
zone 64/29.97.168.192.in-addr.arpa/IN: loaded serial 2026051601
OK
```

Danach wurde BIND9 neu gestartet:

```bash
# command
sudo systemctl restart bind9
```

### 2.5 Tests

Die DNS-Funktion wurde lokal und über den CyberLab-DNS getestet.

Die autoritativen Nameserver der Forward-Zone wurden geprüft:

```bash
# command
dig @127.0.0.1 u8.cyberlab.fhnw.ch NS +short

# response
ns1.u8.cyberlab.fhnw.ch.
ns2.u8.cyberlab.fhnw.ch.
```

Der zusätzliche A-Record für `ns2` wurde ebenfalls geprüft:

```bash
# command
dig @127.0.0.1 ns2.u8.cyberlab.fhnw.ch A +short

# response
192.168.97.65
```

Die Nameserver der Reverse-Zone wurden ebenfalls geprüft:

```bash
# command
dig @127.0.0.1 64/29.97.168.192.in-addr.arpa NS +short

# response
ns1.u8.cyberlab.fhnw.ch.
ns2.u8.cyberlab.fhnw.ch.
```

Forward Lookup über den lokalen BIND-Server:

```bash
# command
dig @127.0.0.1 mail.u8.cyberlab.fhnw.ch +short

# response
192.168.97.64
```

Reverse Lookup direkt auf die eigene Reverse-Zone:

```bash
# command
dig @127.0.0.1 64.64/29.97.168.192.in-addr.arpa PTR +short

# response
mail.u8.cyberlab.fhnw.ch.
```

Normale Reverse-Auflösung:

```bash
# command
dig -x 192.168.97.64 +short

# response
64.64/29.97.168.192.in-addr.arpa.
mail.u8.cyberlab.fhnw.ch.
```

MX-Record über die echte Server-IP:

```bash
# command
dig @192.168.97.64 u8.cyberlab.fhnw.ch MX +short

# response
10 mail.u8.cyberlab.fhnw.ch.
```

Forward Lookup über den CyberLab-DNS:

```bash
# command
dig @192.168.64.10 mail.u8.cyberlab.fhnw.ch +short

# response
192.168.97.64
```

MX-Record über den CyberLab-DNS:

```bash
# command
dig @192.168.64.10 u8.cyberlab.fhnw.ch MX +short

# response
10 mail.u8.cyberlab.fhnw.ch.
```

Reverse Lookup über den CyberLab-DNS:

```bash
# command
dig @192.168.64.10 -x 192.168.97.64 +short

# response
64.64/29.97.168.192.in-addr.arpa.
mail.u8.cyberlab.fhnw.ch.
```

Damit funktionieren Forward Lookup, Reverse Lookup und MX Lookup sowohl lokal als auch über den CyberLab-DNS.

---

## 3 Mailserver (Postfix)

In diesem Kapitel wird Postfix als Mailserver für `u8.cyberlab.fhnw.ch` eingerichtet. Danach wird geprüft, ob lokale Zustellung sowie Versand und Empfang über den CyberLab-Reflector funktionieren.

### 3.1 Installation

Zuerst wurde geprüft, ob Postfix, `swaks` und `rsyslog` bereits installiert sind:

```bash
# command
dpkg -l | grep -E "postfix|swaks|rsyslog"
```

Da die benötigten Pakete noch nicht vorhanden waren, wurden sie installiert:

```bash
# command
sudo apt install -y postfix swaks rsyslog
```

Während der Installation wurde Postfix als `Internet Site` eingerichtet. Als System-Mail-Name wurde `u8.cyberlab.fhnw.ch` verwendet.

Danach wurden die wichtigsten Postfix-Werte gesetzt:

```bash
# command
sudo postconf -e "myhostname=mail.u8.cyberlab.fhnw.ch" && \
sudo postconf -e "mydomain=u8.cyberlab.fhnw.ch" && \
sudo postconf -e "myorigin=/etc/mailname" && \
sudo postconf -e 'mydestination = $myhostname, $mydomain, localhost.$mydomain, localhost' && \
echo "u8.cyberlab.fhnw.ch" | sudo tee /etc/mailname && \
sudo postconf -e "smtpd_banner=mail.u8.cyberlab.fhnw.ch ESMTP Postfix" && \
sudo postconf -e "inet_protocols=ipv4" && \
sudo systemctl restart postfix
```

Die gesetzten Werte wurden geprüft:

```bash
# command
sudo postconf -n | grep -E "^(myhostname|mydomain|myorigin|mydestination|inet_protocols|smtpd_banner)"
cat /etc/mailname
```

Relevante Ausgabe:

```text
inet_protocols = ipv4
mydestination = $myhostname, $mydomain, localhost.$mydomain, localhost
mydomain = u8.cyberlab.fhnw.ch
myhostname = mail.u8.cyberlab.fhnw.ch
myorigin = /etc/mailname
smtpd_banner = mail.u8.cyberlab.fhnw.ch ESMTP Postfix
u8.cyberlab.fhnw.ch
```

Der System-Hostname wurde ebenfalls passend gesetzt:

```bash
# command
sudo hostnamectl set-hostname mail.u8.cyberlab.fhnw.ch
hostname -f

# response
mail.u8.cyberlab.fhnw.ch
```

Postfix wurde anschliessend geprüft:

```bash
# command
systemctl is-active postfix

# response
active
```

### 3.2 Konfiguration und Smoke-Tests

Zuerst wurde geprüft, ob Postfix lokal auf Port 25 erreichbar ist und den korrekten SMTP-Banner ausgibt:

```bash
# command
nc -v 127.0.0.1 25

# response
localhost [127.0.0.1] 25 (smtp) open
220 mail.u8.cyberlab.fhnw.ch ESMTP Postfix
QUIT
221 2.0.0 Bye
```

Der Test wurde über `127.0.0.1` durchgeführt, damit zuerst nur der lokale Postfix-Dienst geprüft wird.

Danach wurde kontrolliert, ob `rsyslog` aktiv ist und Mail-Logs geschrieben werden:

```bash
# command
systemctl is-active rsyslog
ls -l /var/log/mail.log
```

Relevante Ausgabe:

```text
active
-rw-r----- 1 root adm ... /var/log/mail.log
```

Lokale Mailzustellung:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to fatlum@u8.cyberlab.fhnw.ch --server 127.0.0.1
```

Relevante Ausgabe:

```text
250 2.0.0 Ok: queued as CBFBF407BE
```

Prüfung der lokalen Mailbox:

```bash
# command
sudo tail -n 40 /var/mail/fatlum
```

Relevante Ausgabe:

```text
Delivered-To: fatlum@u8.cyberlab.fhnw.ch
From: fatlum@u8.cyberlab.fhnw.ch
Subject: test Sat, 16 May 2026 16:11:48 +0200
This is a test mailing
```

Danach wurde eine Testmail an den CyberLab-Reflector gesendet:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to reflector@cyberlab.fhnw.ch --server 127.0.0.1
```

Relevante Ausgabe:

```text
250 2.0.0 Ok: queued as 21957407BE
```

Im Mail-Log wurde geprüft, ob die Mail erfolgreich weitergegeben wurde:

```bash
# command
sudo grep "21957407BE" /var/log/mail.log
```

Relevante Ausgabe:

```text
to=<reflector@cyberlab.fhnw.ch>, relay=srvMSG01.cyberlab.fhnw.ch[192.168.64.33]:25, dsn=2.0.0, status=sent
```

Die Antwort des Reflectors wurde in der lokalen Mailbox geprüft:

```bash
# command
sudo tail -n 80 /var/mail/fatlum
```

Relevante Ausgabe:

```text
From: reflector@cyberlab.fhnw.ch
Subject: reflected: test Sat, 16 May 2026 16:13:25 +0200

This is a message from reflector
I received the attached mail
```

Zum Schluss wurde geprüft, ob die Mailqueue leer ist:

```bash
# command
mailq

# response
Mail queue is empty
```

Damit ist die Postfix-Grundkonfiguration abgeschlossen.

---

## 4 SPF

In diesem Kapitel wird SPF (Sender Policy Framework) für `u8.cyberlab.fhnw.ch` eingerichtet. SPF legt fest, welche Server E-Mails im Namen einer Domain versenden dürfen. Dadurch können empfangende Mailserver Spoofing-Versuche besser erkennen.

Die Konfiguration besteht aus zwei Teilen: Zuerst wird ein SPF-TXT-Record in der DNS-Zone veröffentlicht. Danach wird Postfix so erweitert, dass eingehende Mails ebenfalls anhand von SPF geprüft werden.

### 4.1 Ausgehend

Für ausgehende Mails wird ein SPF-TXT-Record in der Forward-Zone ergänzt. Vor jeder Änderung an einer DNS-Zonendatei muss die Serial im SOA-Record erhöht werden. Dadurch erkennt BIND, dass eine neue Version der Zone vorliegt.

In der Datei `/var/lib/bind/u8.cyberlab.fhnw.ch/forward.db` stand zuerst folgende Serial:

```dns
2026051601 ; serial
```

Diese wurde auf `2026051602` erhöht. Gleichzeitig wurde am Ende der Datei der SPF-Record ergänzt:

```bash
# command
sudo sh -c 'sed -i "s/2026051601/2026051602/" /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db && printf "\n; SPF\n@    IN TXT \"v=spf1 mx -all\"\n" >> /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db'
```

Der Befehl macht zwei Dinge:

1. `sed -i "s/2026051601/2026051602/" ...` ersetzt die alte Serial durch die neue Serial.
2. `printf ... >> forward.db` fügt den SPF-TXT-Record am Ende der Zonendatei hinzu.

Der relevante Teil der Zone sieht danach so aus:

```dns
@    IN MX 10 mail.u8.cyberlab.fhnw.ch.

; SPF
@    IN TXT "v=spf1 mx -all"
```

Der Eintrag `v=spf1 mx -all` bedeutet:

- `v=spf1`: Es handelt sich um einen SPF-Record.
- `mx`: Server, die im MX-Record der Domain stehen, dürfen Mails für diese Domain senden.
- `-all`: Alle anderen Server sind nicht erlaubt.

Danach wurde die Zone geprüft:

```bash
# command
sudo named-checkzone u8.cyberlab.fhnw.ch /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
zone u8.cyberlab.fhnw.ch/IN: loaded serial 2026051602
OK
```

Anschliessend wurde BIND neu geladen:

```bash
# command
sudo rndc reload

# response
server reload successful
```

Der SPF-TXT-Record wurde über DNS geprüft:

```bash
# command
dig u8.cyberlab.fhnw.ch TXT +short

# response
"v=spf1 mx -all"
```

Damit ist SPF für ausgehende Mails veröffentlicht.

### 4.2 Eingehend

Damit unser Mailserver eingehende Mails anhand von SPF prüfen kann, wurde `postfix-policyd-spf-perl` installiert:

```bash
# command
sudo apt install -y postfix-policyd-spf-perl
```

Danach wurde geprüft, ob der SPF-Policy-Dienst bereits in `/etc/postfix/master.cf` vorhanden ist:

```bash
# command
grep -n "policyd-spf" /etc/postfix/master.cf
```

Da keine Ausgabe erschien, wurde der Dienst ergänzt:

```bash
# command
sudo sh -c 'printf "\npolicyd-spf  unix  -       n       n       -       0       spawn\n    user=policyd-spf argv=/usr/sbin/postfix-policyd-spf-perl\n" >> /etc/postfix/master.cf'
```

Relevanter Eintrag in `/etc/postfix/master.cf`:

```text
policyd-spf  unix  -       n       n       -       0       spawn
    user=policyd-spf argv=/usr/sbin/postfix-policyd-spf-perl
```

Damit kennt Postfix den SPF-Policy-Dienst. Zusätzlich muss Postfix aber noch angewiesen werden, diesen Dienst bei eingehenden Mails zu verwenden. Dazu wurde `smtpd_recipient_restrictions` gesetzt:

```bash
# command
sudo postconf -e "smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, check_policy_service unix:private/policyd-spf, reject_unauth_destination"
sudo postconf -n smtpd_recipient_restrictions
```

Relevante Ausgabe:

```text
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, check_policy_service unix:private/policyd-spf, reject_unauth_destination
```

Die Reihenfolge bedeutet:

- `permit_mynetworks`: lokale vertrauenswürdige Netze werden erlaubt.
- `permit_sasl_authenticated`: authentifizierte Benutzer werden erlaubt.
- `check_policy_service unix:private/policyd-spf`: SPF-Prüfung wird ausgeführt.
- `reject_unauth_destination`: fremde Ziel-Domains werden abgelehnt, wenn der Server dafür nicht zuständig ist.

Danach wurde Postfix neu gestartet:

```bash
# command
sudo systemctl restart postfix
```

Kontrolle:

```bash
# command
systemctl is-active postfix

# response
active
```

### 4.3 Tests

Zum Testen wurde eine Mail an den CyberLab-Reflector gesendet. Der Reflector sendet die Mail zurück, wodurch die eingehende SPF-Prüfung getestet werden kann.

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to reflector@cyberlab.fhnw.ch --server 127.0.0.1
```

Relevante Ausgabe:

```text
250 2.0.0 Ok: queued as CB7B44074A
```

Danach wurde im Mail-Log nach der Queue-ID und nach SPF gesucht:

```bash
# command
sudo grep -Ei "CB7B44074A|spf|policyd" /var/log/mail.log | tail -n 80
```

Relevante Ausgabe:

```text
CB7B44074A: to=<reflector@cyberlab.fhnw.ch>, relay=srvMSG01.cyberlab.fhnw.ch[192.168.64.33]:25, dsn=2.0.0, status=sent
postfix/policy-spf[2663]: Policy action=PREPEND Received-SPF: pass (cyberlab.fhnw.ch: 192.168.64.33 is authorized to use 'reflector@cyberlab.fhnw.ch' in 'mfrom' identity)
```

Wichtig ist:

```text
Received-SPF: pass
```

Das bedeutet, dass der SPF-Policy-Dienst aktiv war und die eingehende Mail vom Reflector erfolgreich geprüft wurde.

Zusätzlich wurde die lokale Mailbox geprüft:

```bash
# command
sudo grep -Ei "Received-SPF|Subject: reflected|From: reflector|Delivered-To:" /var/mail/fatlum | tail -n 30
```

Relevante Ausgabe:

```text
Delivered-To: fatlum@u8.cyberlab.fhnw.ch
Received-SPF: pass (cyberlab.fhnw.ch: 192.168.64.33 is authorized to use 'reflector@cyberlab.fhnw.ch' in 'mfrom' identity)
Subject: reflected: test Sat, 16 May 2026 16:42:15 +0200
From: reflector@cyberlab.fhnw.ch
```

Damit ist bestätigt, dass SPF sowohl im DNS veröffentlicht als auch bei eingehenden Mails durch Postfix geprüft wird.

---

## 5 DKIM

In diesem Kapitel wird DKIM (DomainKeys Identified Mail) für `u8.cyberlab.fhnw.ch` eingerichtet. DKIM signiert ausgehende E-Mails kryptografisch, damit empfangende Mailserver prüfen können, ob die Mail wirklich von unserer Domain stammt und unterwegs nicht verändert wurde.

Dafür wird OpenDKIM installiert, mit Postfix über einen Milter verbunden und ein öffentlicher DKIM-Schlüssel als DNS-TXT-Record veröffentlicht. Anschliessend wird geprüft, ob ausgehende Mails korrekt signiert und vom CyberLab-Reflector erfolgreich verifiziert werden.

### 5.1 OpenDKIM und Postfix (Milter)

Zuerst wurde geprüft, ob OpenDKIM bereits installiert ist:

```bash
# command
dpkg -l | grep -E "^ii\s+opendkim|^ii\s+opendkim-tools" || true

# response
# keine Ausgabe
```

Da OpenDKIM noch nicht installiert war, wurden die benötigten Pakete installiert:

```bash
# command
sudo apt install -y opendkim opendkim-tools
```

Danach wurde geprüft, ob der Dienst läuft:

```bash
# command
systemctl is-active opendkim

# response
active
```

Anschliessend wurde die OpenDKIM-Konfiguration geprüft:

```bash
# command
sudo grep -nE "^(#)?(Mode|Socket|KeyTable|SigningTable|ExternalIgnoreList|InternalHosts|Syslog|UserID)" /etc/opendkim.conf /etc/default/opendkim 2>/dev/null
```

Relevante Ausgabe:

```text
/etc/opendkim.conf:14:#Mode          sv
/etc/opendkim.conf:30:UserID         opendkim
/etc/opendkim.conf:37:Socket         local:/run/opendkim/opendkim.sock
/etc/opendkim.conf:46:#InternalHosts 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12
```

Die Standardkonfiguration verwendet noch einen lokalen Socket. Für die Anbindung an Postfix wurde OpenDKIM so konfiguriert, dass es über `inet:12301@localhost` erreichbar ist. Zusätzlich wurden Tabellen für Schlüssel, Signaturen und vertrauenswürdige Hosts eingebunden.

```bash
# command
sudo nano /etc/opendkim.conf
```

Relevante Konfiguration:

```text
Mode            sv
Socket          inet:12301@localhost
KeyTable        refile:/etc/opendkim/KeyTable
SigningTable    refile:/etc/opendkim/SigningTable
ExternalIgnoreList refile:/etc/opendkim/TrustedHosts
InternalHosts   refile:/etc/opendkim/TrustedHosts
```

Prüfung:

```bash
# command
sudo grep -nE "^(Mode|Socket|KeyTable|SigningTable|ExternalIgnoreList|InternalHosts)" /etc/opendkim.conf

# response
47:Mode            sv
48:Socket          inet:12301@localhost
49:KeyTable        refile:/etc/opendkim/KeyTable
50:SigningTable    refile:/etc/opendkim/SigningTable
51:ExternalIgnoreList refile:/etc/opendkim/TrustedHosts
52:InternalHosts   refile:/etc/opendkim/TrustedHosts
```

Danach wurde ein Verzeichnis für die DKIM-Schlüssel erstellt:

```bash
# command
sudo mkdir -p /etc/opendkim/keys/u8.cyberlab.fhnw.ch
```

Der DKIM-Schlüssel wurde mit dem Selector `default` für die Domain `u8.cyberlab.fhnw.ch` erzeugt:

```bash
# command
sudo opendkim-genkey -b 2048 -s default -d u8.cyberlab.fhnw.ch -D /etc/opendkim/keys/u8.cyberlab.fhnw.ch
```

Danach wurden Besitzer und Berechtigungen gesetzt:

```bash
# command
sudo chown -R opendkim:opendkim /etc/opendkim/keys
sudo chmod 700 /etc/opendkim/keys
sudo chmod 700 /etc/opendkim/keys/u8.cyberlab.fhnw.ch
sudo chmod 600 /etc/opendkim/keys/u8.cyberlab.fhnw.ch/default.private
```

Prüfung:

```bash
# command
sudo ls -l /etc/opendkim/keys/u8.cyberlab.fhnw.ch

# response
-rw------- 1 opendkim opendkim 1704 May 16 17:51 default.private
-rw------- 1 opendkim opendkim  515 May 16 17:51 default.txt
```

Die Datei `default.private` enthält den privaten Schlüssel und bleibt lokal auf dem Server. Die Datei `default.txt` enthält den öffentlichen DNS-TXT-Record.

Nun wurden die OpenDKIM-Tabellen erstellt.

```bash
# command
sudo nano /etc/opendkim/KeyTable
```

Inhalt:

```text
default._domainkey.u8.cyberlab.fhnw.ch u8.cyberlab.fhnw.ch:default:/etc/opendkim/keys/u8.cyberlab.fhnw.ch/default.private
```

```bash
# command
sudo nano /etc/opendkim/SigningTable
```

Inhalt:

```text
*@u8.cyberlab.fhnw.ch default._domainkey.u8.cyberlab.fhnw.ch
```

```bash
# command
sudo nano /etc/opendkim/TrustedHosts
```

Inhalt:

```text
127.0.0.1
localhost
192.168.97.64
mail.u8.cyberlab.fhnw.ch
u8.cyberlab.fhnw.ch
```

Danach wurden Besitzer und Berechtigungen gesetzt:

```bash
# command
sudo chown opendkim:opendkim /etc/opendkim/KeyTable
sudo chown opendkim:opendkim /etc/opendkim/SigningTable
sudo chown opendkim:opendkim /etc/opendkim/TrustedHosts
sudo chmod 644 /etc/opendkim/KeyTable
sudo chmod 644 /etc/opendkim/SigningTable
sudo chmod 644 /etc/opendkim/TrustedHosts
```

Anschliessend wurde geprüft, ob Postfix bereits Milter-Einstellungen besitzt:

```bash
# command
sudo postconf -n | grep -E "^(milter_protocol|milter_default_action|smtpd_milters|non_smtpd_milters)"

# response
# keine Ausgabe
```

Da noch keine Milter konfiguriert waren, wurde OpenDKIM in Postfix eingebunden:

```bash
# command
sudo postconf -e "milter_protocol = 6"
sudo postconf -e "milter_default_action = accept"
sudo postconf -e "smtpd_milters = inet:localhost:12301"
sudo postconf -e "non_smtpd_milters = inet:localhost:12301"
```

Prüfung:

```bash
# command
sudo postconf -n | grep -E "^(milter_protocol|milter_default_action|smtpd_milters|non_smtpd_milters)"

# response
milter_default_action = accept
milter_protocol = 6
non_smtpd_milters = inet:localhost:12301
smtpd_milters = inet:localhost:12301
```

Zum Schluss wurden OpenDKIM und Postfix neu gestartet:

```bash
# command
sudo systemctl restart opendkim
sudo systemctl restart postfix
```

Kontrolle:

```bash
# command
systemctl is-active opendkim
systemctl is-active postfix

# response
active
active
```

Damit ist OpenDKIM eingerichtet und als Milter mit Postfix verbunden.

### 5.2 DKIM-Schlüssel in der DNS-Zone

Damit andere Mailserver die DKIM-Signatur prüfen können, muss der öffentliche DKIM-Schlüssel als DNS-TXT-Record veröffentlicht werden. Der öffentliche Record befindet sich in der Datei `default.txt`.

```bash
# command
sudo cat /etc/opendkim/keys/u8.cyberlab.fhnw.ch/default.txt
```

Relevante Ausgabe:

```text
default._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A..."
) ; ----- DKIM key default for u8.cyberlab.fhnw.ch
```

Der Record wird in die Forward-Zone eingefügt. Vor der Änderung wird die Serial im SOA-Record von `2026051602` auf `2026051603` erhöht, damit BIND die neue Zonenversion erkennt.

```bash
# command
sudo nano /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db
```

Relevante Änderung:

```dns
@    IN TXT "v=spf1 mx -all"

; DKIM
default._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
    "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A..."
) ; ----- DKIM key default for u8.cyberlab.fhnw.ch
```

Die Serial im SOA-Record wurde ebenfalls erhöht:

```dns
2026051603 ; serial
```

Danach wurde die Zone geprüft:

```bash
# command
sudo named-checkzone u8.cyberlab.fhnw.ch /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
zone u8.cyberlab.fhnw.ch/IN: loaded serial 2026051603
OK
```

Anschliessend wurde BIND neu geladen:

```bash
# command
sudo rndc reload

# response
server reload successful
```

Danach wurde geprüft, ob der DKIM-TXT-Record über DNS sichtbar ist:

```bash
# command
dig default._domainkey.u8.cyberlab.fhnw.ch TXT +short
```

Relevante Ausgabe:

```text
"v=DKIM1; h=sha256; k=rsa; " "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A..."
```

Dass der TXT-Record in mehrere Anführungszeichen-Blöcke aufgeteilt wird, ist normal bei langen DNS-TXT-Records.

Zusätzlich wurde geprüft, ob OpenDKIM den privaten Schlüssel mit dem öffentlichen DNS-Record abgleichen kann:

```bash
# command
sudo opendkim-testkey -d u8.cyberlab.fhnw.ch -s default -vvv

# response
opendkim-testkey: using default configfile /etc/opendkim.conf
opendkim-testkey: checking key 'default._domainkey.u8.cyberlab.fhnw.ch'
opendkim-testkey: key not secure
opendkim-testkey: key OK
```

Die Ausgabe `key OK` zeigt, dass privater Schlüssel und DNS-Record zusammenpassen. Die Meldung `key not secure` bedeutet nur, dass der DNS-Record zu diesem Zeitpunkt noch nicht durch DNSSEC abgesichert ist. DNSSEC wird später separat eingerichtet.

### 5.3 Tests

Zum Testen wurde eine Mail an den CyberLab-Reflector gesendet:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to reflector@cyberlab.fhnw.ch --server 127.0.0.1
```

Relevante Ausgabe:

```text
250 2.0.0 Ok: queued as A56CB4076E
```

Danach wurde im Mail-Log geprüft, ob OpenDKIM die Mail signiert hat:

```bash
# command
sudo grep -Ei "A56CB4076E|dkim|opendkim" /var/log/mail.log | tail -n 80
```

Relevante Ausgabe:

```text
opendkim[3233]: A56CB4076E: DKIM-Signature field added (s=default, d=u8.cyberlab.fhnw.ch)
postfix/smtp[3279]: A56CB4076E: to=<reflector@cyberlab.fhnw.ch>, relay=srvMSG01.cyberlab.fhnw.ch[192.168.64.33]:25, dsn=2.0.0, status=sent
```

Die Zeile `DKIM-Signature field added` zeigt, dass OpenDKIM die ausgehende Mail erfolgreich signiert hat.

In der zurückgesendeten Reflector-Mail wurde zusätzlich geprüft, ob die DKIM-Signatur vom CyberLab-Mailserver akzeptiert wurde.

```bash
# command
sudo grep -Ei "Authentication-Results|DKIM-Signature|Received-SPF|Subject: reflected|From: reflector" /var/mail/fatlum | tail -n 20
```

Relevante Ausgabe:

```text
From: reflector@cyberlab.fhnw.ch
Subject: reflected: test Sat, 16 May 2026 17:59:23 +0200
Authentication-Results: srvmsg01.cyberlab.fhnw.ch (amavisd-new); dkim=pass (2048-bit key) header.d=u8.cyberlab.fhnw.ch
DKIM-Signature: v=1; a=rsa-sha256; d=u8.cyberlab.fhnw.ch; s=default;
```

Die Ausgabe `dkim=pass` bestätigt, dass die DKIM-Signatur erfolgreich geprüft wurde.

Zum Schluss wurde kontrolliert, ob keine Mail mehr in der Queue hängt:

```bash
# command
mailq

# response
Mail queue is empty
```

Damit ist DKIM erfolgreich eingerichtet. Ausgehende Mails werden durch OpenDKIM signiert und die Signatur kann über den veröffentlichten DNS-TXT-Record geprüft werden.

---

## 6 DMARC

In diesem Kapitel wird DMARC (Domain-based Message Authentication, Reporting and Conformance) eingerichtet. DMARC baut auf SPF und DKIM auf und legt fest, wie empfangende Mailserver mit Nachrichten umgehen sollen, die diese Prüfungen nicht bestehen. Zusätzlich kann DMARC Berichte erzeugen, damit sichtbar wird, ob fremde Server versuchen, E-Mails im Namen unserer Domain zu versenden.

Für die Umsetzung wird OpenDMARC als Milter in Postfix eingebunden. Dadurch prüft unser Mailserver eingehende E-Mails auf DMARC und schreibt das Ergebnis in die Mail-Header und in das Mail-Log.

### 6.1 Konfiguration

Zuerst wurde geprüft, ob OpenDMARC bereits installiert ist:

```bash
# command
dpkg-query -W -f='${binary:Package} ${Version}\n' opendmarc 2>/dev/null
```

Da keine Ausgabe erschien, war OpenDMARC noch nicht installiert. Deshalb wurde das Paket installiert:

```bash
# command
sudo apt-get install -y opendmarc
```

Während der Installation wurde die optionale Datenbankeinrichtung nicht verwendet. Für diese Laborumgebung wird OpenDMARC nur als Milter für Postfix benötigt.

Danach wurde geprüft, ob der Dienst läuft:

```bash
# command
systemctl is-active opendmarc

# response
active
```

Anschliessend wurde die bestehende OpenDMARC-Konfiguration geprüft:

```bash
# command
sudo grep -E '^(Socket|AuthservID|TrustedAuthservIDs|IgnoreHosts|HistoryFile)' /etc/opendmarc.conf

# response
Socket local:/run/opendmarc/opendmarc.sock
```

Standardmässig verwendete OpenDMARC also einen lokalen Unix-Socket. Für die Postfix-Anbindung wurde OpenDMARC auf einen TCP-Socket auf `localhost` mit Port `12302` umgestellt.

Vor der Anpassung wurde die ursprüngliche Konfiguration gesichert:

```bash
# command
sudo cp /etc/opendmarc.conf /etc/opendmarc.conf.bak-before-dmarc
```

Danach wurde `/etc/opendmarc.conf` angepasst. Die relevante Konfiguration sieht danach so aus:

```bash
# command
sudo grep -nE '^(AuthservID|Socket|SPFSelfValidate|HistoryFile|IgnoreHosts|TrustedAuthservIDs|Syslog)' /etc/opendmarc.conf

# response
115:AuthservID mail.u8.cyberlab.fhnw.ch
116:Socket inet:12302@localhost
117:SPFSelfValidate yes
118:HistoryFile /var/run/opendmarc/opendmarc.dat
119:IgnoreHosts /etc/opendmarc/ignore.hosts
120:TrustedAuthservIDs mail.u8.cyberlab.fhnw.ch
121:Syslog true
```

Die wichtigsten Werte sind:

- `AuthservID mail.u8.cyberlab.fhnw.ch`: Der eigene Mailserver wird als prüfender Authentifizierungsdienst angegeben.
- `Socket inet:12302@localhost`: OpenDMARC lauscht lokal auf Port `12302`.
- `SPFSelfValidate yes`: OpenDMARC darf SPF selbst prüfen, falls nötig.
- `IgnoreHosts /etc/opendmarc/ignore.hosts`: Bestimmte lokale Hosts werden von der DMARC-Prüfung ausgenommen.
- `TrustedAuthservIDs mail.u8.cyberlab.fhnw.ch`: Authentifizierungsergebnisse dieses Servers werden als vertrauenswürdig behandelt.
- `Syslog true`: OpenDMARC schreibt Logmeldungen ins Systemlog.

Danach wurde das Verzeichnis für die Datei `ignore.hosts` erstellt:

```bash
# command
sudo install -d -m 755 /etc/opendmarc
```

Die Datei `/etc/opendmarc/ignore.hosts` wurde anschliessend erstellt:

```bash
# command
sudo tee /etc/opendmarc/ignore.hosts >/dev/null <<'EOF'
127.0.0.1
localhost
192.168.97.64
EOF
```

Die Datei enthält damit folgende Einträge:

```bash
# command
sudo cat /etc/opendmarc/ignore.hosts

# response
127.0.0.1
localhost
192.168.97.64
```

Diese Hosts werden ignoriert, damit lokale Verbindungen und der eigene Mailserver nicht unnötig durch OpenDMARC geprüft werden.

Danach wurde OpenDMARC neu gestartet:

```bash
# command
sudo systemctl restart opendmarc
```

Der Dienst wurde erneut geprüft:

```bash
# command
systemctl is-active opendmarc

# response
active
```

Zusätzlich wurde kontrolliert, ob OpenDMARC wirklich auf Port `12302` lauscht:

```bash
# command
sudo ss -ltnp | grep ':12302'

# response
LISTEN 0 4096 127.0.0.1:12302 0.0.0.0:* users:(("opendmarc",pid=1753,fd=3))
```

Damit ist OpenDMARC korrekt gestartet und lokal über `127.0.0.1:12302` erreichbar.

Als Nächstes wurde die bestehende Milter-Konfiguration von Postfix geprüft, damit die bestehende OpenDKIM-Konfiguration nicht überschrieben wird:

```bash
# command
sudo postconf -n | grep -E '^(smtpd_milters|non_smtpd_milters|milter_default_action|milter_protocol)'

# response
milter_default_action = accept
milter_protocol = 6
non_smtpd_milters = inet:localhost:12301
smtpd_milters = inet:localhost:12301
```

Zu diesem Zeitpunkt war nur OpenDKIM auf Port `12301` eingebunden. Danach wurde OpenDMARC zusätzlich in `smtpd_milters` eingetragen:

```bash
# command
sudo postconf -e "smtpd_milters = inet:localhost:12301, inet:localhost:12302"
```

Die neue Konfiguration wurde geprüft:

```bash
# command
sudo postconf -n smtpd_milters

# response
smtpd_milters = inet:localhost:12301, inet:localhost:12302
```

Damit verwendet Postfix für eingehende SMTP-Mails nun beide Milter:

- OpenDKIM auf `inet:localhost:12301`
- OpenDMARC auf `inet:localhost:12302`

Anschliessend wurde Postfix neu geladen:

```bash
# command
sudo systemctl reload postfix
```

Die finale Milter-Konfiguration sieht so aus:

```bash
# command
sudo postconf -n | grep -E '^(smtpd_milters|non_smtpd_milters|milter_default_action|milter_protocol)'

# response
milter_default_action = accept
milter_protocol = 6
non_smtpd_milters = inet:localhost:12301
smtpd_milters = inet:localhost:12301, inet:localhost:12302
```

`non_smtpd_milters` bleibt nur bei OpenDKIM. OpenDMARC wird hier vor allem für eingehende SMTP-Mails benötigt und ist deshalb in `smtpd_milters` eingetragen.

### 6.2 DNS und Richtlinien

Damit andere Mailserver wissen, welche DMARC-Richtlinie für unsere Domain gilt, muss ein DMARC-TXT-Record in der DNS-Zone gesetzt werden.

Zuerst wurde geprüft, ob bereits ein DMARC-Record existiert:

```bash
# command
dig @127.0.0.1 _dmarc.u8.cyberlab.fhnw.ch TXT +short
```

Da keine Ausgabe erschien, existierte noch kein DMARC-Record.

Danach wurde die Forward-Zone geprüft:

```bash
# command
sudo grep -nE 'SOA|serial|TXT|MX|_dmarc' /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
2:@   IN SOA ns1.u8.cyberlab.fhnw.ch. admin.u8.cyberlab.fhnw.ch. (
3:        2026051604 ; serial
20:@    IN MX 10 mail.u8.cyberlab.fhnw.ch.
23:@    IN TXT "v=spf1 mx -all"
26:default._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
```

Dabei war ersichtlich, dass SPF und DKIM bereits vorhanden waren, aber DMARC noch fehlte.

Vor der Anpassung wurde die Forward-Zone gesichert:

```bash
# command
sudo cp /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db.bak-before-dmarc
```

Danach wurde der DMARC-Record ergänzt. Gleichzeitig wurde der DNS-Serial von `2026051604` auf `2026051605` erhöht.

Der neue DMARC-Eintrag lautet:

```dns
; DMARC policy
_dmarc IN TXT "v=DMARC1; p=none; rua=mailto:fatlum@u8.cyberlab.fhnw.ch"
```

Die gesetzten Werte bedeuten:

- `v=DMARC1`: Es handelt sich um einen DMARC-Record.
- `p=none`: Es wird noch keine strenge Aktion wie Quarantäne oder Ablehnung erzwungen.
- `rua=mailto:fatlum@u8.cyberlab.fhnw.ch`: Aggregierte DMARC-Berichte sollen an diese Adresse gesendet werden.

Die Policy `p=none` ist für den Anfang sinnvoll, weil damit DMARC zuerst beobachtend betrieben wird. So kann geprüft werden, ob SPF und DKIM korrekt funktionieren, ohne dass legitime E-Mails direkt abgelehnt werden.

Danach wurde kontrolliert, ob der Record korrekt in der Zone steht:

```bash
# command
sudo grep -nE 'serial|_dmarc' /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
3:        2026051605 ; serial
31:_dmarc IN TXT "v=DMARC1; p=none; rua=mailto:fatlum@u8.cyberlab.fhnw.ch"
```

Anschliessend wurde die Zone syntaktisch geprüft:

```bash
# command
sudo named-checkzone u8.cyberlab.fhnw.ch /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
zone u8.cyberlab.fhnw.ch/IN: loaded serial 2026051605
OK
```

Danach wurde die Zone neu geladen:

```bash
# command
sudo rndc reload u8.cyberlab.fhnw.ch

# response
zone reload queued
```

Der DMARC-Record wurde danach lokal über BIND geprüft:

```bash
# command
dig @127.0.0.1 _dmarc.u8.cyberlab.fhnw.ch TXT +short

# response
"v=DMARC1; p=none; rua=mailto:fatlum@u8.cyberlab.fhnw.ch"
```

Zusätzlich wurde geprüft, ob der Record auch über den CyberLab-DNS sichtbar ist:

```bash
# command
dig @192.168.64.10 _dmarc.u8.cyberlab.fhnw.ch TXT +short

# response
"v=DMARC1; p=none; rua=mailto:fatlum@u8.cyberlab.fhnw.ch"
```

Damit ist der DMARC-DNS-Record korrekt veröffentlicht.

### 6.3 Tests

Zum Test wurde eine Mail über den lokalen Postfix an den CyberLab-Reflector gesendet. Die Ausgabe von `swaks` wurde dabei unterdrückt, weil nur die Logs relevant sind:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to reflector@cyberlab.fhnw.ch --server 127.0.0.1 >/dev/null
```

Danach wurde im Mail-Log geprüft, ob OpenDMARC aktiv wurde:

```bash
# command
sudo grep -i 'opendmarc' /var/log/mail.log | tail -n 10

# response
2026-05-17T14:57:51.742322+02:00 mail opendmarc[1700]: OpenDMARC Filter v1.4.2 starting ()
2026-05-17T14:57:51.742368+02:00 mail opendmarc[1700]: additional trusted authentication services: (none)
2026-05-17T15:03:37.223100+02:00 mail opendmarc[1700]: OpenDMARC Filter v1.4.2 terminating with status 0, errno = 0
2026-05-17T15:03:37.256132+02:00 mail opendmarc[1753]: OpenDMARC Filter v1.4.2 starting ()
2026-05-17T15:03:37.256289+02:00 mail opendmarc[1753]: additional trusted authentication services: mail.u8.cyberlab.fhnw.ch
2026-05-17T15:12:04.349378+02:00 mail opendmarc[1753]: ignoring connection from localhost
2026-05-17T15:12:18.383250+02:00 mail opendmarc[1753]: 4B8BB4001C: cyberlab.fhnw.ch pass
```

Die Zeile zeigt, dass OpenDMARC die eingehende Antwort des CyberLab-Reflectors geprüft hat. Das erkennt man an `header.from=cyberlab.fhnw.ch`. Der Wert `p=reject` stammt deshalb von der DMARC-Policy der Domain `cyberlab.fhnw.ch` und nicht von unserer eigenen Domain. Unsere eigene DMARC-Policy bleibt weiterhin `p=none`, wie der vorherige DNS-Test zeigt.

Zusätzlich wurden die Mail-Header der empfangenen Mail geprüft:

```bash
# command
sudo grep -iE 'Authentication-Results|DMARC|DKIM|SPF' /var/mail/fatlum | tail -n 30

# response
Received-SPF: pass (cyberlab.fhnw.ch: 192.168.64.33 is authorized to use 'reflector@cyberlab.fhnw.ch' in 'mfrom' identity (mechanism 'mx' matched)) receiver=u8.cyberlab.fhnw.ch; identity=mailfrom; envelope-from="reflector@cyberlab.fhnw.ch"; helo=srvMSG01.cyberlab.fhnw.ch; client-ip=192.168.64.33
Received-SPF: pass (cyberlab.fhnw.ch: 192.168.64.33 is authorized to use 'reflector@cyberlab.fhnw.ch' in 'mfrom' identity (mechanism 'mx' matched)) receiver=u8.cyberlab.fhnw.ch; identity=mailfrom; envelope-from="reflector@cyberlab.fhnw.ch"; helo=srvMSG01.cyberlab.fhnw.ch; client-ip=192.168.64.33
Authentication-Results: mail.u8.cyberlab.fhnw.ch; dmarc=pass (p=reject dis=none) header.from=cyberlab.fhnw.ch
Received-SPF: pass (cyberlab.fhnw.ch: 192.168.64.33 is authorized to use 'reflector@cyberlab.fhnw.ch' in 'mfrom' identity (mechanism 'mx' matched)) receiver=u8.cyberlab.fhnw.ch; identity=mailfrom; envelope-from="reflector@cyberlab.fhnw.ch"; helo=srvMSG01.cyberlab.fhnw.ch; client-ip=192.168.64.33
```

Die wichtigste Zeile ist:

```text
Authentication-Results: mail.u8.cyberlab.fhnw.ch; dmarc=pass (p=reject dis=none) header.from=cyberlab.fhnw.ch
```

Damit ist bestätigt, dass OpenDMARC die eingehende Mail geprüft und das Ergebnis in die Header geschrieben hat.

Zum Schluss wurde geprüft, ob die Mailqueue leer ist:

```bash
# command
mailq

# response
Mail queue is empty
```

Zusätzlich wurden alle beteiligten Dienste geprüft:

```bash
# command
systemctl is-active bind9 postfix opendkim opendmarc

# response
active
active
active
active
```

Damit ist DMARC erfolgreich eingerichtet und getestet. Der Mailserver veröffentlicht eine DMARC-Policy im DNS, Postfix leitet eingehende Mails an OpenDMARC weiter und die Logs sowie Mail-Header zeigen erfolgreiche DMARC-Prüfungen.

---

## 7 Virenscan (Amavis, ClamAV)

In diesem Kapitel wird ein Virenscan für eingehende und ausgehende E-Mails eingerichtet. Dafür wird Amavis als Content-Filter zwischen Postfix und der eigentlichen Zustellung eingebunden. ClamAV übernimmt dabei die Erkennung von Viren und Malware.

Die Mailverarbeitung läuft danach vereinfacht so ab: Postfix nimmt eine Mail an, leitet sie an Amavis weiter, Amavis prüft sie mit ClamAV und gibt saubere Mails wieder an Postfix zurück.

### 7.1 Installation

Zuerst wurden Amavis und ClamAV installiert:

```bash
# command
sudo apt-get install -y amavisd-new clamav clamav-daemon clamav-freshclam
```

Danach wurde der automatische Freshclam-Dienst kurz gestoppt, damit die Signaturdatenbank manuell aktualisiert werden kann:

```bash
# command
sudo systemctl stop clamav-freshclam
```

Die ClamAV-Signaturen wurden aktualisiert:

```bash
# command
sudo freshclam
```

Relevante Ausgabe:

```text
ClamAV update process started
daily.cvd database is up-to-date
main.cvd database is up-to-date
bytecode.cvd database is up-to-date
```

Danach wurde der Freshclam-Dienst wieder gestartet:

```bash
# command
sudo systemctl start clamav-freshclam
```

Der ClamAV-Daemon wurde anschliessend neu gestartet:

```bash
# command
sudo systemctl restart clamav-daemon
```

Damit sind Amavis und ClamAV installiert und die Virensignaturen aktuell.

### 7.2 Grundkonfiguration

Damit ClamAV auf die temporären Dateien von Amavis zugreifen kann, wurde der Benutzer `clamav` zur Gruppe `amavis` hinzugefügt:

```bash
# command
sudo adduser clamav amavis
```

Danach wurde die Amavis-Konfigurationsdatei für Content-Filter gesichert und geöffnet:

```bash
# command
sudo cp /etc/amavis/conf.d/15-content_filter_mode /etc/amavis/conf.d/15-content_filter_mode.bak-before-virus
sudo nano /etc/amavis/conf.d/15-content_filter_mode
```

In dieser Datei wurde der Virus-Block aktiviert:

```perl
@bypass_virus_checks_maps = (
   \%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);
```

Der Spam-Block wurde in diesem Schritt noch nicht aktiviert, da hier zuerst nur der Virenscan eingerichtet wird.

Danach wurde der Amavis-Hostname gesetzt. Dafür wurde zuerst die Datei gesichert und geöffnet:

```bash
# command
sudo cp /etc/amavis/conf.d/05-node_id /etc/amavis/conf.d/05-node_id.bak-before-virus
sudo nano /etc/amavis/conf.d/05-node_id
```

Die automatische Hostname-Ermittlung wurde auskommentiert:

```perl
# chomp($myhostname = `hostname --fqdn`);
```

Darunter wurde der korrekte Mailserver-Hostname gesetzt:

```perl
$myhostname = "mail.u8.cyberlab.fhnw.ch";
```

Danach wurden ClamAV und Amavis neu gestartet:

```bash
# command
sudo systemctl restart clamav-daemon amavis
```

Kontrolle:

```bash
# command
systemctl is-active clamav-daemon amavis

# response
active
active
```

Damit laufen ClamAV und Amavis korrekt.

### 7.3 Anbindung an Postfix

Damit Postfix Mails an Amavis weiterleitet, wurde zuerst die Postfix-`master.cf` gesichert und geöffnet:

```bash
# command
sudo cp /etc/postfix/master.cf /etc/postfix/master.cf.bak-before-amavis
sudo nano /etc/postfix/master.cf
```

Am Ende der Datei wurde folgender Block ergänzt:

```text
smtp-amavis unix -    -    -    -    2    smtp
    -o smtp_data_done_timeout=1200
    -o smtp_send_xforward_command=yes
    -o smtp_dns_support_level=disabled
    -o max_use=20

127.0.0.1:10025 inet n    -    -    -    -    smtpd
    -o content_filter=
    -o smtpd_milters=
    -o local_recipient_maps=
    -o relay_recipient_maps=
    -o smtpd_restriction_classes=
    -o smtpd_delay_reject=no
    -o smtpd_client_restrictions=permit_mynetworks,reject
    -o smtpd_helo_restrictions=
    -o smtpd_sender_restrictions=
    -o smtpd_recipient_restrictions=permit_mynetworks,reject
    -o smtpd_data_restrictions=reject_unauth_pipelining
    -o smtpd_end_of_data_restrictions=
    -o mynetworks=127.0.0.0/8
    -o smtpd_error_sleep_time=0
    -o smtpd_soft_error_limit=1001
    -o smtpd_hard_error_limit=1000
    -o smtpd_client_connection_count_limit=0
    -o smtpd_client_connection_rate_limit=0
    -o receive_override_options=no_unknown_recipient_checks,no_header_body_checks
```

Der erste Block definiert den Transport von Postfix zu Amavis auf Port `10024`. Der zweite Block definiert den Rückkanal von Amavis zurück zu Postfix auf Port `10025`.

Danach wurde in Postfix der Content-Filter aktiviert:

```bash
# command
sudo postconf -e "content_filter = smtp-amavis:[127.0.0.1]:10024"
```

Kontrolle:

```bash
# command
sudo postconf -n content_filter

# response
content_filter = smtp-amavis:[127.0.0.1]:10024
```

Bei der Kontrolle erschien zuerst folgende Warnung:

```text
postconf: warning: /etc/postfix/master.cf: support for parameter "disable_dns_lookups" will be removed; instead, specify "smtp_dns_support_level"
```

Deshalb wurde in `/etc/postfix/master.cf` im `smtp-amavis`-Block diese alte Zeile ersetzt:

```text
-o disable_dns_lookups=yes
```

durch:

```text
-o smtp_dns_support_level=disabled
```

Danach wurde die Postfix-Konfiguration geprüft:

```bash
# command
sudo postfix check
```

Dabei erschien folgende Warnung:

```text
postfix/postfix-script: warning: not owned by root: /var/spool/postfix/etc/resolv.conf
```

Die Datei wurde geprüft:

```bash
# command
ls -l /var/spool/postfix/etc/resolv.conf

# response
-rw-r--r-- 1 systemd-resolve systemd-resolve ... /var/spool/postfix/etc/resolv.conf
```

Der Besitzer wurde korrigiert:

```bash
# command
sudo chown root:root /var/spool/postfix/etc/resolv.conf
```

Danach war `postfix check` sauber:

```bash
# command
sudo postfix check
```

Postfix und Amavis wurden neu gestartet:

```bash
# command
sudo systemctl restart postfix amavis
```

Danach wurde geprüft, ob die Ports korrekt geöffnet sind:

```bash
# command
sudo ss -ltnp | grep -E ':10024|:10025'
```

Relevante Ausgabe:

```text
127.0.0.1:10025 users:(("master",...))
127.0.0.1:10024 users:(("/usr/sbin/amavi",...))
[::1]:10024 users:(("/usr/sbin/amavi",...))
```

Damit ist Postfix korrekt mit Amavis verbunden.

### 7.4 Tests und DKIM-Reihenfolge

Zuerst wurde eine saubere Testmail lokal versendet:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to fatlum@u8.cyberlab.fhnw.ch --server 127.0.0.1 --header "Subject: Amavis clean test" --body "clean test" >/dev/null
```

Danach wurde im Mail-Log geprüft, ob Amavis die Mail verarbeitet hat:

```bash
# command
sudo grep -Ei "amavis|Amavis clean test|Passed CLEAN" /var/log/mail.log | tail -n 30
```

Relevante Ausgabe:

```text
Using primary internal av scanner code for ClamAV-clamd
Found secondary av scanner ClamAV-clamscan at /usr/bin/clamscan
Passed CLEAN
```

Die Ausgabe `Passed CLEAN` zeigt, dass Amavis die Mail geprüft und als sauber weitergeleitet hat.

Danach wurde eine EICAR-Testdatei erstellt. EICAR ist keine echte Malware, sondern eine harmlose Testsignatur für Virenscanner.

```bash
# command
cat > /tmp/eicar.txt <<'EOF'
X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
EOF
```

Die EICAR-Datei wurde als Anhang verschickt:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to fatlum@u8.cyberlab.fhnw.ch --server 127.0.0.1 --header "Subject: EICAR virus test" --attach @/tmp/eicar.txt 2>&1 | grep -Ei "queued|reject|virus|eicar|250|554"
```

Relevante Ausgabe:

```text
250 2.0.0 Ok: queued as CDE164099E
```

Postfix hat die Mail zuerst angenommen. Danach wurde geprüft, was Amavis mit der Mail gemacht hat:

```bash
# command
sudo grep -Ei "CDE164099E|EICAR|virus|INFECTED|Blocked|Passed|quarantine" /var/log/mail.log | tail -n 40
```

Relevante Ausgabe:

```text
Blocked INFECTED (Eicar-Signature) {DiscardedInbound,Quarantined}
status=sent (250 2.7.0 Ok, discarded, id=03879-01 - INFECTED: Eicar-Signature)
```

Damit ist bestätigt, dass ClamAV die EICAR-Testsignatur erkannt und Amavis die Mail blockiert hat.

Danach wurde die DKIM-Reihenfolge getestet. Dafür wurde eine saubere Mail an den CyberLab-Reflector gesendet:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to reflector@cyberlab.fhnw.ch --server 127.0.0.1 --header "Subject: Amavis DKIM order test" --body "test after amavis integration" 2>&1 | grep -Ei "queued|250 2.0.0"

# response
250 2.0.0 Ok: queued as EB8974099E
```

Danach wurde im Log nach dieser Queue-ID gesucht:

```bash
# command
sudo grep -Ei "EB8974099E|Amavis DKIM order test|Passed CLEAN|DKIM-Signature|status=sent" /var/log/mail.log | tail -n 40
```

Dabei zeigte sich, dass DKIM zweimal angewendet wurde:

```text
EB8974099E: DKIM-Signature field added
21379409C8: DKIM-Signature field added
Passed CLEAN ... Queue-ID: EB8974099E ... queued_as: 21379409C8
```

Der Grund war, dass beim Rückkanal `127.0.0.1:10025` die Milter erneut aktiv waren. Dadurch wurde die Mail nach der Amavis-Prüfung ein zweites Mal von OpenDKIM signiert.

Zur Korrektur wurde erneut `/etc/postfix/master.cf` geöffnet:

```bash
# command
sudo nano /etc/postfix/master.cf
```

Im Block `127.0.0.1:10025` wurde folgende Zeile ergänzt:

```text
    -o smtpd_milters=
```

Die Zeile wurde direkt unter `content_filter=` eingefügt:

```text
127.0.0.1:10025 inet n    -    -    -    -    smtpd
    -o content_filter=
    -o smtpd_milters=
```

Damit werden beim Rückkanal von Amavis zu Postfix keine Milter erneut ausgeführt.

Danach wurde die Postfix-Konfiguration geprüft und Postfix neu gestartet:

```bash
# command
sudo postfix check
sudo systemctl restart postfix
```

Anschliessend wurde der DKIM-Reihenfolge-Test wiederholt:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to reflector@cyberlab.fhnw.ch --server 127.0.0.1 --header "Subject: Amavis DKIM order retest" --body "test after disabling milter on 10025" 2>&1 | grep -Ei "queued|250 2.0.0"

# response
250 2.0.0 Ok: queued as 28F4C4099E
```

Die Logs wurden erneut geprüft:

```bash
# command
sudo grep -Ei "28F4C4099E|Passed CLEAN|DKIM-Signature|status=sent" /var/log/mail.log | tail -n 40
```

Relevante Ausgabe:

```text
28F4C4099E: DKIM-Signature field added
Passed CLEAN ... Queue-ID: 28F4C4099E ... queued_as: 539DA409C8
28F4C4099E: to=<reflector@cyberlab.fhnw.ch>, relay=127.0.0.1[127.0.0.1]:10024, status=sent
539DA409C8: to=<reflector@cyberlab.fhnw.ch>, relay=srvMSG01.cyberlab.fhnw.ch[192.168.64.33]:25, status=sent
```

Nach dem Fix wurde DKIM nur noch einmal auf der ursprünglichen Queue-ID `28F4C4099E` angewendet. Die Mail wurde danach durch Amavis geprüft und als neue Queue-ID `539DA409C8` weitergesendet, ohne erneut DKIM zu signieren.

Zum Schluss wurde die Mailqueue geprüft:

```bash
# command
mailq

# response
Mail queue is empty
```

Alle beteiligten Dienste wurden ebenfalls geprüft:

```bash
# command
systemctl is-active postfix amavis clamav-daemon clamav-freshclam opendkim

# response
active
active
active
active
active
```

Die EICAR-Testdatei wurde entfernt:

```bash
# command
rm /tmp/eicar.txt
```

Damit sind Amavis und ClamAV erfolgreich eingebunden. Saubere Mails werden durchgelassen, die EICAR-Testsignatur wird blockiert und die DKIM-Reihenfolge wurde so korrigiert, dass ausgehende Mails nicht doppelt signiert werden.

---

## 8 Greylisting

In diesem Kapitel wird Greylisting mit Postgrey eingerichtet. Greylisting verzögert unbekannte Absender beim ersten Zustellversuch temporär. Seriöse Mailserver versuchen die Zustellung später erneut, während viele Spam-Systeme dies nicht tun.

### 8.1 Installation und Postfix

Zuerst wurde Postgrey installiert:

```bash
# command
sudo apt-get install -y postgrey
```

Danach wurde geprüft, ob der Dienst läuft:

```bash
# command
systemctl is-active postgrey

# response
active
```

Zusätzlich wurde geprüft, ob Postgrey auf Port `10023` lauscht:

```bash
# command
sudo ss -ltnp | grep ':10023'

# response
LISTEN 0 4096 127.0.0.1:10023 0.0.0.0:* users:(("postgrey --inet",pid=4803,fd=5))
LISTEN 0 4096 [::1]:10023 [::]:* users:(("postgrey --inet",pid=4803,fd=4))
```

Vor der Postfix-Anbindung wurde die aktuelle Empfängerprüfung kontrolliert:

```bash
# command
sudo postconf -n smtpd_recipient_restrictions

# response
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, check_policy_service unix:private/policyd-spf, reject_unauth_destination
```

Danach wurde Postgrey als zusätzlicher Policy-Dienst ergänzt:

```bash
# command
sudo postconf -e "smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, check_policy_service unix:private/policyd-spf, reject_unauth_destination, check_policy_service inet:127.0.0.1:10023"
```

Kontrolle:

```bash
# command
sudo postconf -n smtpd_recipient_restrictions

# response
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, check_policy_service unix:private/policyd-spf, reject_unauth_destination, check_policy_service inet:127.0.0.1:10023
```

Danach wurde die Postfix-Konfiguration geprüft und neu geladen:

```bash
# command
sudo postfix check
sudo systemctl reload postfix
```

### 8.2 Whitelist und Verzoegerung

Zuerst wurde die Postgrey-Default-Datei gesichert und geöffnet:

```bash
# command
sudo cp /etc/default/postgrey /etc/default/postgrey.bak-before-greylisting
sudo nano /etc/default/postgrey
```

Die Verzögerung wurde auf `60` Sekunden gesetzt:

```text
POSTGREY_OPTS="--inet=10023 --delay=60"
```

Danach wurde Postgrey neu gestartet:

```bash
# command
sudo systemctl restart postgrey
```

Kontrolle:

```bash
# command
systemctl is-active postgrey

# response
active
```

Der Port wurde nochmals geprüft:

```bash
# command
sudo ss -ltnp | grep ':10023'

# response
LISTEN 0 4096 127.0.0.1:10023 0.0.0.0:* users:(("postgrey --inet",pid=4970,fd=5))
LISTEN 0 4096 [::1]:10023 [::]:* users:(("postgrey --inet",pid=4970,fd=4))
```

Danach wurden die vorhandenen Postgrey-Dateien geprüft:

```bash
# command
ls -1 /etc/postgrey

# response
whitelist_clients
whitelist_recipients
```

Die Client-Whitelist wurde geöffnet:

```bash
# command
sudo nano /etc/postgrey/whitelist_clients
```

Folgende Einträge wurden ergänzt:

```text
# CyberLab trusted mail server
srvMSG01.cyberlab.fhnw.ch
srvmsg01.cyberlab.fhnw.ch
192.168.64.33
```

Danach wurde Postgrey neu gestartet:

```bash
# command
sudo systemctl restart postgrey
```

Kontrolle der Whitelist:

```bash
# command
sudo tail -n 10 /etc/postgrey/whitelist_clients

# response
# CyberLab trusted mail server
srvMSG01.cyberlab.fhnw.ch
srvmsg01.cyberlab.fhnw.ch
192.168.64.33
```

Kontrolle der Verzögerung:

```bash
# command
grep -n '^POSTGREY_OPTS' /etc/default/postgrey

# response
8:POSTGREY_OPTS="--inet=10023 --delay=60"
```

### 8.3 Tests

Zuerst wurde Postgrey direkt über den Policy-Port mit einem unbekannten Beispielclient getestet:

```bash
# command
printf 'request=smtpd_access_policy\nprotocol_state=RCPT\nprotocol_name=SMTP\nclient_address=203.0.113.99\nclient_name=unknown.example\nsender=test@example.net\nrecipient=fatlum@u8.cyberlab.fhnw.ch\n\n' | nc 127.0.0.1 10023
```

Relevante Ausgabe:

```text
action=DEFER_IF_PERMIT Greylisted
```

Das bedeutet, dass Postgrey den unbekannten Absender temporär verzögert. Die Mail wird dabei nicht dauerhaft verworfen.

Danach wurde der whitelisted CyberLab-Mailserver getestet:

```bash
# command
printf 'request=smtpd_access_policy\nprotocol_state=RCPT\nprotocol_name=SMTP\nclient_address=192.168.64.33\nclient_name=srvMSG01.cyberlab.fhnw.ch\nsender=reflector@cyberlab.fhnw.ch\nrecipient=fatlum@u8.cyberlab.fhnw.ch\n\n' | nc -w 3 127.0.0.1 10023
```

Ausgabe:

```text
action=DUNNO
```

`DUNNO` bedeutet, dass Postgrey nicht eingreift und Postfix normal weitermachen lässt.

Da in `/var/log/mail.log` keine Postgrey-Einträge sichtbar waren, wurden die Logs über `journalctl` geprüft:

```bash
# command
sudo journalctl -u postgrey --since "10 minutes ago" --no-pager
```

Relevante Ausgabe:

```text
postgrey: action=greylist, reason=new, client_name=unknown.example, client_address=203.0.113.99, sender=test@example.net, recipient=fatlum@u8.cyberlab.fhnw.ch
postgrey: action=pass, reason=client whitelist, client_name=srvMSG01.cyberlab.fhnw.ch, client_address=192.168.64.33, sender=reflector@cyberlab.fhnw.ch, recipient=fatlum@u8.cyberlab.fhnw.ch
```

Zum Schluss wurde die Mailqueue geprüft:

```bash
# command
mailq

# response
Mail queue is empty
```

Damit ist Greylisting korrekt eingerichtet. Unbekannte Absender werden temporär verzögert, während der CyberLab-Reflector über die Whitelist direkt durchgelassen wird.

---

## 9 Tarpit und Anti-UBE

### 9.1 Tarpit-Konfiguration

### 9.2 Anti-UBE-Konfiguration

### 9.3 Tests

## 10 DNSSEC

### 10.1 Berechtigungen und Signierung

### 10.2 Zonen und Vertrauenskette

### 10.3 Verifikation

## 11 Spam-Filter (Bayes)

### 11.1 Aktivierung und Amavis-Header

### 11.2 Datenbank und vertrauenswuerdige Netze

### 11.3 Tests

## 12 Backup

## 13 Gesamttests und Abnahme

## 14 Bewertung und produktiver Betrieb

### 14.1 Stärken und Schwächen

### 14.2 Verbesserungsmöglichkeiten

### 14.3 Betrieb in produktiver Umgebung

## 15 Quellen und KI-Nutzung
