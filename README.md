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

; Hosts
ns1  IN A 192.168.97.64
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

### 6.1 Konfiguration

### 6.2 DNS und Richtlinien

### 6.3 Tests

## 7 Virenscan (Amavis, ClamAV)

### 7.1 Installation

### 7.2 Grundkonfiguration

### 7.3 Anbindung an Postfix

### 7.4 Tests und DKIM-Reihenfolge

## 8 Greylisting

### 8.1 Installation und Postfix

### 8.2 Whitelist und Verzoegerung

### 8.3 Tests

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
