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
- [9 DNSSEC](#9-dnssec)
  - [9.1 Berechtigungen und Signierung](#91-berechtigungen-und-signierung)
  - [9.2 Zonen und Vertrauenskette](#92-zonen-und-vertrauenskette)
  - [9.3 Verifikation](#93-verifikation)
- [10 Spam-Filter (Bayes)](#10-spam-filter-bayes)
  - [10.1 Aktivierung und Amavis-Header](#101-aktivierung-und-amavis-header)
  - [10.2 Datenbank und vertrauenswuerdige Netze](#102-datenbank-und-vertrauenswuerdige-netze)
  - [10.3 Tests](#103-tests)
- [11 Backup](#11-backup)
- [12 Gesamttests und Abnahme](#12-gesamttests-und-abnahme)
- [13 Quellen und KI-Nutzung](#13-quellen-und-ki-nutzung)

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

### 1.4 Zertifikate

## 2 DNS (BIND9)

### 2.1 Installation und Basis

### 2.2 Zonenreferenz und Dateipfade

### 2.3 Forward-Zone

### 2.4 Reverse-Zone

### 2.5 Tests

## 3 Mailserver (Postfix)

### 3.1 Installation

### 3.2 Konfiguration und Smoke-Tests

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

## 9 DNSSEC

### 9.1 Berechtigungen und Signierung

### 9.2 Zonen und Vertrauenskette

### 9.3 Verifikation

## 10 Spam-Filter (Bayes)

### 10.1 Aktivierung und Amavis-Header

### 10.2 Datenbank und vertrauenswuerdige Netze

### 10.3 Tests

## 11 Backup

## 12 Gesamttests und Abnahme

## 13 Quellen und KI-Nutzung
