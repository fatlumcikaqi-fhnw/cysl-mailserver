<!-- markdownlint-disable-file MD033 -->
# Hello World

Ich werde den Mailserver auf meinem MacBookAir M4 machen, mit UTM virtualisere ich eine Debian 13 Trixie maschine

- Image: debian-13.4.0-arm64-netinst.iso
- Link für Download: <https://www.debian.org/releases/trixie/debian-installer/>

## Inhaltsverzeichnis

- [1 VM und Netzwerk](#1-vm-und-netzwerk)
  - [1.1 UTM: VM anlegen und Debian installieren](#11-utm-vm-anlegen-und-debian-installieren)
  - [1.2 Forward-Zone und Nameserver (dig)](#12-forward-zone-und-nameserver-dig)
  - [1.3 Netzwerk und statische IP](#13-netzwerk-und-statische-ip)
  - [1.4 Zertifikate](#14-zertifikate)
- [2 DNS (BIND9)](#2-dns-bind9)
  - [2.1 Installation und Basis](#21-installation-und-basis)
  - [2.2 Zonenreferenz und Dateipfade](#22-zonenreferenz-und-dateipfade)
  - [2.3 Forward-Zone](#23-forward-zone)
  - [2.4 Reverse-Zone](#24-reverse-zone)
  - [2.5 Tests](#25-tests)
- [3 Mailserver (Postfix)](#3-mailserver-postfix)
  - [3.1 Installation](#31-installation)
  - [3.2 Konfiguration und Smoke-Tests](#32-konfiguration-und-smoke-tests)
- [4 SPF](#4-spf)
  - [4.1 Ausgehend](#41-ausgehend)
  - [4.2 Eingehend](#42-eingehend)
  - [4.3 Tests](#43-tests)
- [5 DKIM](#5-dkim)
  - [5.1 OpenDKIM und Postfix (Milter)](#51-opendkim-und-postfix-milter)
  - [5.2 DKIM-Schluessel in der DNS-Zone](#52-dkim-schluessel-in-der-dns-zone)
  - [5.3 Tests](#53-tests)
- [6 DMARC](#6-dmarc)
  - [6.1 Konfiguration](#61-konfiguration)
  - [6.2 DNS und Richtlinien](#62-dns-und-richtlinien)
  - [6.3 Tests](#63-tests)
- [7 Virenscan (Amavis, ClamAV)](#7-virenscan-amavis-clamav)
  - [7.1 Installation](#71-installation)
  - [7.2 Grundkonfiguration](#72-grundkonfiguration)
  - [7.3 Anbindung an Postfix](#73-anbindung-an-postfix)
  - [7.4 Tests und DKIM-Reihenfolge](#74-tests-und-dkim-reihenfolge)
- [8 Greylisting](#8-greylisting)
  - [8.1 Installation und Postfix](#81-installation-und-postfix)
  - [8.2 Whitelist und Verzoegerung](#82-whitelist-und-verzoegerung)
  - [8.3 Tests](#83-tests)
- [9 Tarpit und Anti-UBE](#9-tarpit-und-anti-ube)
  - [9.1 Tarpit-Konfiguration](#91-tarpit-konfiguration)
  - [9.2 Anti-UBE-Konfiguration](#92-anti-ube-konfiguration)
  - [9.3 Tests](#93-tests)
- [10 DNSSEC](#10-dnssec)
  - [10.1 Berechtigungen und Signierung](#101-berechtigungen-und-signierung)
  - [10.2 Zonen und Vertrauenskette](#102-zonen-und-vertrauenskette)
  - [10.3 Verifikation](#103-verifikation)
- [11 Spam-Filter (Bayes)](#11-spam-filter-bayes)
  - [11.1 Aktivierung und Amavis-Header](#111-aktivierung-und-amavis-header)
  - [11.2 Datenbank und vertrauenswuerdige Netze](#112-datenbank-und-vertrauenswuerdige-netze)
  - [11.3 Tests](#113-tests)
- [12 Backup](#12-backup)
- [13 Gesamttests und Abnahme](#13-gesamttests-und-abnahme)
- [14 Bewertung und produktiver Betrieb](#14-bewertung-und-produktiver-betrieb)
  - [14.1 Stärken und Schwächen](#141-stärken-und-schwächen)
  - [14.2 Verbesserungsmöglichkeiten](#142-verbesserungsmöglichkeiten)
  - [14.3 Betrieb in produktiver Umgebung](#143-betrieb-in-produktiver-umgebung)
- [15 Quellen und KI-Nutzung](#15-quellen-und-ki-nutzung)

## 1 VM und Netzwerk

### 1.1 UTM: VM anlegen und Debian installieren

<img src="images/14.07.10.4gb-ram.png" alt="4 GB RAM" width="500">
Im UTM-Dialog den Arbeitsspeicher der VM auf 4 GB gesetzt und bestätigt.

<img src="images/14.09.10.iso-hinterlegen.png" alt="ISO hinterlegen" width="500">
Die Debian-Netinstall-ISO als Boot-Medium der VM zugewiesen.

<img src="images/14.10.10.30-gb-disk.png" alt="30 GB Disk" width="500">
Virtuelle Festplatte mit 30 GB angelegt bzw. Größe festgelegt.

<img src="images/14.11.10.continou-after-disk.png" alt="Fortfahren nach Disk" width="500">
Nach der Festplattenangabe auf Fortfahren geklickt.

<img src="images/14.11.15.zusammenfassung-and-open-settings.png" alt="Zusammenfassung und Einstellungen" width="500">
Zusammenfassung der VM geprüft und die Einstellungen geöffnet.

<img src="images/14.12.10.gpu-auswaehlen.png" alt="GPU auswählen" width="500">
GPU-/Grafikoption in UTM ausgewählt.

<img src="images/14.20.10.netzwerk-cyberlab-auswaehlen.png" alt="Netzwerk Cyberlab" width="500">
Als Netzwerkmodus das Cyberlab-Netzwerk ausgewählt.

<img src="images/14.22.10.click-install.png" alt="Installation starten" width="500">
Die VM gestartet bzw. den Installationsablauf begonnen.

<img src="images/14.26.42.sprachauswahl.png" alt="Sprachauswahl" width="500">
Im Debian-Installer die Anzeigesprache gewählt.

<img src="images/14.27.15.location-other.png" alt="Location Other" width="500">
Bei der Region „Other“ ausgewählt.

<img src="images/14.27.22.location-continent.png" alt="Kontinent" width="500">
Den Kontinent für Tastatur und Locale markiert.

<img src="images/14.27.32.country.png" alt="Land" width="500">
Das Land ausgewählt.

<img src="images/14.30.39.keyboard-location.png" alt="Tastatur-Location" width="500">
Tastatur-Herkunft bzw. -layout-Schritt bestätigt oder angepasst.

<img src="images/14.30.53.keyboard-settings.png" alt="Tastatureinstellungen" width="500">
Konkrete Tastaturbelegung gewählt.

<img src="images/14.35.53.network-hostname.png" alt="Hostname" width="500">
Rechnernamen (Hostname) für die VM eingetragen.

<img src="images/14.37.18.network-domain.png" alt="Domain" width="500">
Domainnamen fürs Netzwerk angegeben.

<img src="images/14.39.11.user-pw.png" alt="Root-Passwort" width="500">
Leeres Passwort für den Benutzer root gesetzt.

<img src="images/14.39.20.user-pw-repeat.png" alt="Root-Passwort wiederholen" width="500">
Leeres Root-Passwort zur Bestätigung erneut eingegeben.

<img src="images/14.41.15.fullname-user.png" alt="Vollständiger Name" width="500">
Vollständigen Namen des normalen Benutzerkontos eingetragen.

<img src="images/14.41.19.username.png" alt="Benutzername" width="500">
Login-Namen `fatlum` des Benutzers festgelegt.

<img src="images/14.42.12.user-pw.png" alt="Benutzer-Passwort" width="500">
Passwort `HokageKonoha..-` für den normalen Benutzer gesetzt.

<img src="images/14.42.33.user-pw-repeat.png" alt="Benutzer-Passwort wiederholen" width="500">
Benutzerpasswort wiederholt.

<img src="images/14.44.06.user-partition.png" alt="Partitionierung Benutzerbereich" width="500">
Partitionierungsmethode bzw. Heimverzeichnis-Schritt ausgewählt.

<img src="images/14.44.56.virtual-disk.png" alt="Virtuelle Disk" width="500">
Die virtuelle Festplatte als Ziel für die Partitionierung markiert.

<img src="images/14.45.27.virtualdisk-partitioning.png" alt="Partitionsschema" width="500">
Partitionen auf der virtuellen Disk angelegt oder angepasst.

<img src="images/14.45.52.finish-partition.png" alt="Partitionierung abschließen" width="500">
Partitionierung fertiggestellt und mit Fortfahren bestätigt.

<img src="images/14.46.30.write-partitioning.png" alt="Partitionen schreiben" width="500">
„Änderungen auf die Festplatten schreiben?“ mit Ja bestätigt.

<img src="images/14.47.35.packetmanager.png" alt="Paketmanager" width="500">
Paketquellen / Paketmanager-Schritt im Installer bearbeitet.

<img src="images/14.48.26.mirror-country.png" alt="Mirror-Land" width="500">
Land für den Debian-Paketspiegel ausgewählt.

<img src="images/14.48.59.archive-mirror.png" alt="Archiv-Mirror" width="500">
Konkreten Spiegelserver für das Archiv gewählt.

<img src="images/14.49.28.http-proxy.png" alt="HTTP-Proxy" width="500">
HTTP-Proxy leer gelassen bzw. mit Fortfahren übersprungen.

<img src="images/14.50.20.popularity.png" alt="Popularity Contest" width="500">
Teilnahme am popularity-contest mit Ja oder Nein beantwortet.

<img src="images/14.51.05.software-installation.png" alt="Software-Auswahl" width="500">
Desktop bzw. Softwareprofile für die Installation angeklickt.

<img src="images/14.52.32.installation-complete.png" alt="Installation fertig" width="500">
Installation abgeschlossen und Neustart ausgewählt bzw. bestätigt.

<img src="images/14.57.23.remove-iso.png" alt="ISO entfernen" width="500">
In UTM die ISO vom Laufwerk entfernt, damit von der Platte gebootet wird.

<img src="images/15.00.31.ip-a-for-ssh.png" alt="IP für SSH" width="500">
Nach dem Start `ip a` ausgeführt, um die IP-Adresse für SSH zu sehen.

### 1.2 Forward-Zone und Nameserver (dig)

Um uns ein gesamt Bild der Domain zu machen, legen wir zuerst ein query auf der forward domain ab:

```bash
# command
dig +norecurse u8.cyberlab.fhnw.ch

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> +norecurse u8.cyberlab.fhnw.ch
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5077
;; flags: qr ra; QUERY: 1, ANSWER: 0, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 5e1fd42112b12ee70100000069f87bc90733d3f9ddd54c6e (good)
;; QUESTION SECTION:
;u8.cyberlab.fhnw.ch.  IN A

;; AUTHORITY SECTION:
u8.cyberlab.fhnw.ch. 4800 IN NS ns2.u8.cyberlab.fhnw.ch.
u8.cyberlab.fhnw.ch. 4800 IN NS ns1.u8.cyberlab.fhnw.ch.

;; ADDITIONAL SECTION:
ns2.u8.cyberlab.fhnw.ch. 4800 IN A 192.168.97.65
ns1.u8.cyberlab.fhnw.ch. 4800 IN A 192.168.97.64

;; Query time: 0 msec
;; SERVER: 192.168.64.10#53(192.168.64.10) (UDP)
;; WHEN: Mon May 04 12:58:17 CEST 2026
;; MSG SIZE  rcvd: 144
```

Damit sieht man, dass wir für die Domain u8.cyberlab.fhnw.ch und die Namesservers `ns1.u8.cyberlab.fhnw.ch` und `ns2.u8.cyberlab.fhnw.ch`sind.

Jetzt müssen wir herausfinden, welche IP-Range wir haben und reverse DNS-Zone

```bash
# command
dig +norecurse -x 192.168.97.65

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> +norecurse -x 192.168.97.65
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25828
;; flags: qr aa ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 07e3662f5876a7bb0100000069f88892e65918144b81715a (good)
;; QUESTION SECTION:
;65.97.168.192.in-addr.arpa. IN PTR

;; ANSWER SECTION:
65.97.168.192.in-addr.arpa. 604800 IN CNAME 65.64/29.97.168.192.in-addr.arpa.

;; AUTHORITY SECTION:
64/29.97.168.192.in-addr.arpa. 604800 IN NS ns2.u8.cyberlab.fhnw.ch.
64/29.97.168.192.in-addr.arpa. 604800 IN NS ns1.u8.cyberlab.fhnw.ch.

;; Query time: 4 msec
;; SERVER: 192.168.64.10#53(192.168.64.10) (UDP)
;; WHEN: Mon May 04 13:52:50 CEST 2026
;; MSG SIZE  rcvd: 184
```

Wir verwalten die DNS-Zone 64/29.97.168.192.in-addr.arpa für das Subnetz 192.168.97.64 - .71. Während wir DNS-seitig nur diesen /29-Block kontrollieren, konfigurieren wir die VM-Netzmaske auf /22, um mit allen anderen Systemen im CyberLab-Netzwerk kommunizieren zu können.

Hier ist dein ergänzter und strukturierter Text für das Kapitel 1.3. Ich habe den Teil mit der **Namensauflösung (systemd-resolved)** integriert, da dies bei Debian 13 entscheidend ist, damit dein System nach der Umstellung nicht "stumm" bleibt.

---

### 1.3 Netzwerk und statische IP

Um einen stabilen Betrieb als DNS- und Mailserver zu gewährleisten, stellen wir die IP-Konfiguration von DHCP auf statisch um. Hierbei verwenden wir `systemd-networkd`.

Zuerst ermitteln wir den Namen des Netzwerk-Interfaces:

```bash
# command
ip a
```

In diesem Fall ist die relevante Schnittstelle `enp0s1`. Wir legen nun eine Konfigurationsdatei für dieses Interface an:

```bash
# Datei anlegen
sudo nano /etc/systemd/network/10-enp0s1.network
```

Inhalt der Datei:

```ini
[Match]
Name=enp0s1

[Network]
Address=192.168.97.64/22
Gateway=192.168.96.1
DNS=192.168.64.10
DNS=192.168.64.11
```

**Hinweis zur Netzmaske:** Wir verwenden hier `/22`, um mit der gesamten CyberLab-Infrastruktur kommunizieren zu können, obwohl wir DNS-technisch später nur für ein `/29`-Subnetz verantwortlich sind.

Damit die Konfiguration greift, wechseln wir von der alten ifupdown/DHCP-Konfiguration zu `systemd-networkd`. **Achtung:** Dies unterbricht bestehende SSH-Verbindungen.

```bash
# Altes System deaktivieren
sudo systemctl stop networking
sudo systemctl disable networking

# systemd-networkd aktivieren
sudo systemctl enable systemd-networkd
sudo systemctl restart systemd-networkd
```

#### Fehlerbehebung: Alte DHCP-Konfiguration entfernen

Nach der Umstellung auf `systemd-networkd` prüfen wir, ob das Interface wirklich nur noch statisch verwaltet wird:

```bash
ip a
ip route
ps aux | grep -E "dhcpcd|dhclient|NetworkManager|systemd-networkd" | grep -v grep
````

Falls zusätzlich zur statischen Adresse `192.168.97.64` noch eine dynamische Adresse erscheint, wird das Interface wahrscheinlich noch über die alte Datei `/etc/network/interfaces` per DHCP gestartet. In unserem Fall war dort noch folgende Konfiguration vorhanden:

```text
allow-hotplug enp0s1
iface enp0s1 inet dhcp
```

Diese DHCP-Konfiguration muss entfernt werden, damit `enp0s1` nur noch durch `systemd-networkd` verwaltet wird.

```bash
sudo nano /etc/network/interfaces
```

Die Datei sollte danach nur noch die Loopback-Konfiguration enthalten:

```text
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback
```

Danach stoppen wir den alten `ifup`-Dienst für das Interface und beenden den DHCP-Client:

```bash
sudo systemctl stop ifup@enp0s1.service
sudo pkill dhcpcd
```

Anschließend prüfen wir erneut die IP-Adressen des Interfaces:

```bash
ip a
ip route
```

Falls neben der statischen Adresse `192.168.97.64/22` noch eine zusätzliche dynamische Adresse vorhanden ist, entfernen wir genau diese DHCP-Adresse. In unserem Fall war dies `192.168.97.168/22`:

```bash
sudo ip addr del 192.168.97.168/22 dev enp0s1
```

Bei einer anderen Installation kann die dynamische Adresse abweichen. Dann muss statt `192.168.97.168/22` die tatsächlich angezeigte DHCP-Adresse verwendet werden:

```bash
sudo ip addr del <DHCP-IP>/22 dev enp0s1
```

Danach laden wir die statische Netzwerkkonfiguration neu:

```bash
sudo systemctl restart systemd-networkd
```

**Hinweis:** Dieser Schritt kann eine bestehende SSH-Verbindung kurz unterbrechen, da die Netzwerkkonfiguration neu geladen wird. Danach kann man sich wieder über die statische Adresse verbinden:

```bash
ssh fatlum@192.168.97.64
```

Zum Schluss prüfen wir, ob nur noch `systemd-networkd` aktiv ist:

```bash
ip a
ip route
ps aux | grep -E "dhcpcd|dhclient|NetworkManager|systemd-networkd" | grep -v grep
```

Der Zielzustand ist, dass `enp0s1` nur noch die statische IPv4-Adresse `192.168.97.64/22` besitzt und keine DHCP-Route mehr vorhanden ist.

#### Fehlerbehebung: Namensauflösung (systemd-resolved)

Nach dem Wechsel kann es vorkommen, dass `ping google.com` fehlschlägt, da `systemd-networkd` zwar die Leitung konfiguriert, aber keinen lokalen Resolver bereitstellt. Unter Debian 13 muss dafür oft `systemd-resolved` nachinstalliert werden.

Da ohne funktionierendes DNS kein `apt install` möglich ist, erzwingen wir kurzzeitig einen Nameserver und installieren dann den Dienst:

```bash
# Temporären DNS für die Installation setzen
echo "nameserver 192.168.64.10" | sudo tee /etc/resolv.conf

# Paket nachinstallieren
sudo apt update && sudo apt install systemd-resolved -y

# Dienst aktivieren
sudo systemctl enable --now systemd-resolved

# WICHTIG: Symlink erstellen, damit das System systemd-resolved nutzt
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Durch den Symlink auf die `stub-resolv.conf` greifen alle Programme (via `/etc/resolv.conf`) nun auf den DNS-Manager von systemd zu, welcher die DNS-Server aus unserer `.network`-Datei verwendet.

**Test der Konfiguration:**

```bash
ping -c 3 google.com
```

Wenn der Ping erfolgreich ist, verfügt die VM über eine stabile IP und eine funktionierende Namensauflösung.

---

### 1.4 Zertifikate

Für diese Laborumgebung wurden keine zusätzlichen Zertifikate eingerichtet. Die im Auftrag geforderten Sicherungsmassnahmen beziehen sich auf die Mailinfrastruktur selbst, insbesondere DNS, Postfix, SPF/DKIM, Greylisting, Tarpit, Anti-UBE, DNSSEC sowie Viren- und Spamfilterung.

Ein zusätzlicher Import der CyberLab-Zertifizierungsstelle wäre nur notwendig, falls interne HTTPS-Dienste mit eigenen Zertifikaten verwendet werden und die VM diesen Zertifikaten vertrauen muss. Für den hier dokumentierten Mailserver-Aufbau war dies nicht erforderlich.

---

## 2 DNS (BIND9)

In diesem Kapitel richten wir BIND9 als autoritativen DNS-Server für `u8.cyberlab.fhnw.ch` ein. Dazu erstellen wir eine Forward-Zone, eine Reverse-Zone und prüfen anschliessend, ob die Namensauflösung intern sowie über den CyberLab-DNS korrekt funktioniert.

### 2.1 Installation und Basis

Zuerst prüfen wir, ob BIND9 bereits installiert ist:

```bash
# command
dpkg -l | grep -E "bind9|dnsutils"

# response
ii  bind9-dnsutils                    1:9.20.21-1~deb13u1                  arm64        Clients provided with BIND 9
ii  bind9-host                        1:9.20.21-1~deb13u1                  arm64        DNS Lookup Utility
ii  bind9-libs:arm64                  1:9.20.21-1~deb13u1                  arm64        Shared Libraries used by BIND 9
```

Es sind bereits DNS-Tools vorhanden, aber der eigentliche BIND9-Server fehlt noch. Deshalb installieren wir `bind9`:

```bash
# command
sudo apt install -y bind9
```

Danach prüfen wir, ob der Dienst läuft:

```bash
# command
systemctl status bind9 --no-pager

# response
● named.service - BIND Domain Name Server
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-05-16 14:45:34 CEST; 49s ago
 Invocation: 8eade6df3aaf4256982b6e006bd0f434
       Docs: man:named(8)
   Main PID: 981 (named)
     Status: "running"
      Tasks: 8 (limit: 4579)
     Memory: 41.2M (peak: 41.6M)
        CPU: 15ms
     CGroup: /system.slice/named.service
             └─981 /usr/sbin/named -f -u bind
```

Zusätzlich setzen wir die lokale Host-Auflösung korrekt. Der Name `mail.u8.cyberlab.fhnw.ch` soll nicht auf `127.0.1.1`, sondern auf die statische Server-IP zeigen:

```bash
# command
grep -nE "127\.0\.1\.1|mail\.u8\.cyberlab\.fhnw\.ch|u8\.cyberlab\.fhnw\.ch|mail" /etc/hosts

# response
2:127.0.1.1 mail.u8.cyberlab.fhnw.ch mail
```

Wir passen die Datei an:

```bash
# command
sudo nano /etc/hosts
```

Danach sieht die Datei so aus:

```bash
# command
cat /etc/hosts

# response
127.0.0.1 localhost
192.168.97.64 mail.u8.cyberlab.fhnw.ch mail

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

### 2.2 Zonenreferenz und Dateipfade

Für die DNS-Zonen erstellen wir ein eigenes Verzeichnis unter `/var/lib/bind`. Dort werden später die Forward- und Reverse-Zonendateien abgelegt.

```bash
# command
sudo mkdir -p /var/lib/bind/u8.cyberlab.fhnw.ch && sudo chown -R bind:bind /var/lib/bind/u8.cyberlab.fhnw.ch && ls -ld /var/lib/bind/u8.cyberlab.fhnw.ch

# response
drwxr-xr-x 2 bind bind 4096 May 16 14:47 /var/lib/bind/u8.cyberlab.fhnw.ch
```

Danach legen wir die beiden Zonendateien an:

```bash
# command
sudo touch /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db /var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db && sudo chown bind:bind /var/lib/bind/u8.cyberlab.fhnw.ch/*.db && ls -l /var/lib/bind/u8.cyberlab.fhnw.ch/

# response
total 0
-rw-r--r-- 1 bind bind 0 May 16 14:48 forward.db
-rw-r--r-- 1 bind bind 0 May 16 14:48 reverse.db
```

Anschliessend prüfen wir die lokale BIND-Konfigurationsdatei:

```bash
# command
cat /etc/bind/named.conf.local

# response
//
// Do any local configuration here
//
```

Nun tragen wir die Forward- und Reverse-Zone ein:

```bash
# command
sudo nano /etc/bind/named.conf.local
```

Die Datei enthält danach:

```bash
# command
cat /etc/bind/named.conf.local

# response
//
// Do any local configuration here
//

zone "u8.cyberlab.fhnw.ch" {
    type master;
    file "/var/lib/bind/u8.cyberlab.fhnw.ch/forward.db";
};

zone "64/29.97.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db";
};
```

Die Konfiguration wird anschliessend geprüft:

```bash
# command
sudo named-checkconf

# response
# keine Ausgabe
```

Keine Ausgabe bedeutet, dass die BIND-Konfiguration syntaktisch korrekt ist.

### 2.3 Forward-Zone

Nun erstellen wir die Forward-Zone. Diese bildet Namen wie `mail.u8.cyberlab.fhnw.ch` auf IP-Adressen ab.

```bash
# command
sudo nano /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db
```

Die Datei enthält danach:

```bash
# command
cat /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
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

Die Forward-Zone wird danach geprüft:

```bash
# command
sudo named-checkzone u8.cyberlab.fhnw.ch /var/lib/bind/u8.cyberlab.fhnw.ch/forward.db

# response
zone u8.cyberlab.fhnw.ch/IN: loaded serial 2026051601
OK
```

### 2.4 Reverse-Zone

Nun erstellen wir die Reverse-Zone. Diese bildet die IP-Adresse `192.168.97.64` zurück auf den Hostnamen `mail.u8.cyberlab.fhnw.ch`.

```bash
# command
sudo nano /var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db
```

Die Datei enthält danach:

```bash
# command
cat /var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db

# response
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

Die Reverse-Zone wird danach geprüft:

```bash
# command
sudo named-checkzone 64/29.97.168.192.in-addr.arpa /var/lib/bind/u8.cyberlab.fhnw.ch/reverse.db

# response
zone 64/29.97.168.192.in-addr.arpa/IN: loaded serial 2026051601
OK
```

Danach starten wir BIND9 neu:

```bash
# command
sudo systemctl restart bind9
```

Anschliessend prüfen wir den Status:

```bash
# command
systemctl status bind9 --no-pager

# response
● named.service - BIND Domain Name Server
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-05-16 14:55:08 CEST; 7s ago
 Invocation: df91f16dc3824dc892858aeea5c43e55
       Docs: man:named(8)
   Main PID: 1063 (named)
     Status: "running"
      Tasks: 6 (limit: 4579)
     Memory: 37.1M (peak: 37.5M)
        CPU: 23ms
     CGroup: /system.slice/named.service
             └─1063 /usr/sbin/named -f -u bind
```

### 2.5 Tests

Zuerst testen wir die Forward-Zone direkt über den lokalen BIND-Server:

```bash
# command
dig @127.0.0.1 mail.u8.cyberlab.fhnw.ch

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @127.0.0.1 mail.u8.cyberlab.fhnw.ch
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5091
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 50238781c9ed199c010000006a08695fd9852689da6e44a6 (good)
;; QUESTION SECTION:
;mail.u8.cyberlab.fhnw.ch. IN A

;; ANSWER SECTION:
mail.u8.cyberlab.fhnw.ch. 86400 IN A 192.168.97.64

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Sat May 16 14:55:59 CEST 2026
;; MSG SIZE  rcvd: 97
```

Danach testen wir die Reverse-Zone direkt über den lokalen BIND-Server:

```bash
# command
dig @127.0.0.1 64.64/29.97.168.192.in-addr.arpa PTR

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @127.0.0.1 64.64/29.97.168.192.in-addr.arpa PTR
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24663
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: a6952804fb789834010000006a0869a3967eaab4a7d4bf0c (good)
;; QUESTION SECTION:
;64.64/29.97.168.192.in-addr.arpa. IN PTR

;; ANSWER SECTION:
64.64/29.97.168.192.in-addr.arpa. 86400 IN PTR mail.u8.cyberlab.fhnw.ch.

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1) (UDP)
;; WHEN: Sat May 16 14:57:07 CEST 2026
;; MSG SIZE  rcvd: 127
```

Nun testen wir die normale Reverse-Auflösung:

```bash
# command
dig -x 192.168.97.64

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> -x 192.168.97.64
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16884
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;64.97.168.192.in-addr.arpa. IN PTR

;; ANSWER SECTION:
64.97.168.192.in-addr.arpa. 604800 IN CNAME 64.64/29.97.168.192.in-addr.arpa.
64.64/29.97.168.192.in-addr.arpa. 10 IN PTR mail.u8.cyberlab.fhnw.ch.

;; Query time: 1923 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Sat May 16 14:58:02 CEST 2026
;; MSG SIZE  rcvd: 116
```

Auch die normale Forward-Auflösung zeigt nun auf die richtige IP-Adresse:

```bash
# command
dig mail.u8.cyberlab.fhnw.ch

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> mail.u8.cyberlab.fhnw.ch
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64822
;; flags: qr aa rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;mail.u8.cyberlab.fhnw.ch. IN A

;; ANSWER SECTION:
mail.u8.cyberlab.fhnw.ch. 0 IN A 192.168.97.64

;; Query time: 0 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Sat May 16 15:02:35 CEST 2026
;; MSG SIZE  rcvd: 69
```

Danach testen wir, ob BIND9 auch über die echte Server-IP erreichbar ist:

```bash
# command
dig @192.168.97.64 mail.u8.cyberlab.fhnw.ch

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @192.168.97.64 mail.u8.cyberlab.fhnw.ch
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28202
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: ef9ec678f2a038a2010000006a086b2224645e367da26af9 (good)
;; QUESTION SECTION:
;mail.u8.cyberlab.fhnw.ch. IN A

;; ANSWER SECTION:
mail.u8.cyberlab.fhnw.ch. 86400 IN A 192.168.97.64

;; Query time: 0 msec
;; SERVER: 192.168.97.64#53(192.168.97.64) (UDP)
;; WHEN: Sat May 16 15:03:30 CEST 2026
;; MSG SIZE  rcvd: 97
```

Auch die Reverse-Zone funktioniert über die echte Server-IP:

```bash
# command
dig @192.168.97.64 64.64/29.97.168.192.in-addr.arpa PTR

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @192.168.97.64 64.64/29.97.168.192.in-addr.arpa PTR
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 6764
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 0de72553b4e9da78010000006a086b554a94adb5a2852eda (good)
;; QUESTION SECTION:
;64.64/29.97.168.192.in-addr.arpa. IN PTR

;; ANSWER SECTION:
64.64/29.97.168.192.in-addr.arpa. 86400 IN PTR mail.u8.cyberlab.fhnw.ch.

;; Query time: 0 msec
;; SERVER: 192.168.97.64#53(192.168.97.64) (UDP)
;; WHEN: Sat May 16 15:04:21 CEST 2026
;; MSG SIZE  rcvd: 127
```

Der MX-Record wird ebenfalls korrekt ausgeliefert:

```bash
# command
dig @192.168.97.64 u8.cyberlab.fhnw.ch MX

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @192.168.97.64 u8.cyberlab.fhnw.ch MX
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 42944
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 825f2a2aa151b0e2010000006a086b905dda4a74624c249b (good)
;; QUESTION SECTION:
;u8.cyberlab.fhnw.ch.  IN MX

;; ANSWER SECTION:
u8.cyberlab.fhnw.ch. 86400 IN MX 10 mail.u8.cyberlab.fhnw.ch.

;; ADDITIONAL SECTION:
mail.u8.cyberlab.fhnw.ch. 86400 IN A 192.168.97.64

;; Query time: 0 msec
;; SERVER: 192.168.97.64#53(192.168.97.64) (UDP)
;; WHEN: Sat May 16 15:05:20 CEST 2026
;; MSG SIZE  rcvd: 113
```

Zum Schluss testen wir die Auflösung über den CyberLab-DNS `192.168.64.10`.

```bash
# command
dig @192.168.64.10 mail.u8.cyberlab.fhnw.ch

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @192.168.64.10 mail.u8.cyberlab.fhnw.ch
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12127
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 397fd1d3f2289445010000006a086bd8b489895d852eedf4 (good)
;; QUESTION SECTION:
;mail.u8.cyberlab.fhnw.ch. IN A

;; ANSWER SECTION:
mail.u8.cyberlab.fhnw.ch. 10 IN A 192.168.97.64

;; Query time: 819 msec
;; SERVER: 192.168.64.10#53(192.168.64.10) (UDP)
;; WHEN: Sat May 16 15:06:32 CEST 2026
;; MSG SIZE  rcvd: 97
```

Auch der MX-Record ist über den CyberLab-DNS sichtbar:

```bash
# command
dig @192.168.64.10 u8.cyberlab.fhnw.ch MX

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @192.168.64.10 u8.cyberlab.fhnw.ch MX
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 27128
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 06244426ba818656010000006a086c28807bb1aa0fad8f11 (good)
;; QUESTION SECTION:
;u8.cyberlab.fhnw.ch.  IN MX

;; ANSWER SECTION:
u8.cyberlab.fhnw.ch. 10 IN MX 10 mail.u8.cyberlab.fhnw.ch.

;; Query time: 15 msec
;; SERVER: 192.168.64.10#53(192.168.64.10) (UDP)
;; WHEN: Sat May 16 15:07:52 CEST 2026
;; MSG SIZE  rcvd: 97
```

Auch die Reverse-Auflösung funktioniert über den CyberLab-DNS:

```bash
# command
dig @192.168.64.10 -x 192.168.97.64

# response
; <<>> DiG 9.20.21-1~deb13u1-Debian <<>> @192.168.64.10 -x 192.168.97.64
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3340
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 7c702dd5c73be429010000006a086c691d4a4c1cbfa6593b (good)
;; QUESTION SECTION:
;64.97.168.192.in-addr.arpa. IN PTR

;; ANSWER SECTION:
64.97.168.192.in-addr.arpa. 604800 IN CNAME 64.64/29.97.168.192.in-addr.arpa.
64.64/29.97.168.192.in-addr.arpa. 10 IN PTR mail.u8.cyberlab.fhnw.ch.

;; Query time: 1227 msec
;; SERVER: 192.168.64.10#53(192.168.64.10) (UDP)
;; WHEN: Sat May 16 15:08:57 CEST 2026
;; MSG SIZE  rcvd: 167
```

Damit ist BIND9 korrekt eingerichtet. Forward Lookup, Reverse Lookup und MX Lookup funktionieren sowohl direkt über den eigenen DNS-Server als auch über den CyberLab-DNS.

---

## 3 Mailserver (Postfix)

In diesem Kapitel richten wir Postfix als Mailserver für `u8.cyberlab.fhnw.ch` ein. Danach konfigurieren wir die wichtigsten Hostnamen und testen, ob lokale Mails sowie Mails an den CyberLab-Reflector korrekt gesendet und empfangen werden können.

### 3.1 Installation

Zuerst prüfen wir, ob Postfix, `swaks` und `rsyslog` bereits installiert sind:

```bash
# command
dpkg -l | grep -E "postfix|swaks|rsyslog"

# response
# keine Ausgabe
```

Da noch keine der benötigten Komponenten installiert ist, installieren wir Postfix, `swaks` für SMTP-Tests und `rsyslog` für die Mail-Logs:

```bash
# command
sudo apt install postfix swaks rsyslog
```

Während der Installation wird Postfix nach der Grundkonfiguration gefragt. Für diesen Mailserver wählen wir:

```text
General mail configuration type: Internet Site
System mail name: u8.cyberlab.fhnw.ch
```

Falls beim Dialog versehentlich `mail.u8.cyberlab.fhnw.ch` als System-Mail-Name gesetzt wurde, ist das nicht kritisch. Die Werte werden nach der Installation explizit gesetzt.

Nun konfigurieren wir die wichtigsten Postfix-Werte:

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

# response
u8.cyberlab.fhnw.ch
```

Danach prüfen wir die gesetzten Werte:

```bash
# command
sudo postconf -n | grep -E "^(myhostname|mydomain|myorigin|mydestination|inet_protocols|smtpd_banner)"
echo "--- /etc/mailname ---"
cat /etc/mailname

# response
inet_protocols = ipv4
mydestination = $myhostname, $mydomain, localhost.$mydomain, localhost
mydomain = u8.cyberlab.fhnw.ch
myhostname = mail.u8.cyberlab.fhnw.ch
myorigin = /etc/mailname
smtpd_banner = mail.u8.cyberlab.fhnw.ch ESMTP Postfix
--- /etc/mailname ---
u8.cyberlab.fhnw.ch
```

Damit sind die wichtigsten Postfix-Werte korrekt gesetzt. Nun prüfen wir, ob der Dienst läuft:

```bash
# command
systemctl status postfix --no-pager

# response
● postfix.service - Postfix Mail Transport Agent (main/default instance)
     Loaded: loaded (/usr/lib/systemd/system/postfix.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-05-16 15:56:11 CEST; 1min 12s ago
 Invocation: 90659742343b4de5ad47ab5b1645c88c
       Docs: man:postfix(1)
    Process: 2149 ExecStartPre=postfix check (code=exited, status=0/SUCCESS)
    Process: 2256 ExecStart=postfix debian-systemd-start (code=exited, status=0/SUCCESS)
   Main PID: 2265 (master)
      Tasks: 3 (limit: 4579)
     Memory: 2.7M (peak: 5.4M)
        CPU: 130ms
     CGroup: /system.slice/postfix.service
             ├─2265 /usr/lib/postfix/sbin/master -w
             ├─2266 pickup -l -t unix -u -c
             └─2267 qmgr -l -t unix -u
```

Zusätzlich setzen wir den System-Hostname passend zum Mailserver:

```bash
# command
sudo hostnamectl set-hostname mail.u8.cyberlab.fhnw.ch
```

Danach prüfen wir den Hostnamen:

```bash
# command
sudo hostnamectl

# response
 Static hostname: mail.u8.cyberlab.fhnw.ch
       Icon name: computer-vm
         Chassis: vm 🖴
      Machine ID: eef94e9b87a14aa3ae2820374ed9b6db
         Boot ID: c325d19fc54f41e29fde7180d7dace0b
    Product UUID: a364284e-1fea-4d68-945f-630ff441e506
    AF_VSOCK CID: 1
  Virtualization: qemu
Operating System: Debian GNU/Linux 13 (trixie)        
          Kernel: Linux 6.12.74+deb13+1-arm64
    Architecture: arm64
 Hardware Vendor: QEMU
  Hardware Model: QEMU Virtual Machine
Firmware Version: 0.0.0
   Firmware Date: Fri 2015-02-06
    Firmware Age: 11y 3month 1w 1d
```

Der vollständige Hostname wird ebenfalls geprüft:

```bash
# command
hostname -f

# response
mail.u8.cyberlab.fhnw.ch
```

### 3.2 Konfiguration und Smoke-Tests

Zuerst testen wir, ob Postfix lokal auf Port 25 erreichbar ist und ob der SMTP-Banner korrekt gesetzt wurde:

```bash
# command
nc -v 127.0.0.1 25

# response
localhost [127.0.0.1] 25 (smtp) open
220 mail.u8.cyberlab.fhnw.ch ESMTP Postfix
QUIT
221 2.0.0 Bye
```

Mit `nc` prüfen wir, ob Postfix lokal auf Port 25 erreichbar ist und sich mit dem korrekten SMTP-Banner meldet. Der Test wird zuerst über `127.0.0.1` durchgeführt, damit nur der lokale Postfix-Dienst geprüft wird und keine externen Netzwerkfaktoren das Ergebnis beeinflussen.

Der Banner zeigt korrekt `mail.u8.cyberlab.fhnw.ch ESMTP Postfix`.

Danach prüfen wir, ob `rsyslog` läuft, damit Mail-Logs unter `/var/log/mail.log` geschrieben werden:

```bash
# command
systemctl status rsyslog --no-pager

# response
● rsyslog.service - System Logging Service
     Loaded: loaded (/usr/lib/systemd/system/rsyslog.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-05-16 15:49:58 CEST; 9min ago
 Invocation: 1815ff23eb7d4d73a746ea32a9b27702
TriggeredBy: ● syslog.socket
       Docs: man:rsyslogd(8)
             man:rsyslog.conf(5)
             https://www.rsyslog.com/doc/
   Main PID: 1709 (rsyslogd)
      Tasks: 4 (limit: 4579)
     Memory: 968K (peak: 2.3M)
        CPU: 18ms
     CGroup: /system.slice/rsyslog.service
             └─1709 /usr/sbin/rsyslogd -n -iNONE
```

Die Logdatei ist vorhanden:

```bash
# command
ls -l /var/log/mail.log

# response
-rw-r----- 1 root adm 1335 May 16 15:58 /var/log/mail.log
```

Nun testen wir zuerst die lokale Mailzustellung. Dazu senden wir eine Mail von `fatlum@u8.cyberlab.fhnw.ch` an denselben Benutzer:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to fatlum@u8.cyberlab.fhnw.ch --server 127.0.0.1

# response
=== Trying 127.0.0.1:25...
=== Connected to 127.0.0.1.
<-  220 mail.u8.cyberlab.fhnw.ch ESMTP Postfix
 -> EHLO mail.u8.cyberlab.fhnw.ch
<-  250-mail.u8.cyberlab.fhnw.ch
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-VRFY
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<fatlum@u8.cyberlab.fhnw.ch>
<-  250 2.1.0 Ok
 -> RCPT TO:<fatlum@u8.cyberlab.fhnw.ch>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Sat, 16 May 2026 16:11:48 +0200
 -> To: fatlum@u8.cyberlab.fhnw.ch
 -> From: fatlum@u8.cyberlab.fhnw.ch
 -> Subject: test Sat, 16 May 2026 16:11:48 +0200
 -> Message-Id: <20260516161148.002311@mail.u8.cyberlab.fhnw.ch>
 -> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 -> 
 -> This is a test mailing
 -> 
 -> 
 -> .
<-  250 2.0.0 Ok: queued as CBFBF407BE
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

Die Zeile `250 2.0.0 Ok` zeigt, dass Postfix die Mail angenommen hat. Nun prüfen wir die lokale Mailbox:

```bash
# command
sudo tail -n 40 /var/mail/fatlum

# response
From fatlum@u8.cyberlab.fhnw.ch  Sat May 16 16:11:48 2026
Return-Path: <fatlum@u8.cyberlab.fhnw.ch>
X-Original-To: fatlum@u8.cyberlab.fhnw.ch
Delivered-To: fatlum@u8.cyberlab.fhnw.ch
Received: from mail.u8.cyberlab.fhnw.ch (localhost [127.0.0.1])
 by mail.u8.cyberlab.fhnw.ch (Postfix) with ESMTP id CBFBF407BE
 for <fatlum@u8.cyberlab.fhnw.ch>; Sat, 16 May 2026 16:11:48 +0200 (CEST)
Date: Sat, 16 May 2026 16:11:48 +0200
To: fatlum@u8.cyberlab.fhnw.ch
From: fatlum@u8.cyberlab.fhnw.ch
Subject: test Sat, 16 May 2026 16:11:48 +0200
Message-Id: <20260516161148.002311@mail.u8.cyberlab.fhnw.ch>
X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/

This is a test mailing
```

Damit funktioniert die lokale Zustellung.

Als nächstes senden wir eine Testmail an den CyberLab-Reflector:

```bash
# command
swaks --from fatlum@u8.cyberlab.fhnw.ch --to reflector@cyberlab.fhnw.ch --server 127.0.0.1

# response
=== Trying 127.0.0.1:25...
=== Connected to 127.0.0.1.
<-  220 mail.u8.cyberlab.fhnw.ch ESMTP Postfix
 -> EHLO mail.u8.cyberlab.fhnw.ch
<-  250-mail.u8.cyberlab.fhnw.ch
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-VRFY
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<fatlum@u8.cyberlab.fhnw.ch>
<-  250 2.1.0 Ok
 -> RCPT TO:<reflector@cyberlab.fhnw.ch>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Sat, 16 May 2026 16:13:25 +0200
 -> To: reflector@cyberlab.fhnw.ch
 -> From: fatlum@u8.cyberlab.fhnw.ch
 -> Subject: test Sat, 16 May 2026 16:13:25 +0200
 -> Message-Id: <20260516161325.002322@mail.u8.cyberlab.fhnw.ch>
 -> X-Mailer: swaks v20240103.0 jetmore.org/john/code/swaks/
 -> 
 -> This is a test mailing
 -> 
 -> 
 -> .
<-  250 2.0.0 Ok: queued as 21957407BE
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

Danach prüfen wir im Mail-Log, ob die Mail erfolgreich an den Reflector weitergegeben wurde:

```bash
# command
sudo grep "21957407BE" /var/log/mail.log

# response
2026-05-16T16:13:25.137663+02:00 mail postfix/smtpd[2312]: 21957407BE: client=localhost[127.0.0.1]
2026-05-16T16:13:25.138386+02:00 mail postfix/cleanup[2315]: 21957407BE: message-id=<20260516161325.002322@mail.u8.cyberlab.fhnw.ch>
2026-05-16T16:13:25.140503+02:00 mail postfix/qmgr[2267]: 21957407BE: from=<fatlum@u8.cyberlab.fhnw.ch>, size=507, nrcpt=1 (queue active)
2026-05-16T16:13:26.396352+02:00 mail postfix/smtp[2323]: 21957407BE: to=<reflector@cyberlab.fhnw.ch>, relay=srvMSG01.cyberlab.fhnw.ch[192.168.64.33]:25, delay=1.3, delays=0/0.01/1/0.23, dsn=2.0.0, status=sent (250 2.0.0 Ok: queued as 56285601BC)
2026-05-16T16:13:26.397062+02:00 mail postfix/qmgr[2267]: 21957407BE: removed
```

Die wichtige Stelle ist `status=sent`. Damit wurde die Mail erfolgreich an `srvMSG01.cyberlab.fhnw.ch` übergeben.

Nun prüfen wir, ob die Antwort des Reflectors angekommen ist:

```bash
# command
sudo tail -n 80 /var/mail/fatlum

# response
From reflector@cyberlab.fhnw.ch  Sat May 16 16:13:37 2026
Return-Path: <reflector@cyberlab.fhnw.ch>
X-Original-To: fatlum@u8.cyberlab.fhnw.ch
Delivered-To: fatlum@u8.cyberlab.fhnw.ch
Received: from srvMSG01.cyberlab.fhnw.ch (srvMSG01.cyberlab.fhnw.ch [192.168.64.33])
 by mail.u8.cyberlab.fhnw.ch (Postfix) with ESMTP id 53A90407BE
 for <fatlum@u8.cyberlab.fhnw.ch>; Sat, 16 May 2026 16:13:37 +0200 (CEST)
Received: from localhost (localhost [127.0.0.1])
 by srvMSG01.cyberlab.fhnw.ch (Postfix) with ESMTP id 3E433601BC
 for <fatlum@u8.cyberlab.fhnw.ch>; Sat, 16 May 2026 16:13:37 +0200 (CEST)
Received: from srvMSG01.cyberlab.fhnw.ch ([IPv6:::1])
 by localhost (srvmsg01.cyberlab.fhnw.ch [IPv6:::1]) (amavisd-new, port 10024)
 with ESMTP id cjBiVvA_9vEG for <fatlum@u8.cyberlab.fhnw.ch>;
 Sat, 16 May 2026 16:13:35 +0200 (CEST)
Received: by srvMSG01.cyberlab.fhnw.ch (Postfix, from userid 1001)
 id 9162E604FF; Sat, 16 May 2026 16:13:34 +0200 (CEST)
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="1804289383-1778940814=:1112176"
Subject: reflected: test Sat, 16 May 2026 16:13:25 +0200
To: <fatlum@cyberlab.fhnw.ch>
User-Agent: mail (GNU Mailutils 3.14)
Date: Sat, 16 May 2026 16:13:34 +0200
Message-Id: <20260516141334.9162E604FF@srvMSG01.cyberlab.fhnw.ch>
From: reflector@cyberlab.fhnw.ch

This is a message from reflector
I received the attached mail
73
reflector aka god
```

Die Antwort des Reflectors ist in der lokalen Mailbox angekommen. Damit ist bestätigt, dass der Mailserver Mails senden und empfangen kann.

Zum Schluss prüfen wir, ob die Mailqueue leer ist:

```bash
# command
mailq

# response
Mail queue is empty
```

Damit ist die Postfix-Grundkonfiguration erfolgreich abgeschlossen.

---

## 4 SPF

### 4.1 Ausgehend

### 4.2 Eingehend

### 4.3 Tests

## 5 DKIM

### 5.1 OpenDKIM und Postfix (Milter)

### 5.2 DKIM-Schluessel in der DNS-Zone

### 5.3 Tests

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
