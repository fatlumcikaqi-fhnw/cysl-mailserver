# Hello World

Ich werde den Mailserver auf meinem MacBookAir M4 machen, mit UTM virtualisere ich eine Debian 13 Trixie maschine

- Image: debian-13.4.0-arm64-netinst.iso
- Link für Download: <https://www.debian.org/releases/trixie/debian-installer/>

## Setup auf dem Mac

<img src="images/14.07.10.4gb-ram.png" alt="4 GB RAM" width="600">

Im UTM-Dialog den Arbeitsspeicher der VM auf 4 GB gesetzt und bestätigt.

<img src="images/14.09.10.iso-hinterlegen.png" alt="ISO hinterlegen" width="600">

Die Debian-Netinstall-ISO als Boot-Medium der VM zugewiesen.

<img src="images/14.10.10.30-gb-disk.png" alt="30 GB Disk" width="600">

Virtuelle Festplatte mit 30 GB angelegt bzw. Größe festgelegt.

<img src="images/14.11.10.continou-after-disk.png" alt="Fortfahren nach Disk" width="600">

Nach der Festplattenangabe auf Fortfahren geklickt.

<img src="images/14.11.15.zusammenfassung-and-open-settings.png" alt="Zusammenfassung und Einstellungen" width="600">

Zusammenfassung der VM geprüft und die Einstellungen geöffnet.

<img src="images/14.12.10.gpu-auswaehlen.png" alt="GPU auswählen" width="600">

GPU-/Grafikoption in UTM ausgewählt.

<img src="images/14.20.10.netzwerk-cyberlab-auswaehlen.png" alt="Netzwerk Cyberlab" width="600">

Als Netzwerkmodus das Cyberlab-Netzwerk ausgewählt.

<img src="images/14.22.10.click-install.png" alt="Installation starten" width="600">

Die VM gestartet bzw. den Installationsablauf begonnen.

<img src="images/14.26.42.sprachauswahl.png" alt="Sprachauswahl" width="600">

Im Debian-Installer die Anzeigesprache gewählt.

<img src="images/14.27.15.location-other.png" alt="Location Other" width="600">

Bei der Region „Other“ ausgewählt.

<img src="images/14.27.22.location-continent.png" alt="Kontinent" width="600">

Den Kontinent für Tastatur und Locale markiert.

<img src="images/14.27.32.country.png" alt="Land" width="600">

Das Land ausgewählt.

<img src="images/14.30.39.keyboard-location.png" alt="Tastatur-Location" width="600">

Tastatur-Herkunft bzw. -layout-Schritt bestätigt oder angepasst.

<img src="images/14.30.53.keyboard-settings.png" alt="Tastatureinstellungen" width="600">

Konkrete Tastaturbelegung gewählt.

<img src="images/14.35.53.network-hostname.png" alt="Hostname" width="600">

Rechnernamen (Hostname) für die VM eingetragen.

<img src="images/14.37.18.network-domain.png" alt="Domain" width="600">

Domainnamen fürs Netzwerk angegeben.

<img src="images/14.39.11.user-pw.png" alt="Root-Passwort" width="600">

Passwort für den Benutzer root gesetzt.

<img src="images/14.39.20.user-pw-repeat.png" alt="Root-Passwort wiederholen" width="600">

Root-Passwort zur Bestätigung erneut eingegeben.

<img src="images/14.41.15.fullname-user.png" alt="Vollständiger Name" width="600">

Vollständigen Namen des normalen Benutzerkontos eingetragen.

<img src="images/14.41.19.username.png" alt="Benutzername" width="600">

Login-Namen des Benutzers festgelegt.

<img src="images/14.42.12.user-pw.png" alt="Benutzer-Passwort" width="600">

Passwort für den normalen Benutzer gesetzt.

<img src="images/14.42.33.user-pw-repeat.png" alt="Benutzer-Passwort wiederholen" width="600">

Benutzerpasswort wiederholt.

<img src="images/14.44.06.user-partition.png" alt="Partitionierung Benutzerbereich" width="600">

Partitionierungsmethode bzw. Heimverzeichnis-Schritt ausgewählt.

<img src="images/14.44.56.virtual-disk.png" alt="Virtuelle Disk" width="600">

Die virtuelle Festplatte als Ziel für die Partitionierung markiert.

<img src="images/14.45.27.virtualdisk-partitioning.png" alt="Partitionsschema" width="600">

Partitionen auf der virtuellen Disk angelegt oder angepasst.

<img src="images/14.45.52.finish-partition.png" alt="Partitionierung abschließen" width="600">

Partitionierung fertiggestellt und mit Fortfahren bestätigt.

<img src="images/14.46.30.write-partitioning.png" alt="Partitionen schreiben" width="600">

„Änderungen auf die Festplatten schreiben?“ mit Ja bestätigt.

<img src="images/14.47.35.packetmanager.png" alt="Paketmanager" width="600">

Paketquellen / Paketmanager-Schritt im Installer bearbeitet.

<img src="images/14.48.26.mirror-country.png" alt="Mirror-Land" width="600">

Land für den Debian-Paketspiegel ausgewählt.

<img src="images/14.48.59.archive-mirror.png" alt="Archiv-Mirror" width="600">

Konkreten Spiegelserver für das Archiv gewählt.

<img src="images/14.49.28.http-proxy.png" alt="HTTP-Proxy" width="600">

HTTP-Proxy leer gelassen bzw. mit Fortfahren übersprungen.

<img src="images/14.50.20.popularity.png" alt="Popularity Contest" width="600">

Teilnahme am popularity-contest mit Ja oder Nein beantwortet.

<img src="images/14.51.05.software-installation.png" alt="Software-Auswahl" width="600">

Desktop bzw. Softwareprofile für die Installation angeklickt.

<img src="images/14.52.32.installation-complete.png" alt="Installation fertig" width="600">

Installation abgeschlossen und Neustart ausgewählt bzw. bestätigt.

<img src="images/14.57.23.remove-iso.png" alt="ISO entfernen" width="600">

In UTM die ISO vom Laufwerk entfernt, damit von der Platte gebootet wird.

<img src="images/15.00.31.ip-a-for-ssh.png" alt="IP für SSH" width="600">

Nach dem Start `ip a` ausgeführt, um die IP-Adresse für SSH zu sehen.
