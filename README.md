# Hello World

Ich werde den Mailserver auf meinem MacBookAir M4 machen, mit UTM virtualisere ich eine Debian 13 Trixie maschine

- Image: debian-13.4.0-arm64-netinst.iso
- Link für Download: <https://www.debian.org/releases/trixie/debian-installer/>

## Setup auf dem Mac

![4 GB RAM](images/14.07.10.4gb-ram.png)

Im UTM-Dialog den Arbeitsspeicher der VM auf 4 GB gesetzt und bestätigt.

![ISO hinterlegen](images/14.09.10.iso-hinterlegen.png)

Die Debian-Netinstall-ISO als Boot-Medium der VM zugewiesen.

![30 GB Disk](images/14.10.10.30-gb-disk.png)

Virtuelle Festplatte mit 30 GB angelegt bzw. Größe festgelegt.

![Fortfahren nach Disk](images/14.11.10.continou-after-disk.png)

Nach der Festplattenangabe auf Fortfahren geklickt.

![Zusammenfassung und Einstellungen](images/14.11.15.zusammenfassung-and-open-settings.png)

Zusammenfassung der VM geprüft und die Einstellungen geöffnet.

![GPU auswählen](images/14.12.10.gpu-auswaehlen.png)

GPU-/Grafikoption in UTM ausgewählt.

![Netzwerk Cyberlab](images/14.20.10.netzwerk-cyberlab-auswaehlen.png)

Als Netzwerkmodus das Cyberlab-Netzwerk ausgewählt.

![Installation starten](images/14.22.10.click-install.png)

Die VM gestartet bzw. den Installationsablauf begonnen.

![Sprachauswahl](images/14.26.42.sprachauswahl.png)

Im Debian-Installer die Anzeigesprache gewählt.

![Location Other](images/14.27.15.location-other.png)

Bei der Region „Other“ ausgewählt.

![Kontinent](images/14.27.22.location-continent.png)

Den Kontinent für Tastatur und Locale markiert.

![Land](images/14.27.32.country.png)

Das Land ausgewählt.

![Tastatur-Location](images/14.30.39.keyboard-location.png)

Tastatur-Herkunft bzw. -layout-Schritt bestätigt oder angepasst.

![Tastatureinstellungen](images/14.30.53.keyboard-settings.png)

Konkrete Tastaturbelegung gewählt.

![Hostname](images/14.35.53.network-hostname.png)

Rechnernamen (Hostname) für die VM eingetragen.

![Domain](images/14.37.18.network-domain.png)

Domainnamen fürs Netzwerk angegeben.

![Root-Passwort](images/14.39.11.user-pw.png)

Passwort für den Benutzer root gesetzt.

![Root-Passwort wiederholen](images/14.39.20.user-pw-repeat.png)

Root-Passwort zur Bestätigung erneut eingegeben.

![Vollständiger Name](images/14.41.15.fullname-user.png)

Vollständigen Namen des normalen Benutzerkontos eingetragen.

![Benutzername](images/14.41.19.username.png)

Login-Namen des Benutzers festgelegt.

![Benutzer-Passwort](images/14.42.12.user-pw.png)

Passwort für den normalen Benutzer gesetzt.

![Benutzer-Passwort wiederholen](images/14.42.33.user-pw-repeat.png)

Benutzerpasswort wiederholt.

![Partitionierung Benutzerbereich](images/14.44.06.user-partition.png)

Partitionierungsmethode bzw. Heimverzeichnis-Schritt ausgewählt.

![Virtuelle Disk](images/14.44.56.virtual-disk.png)

Die virtuelle Festplatte als Ziel für die Partitionierung markiert.

![Partitionsschema](images/14.45.27.virtualdisk-partitioning.png)

Partitionen auf der virtuellen Disk angelegt oder angepasst.

![Partitionierung abschließen](images/14.45.52.finish-partition.png)

Partitionierung fertiggestellt und mit Fortfahren bestätigt.

![Partitionen schreiben](images/14.46.30.write-partitioning.png)

„Änderungen auf die Festplatten schreiben?“ mit Ja bestätigt.

![Paketmanager](images/14.47.35.packetmanager.png)

Paketquellen / Paketmanager-Schritt im Installer bearbeitet.

![Mirror-Land](images/14.48.26.mirror-country.png)

Land für den Debian-Paketspiegel ausgewählt.

![Archiv-Mirror](images/14.48.59.archive-mirror.png)

Konkreten Spiegelserver für das Archiv gewählt.

![HTTP-Proxy](images/14.49.28.http-proxy.png)

HTTP-Proxy leer gelassen bzw. mit Fortfahren übersprungen.

![Popularity Contest](images/14.50.20.popularity.png)

Teilnahme am popularity-contest mit Ja oder Nein beantwortet.

![Software-Auswahl](images/14.51.05.software-installation.png)

Desktop bzw. Softwareprofile für die Installation angeklickt.

![Installation fertig](images/14.52.32.installation-complete.png)

Installation abgeschlossen und Neustart ausgewählt bzw. bestätigt.

![ISO entfernen](images/14.57.23.remove-iso.png)

In UTM die ISO vom Laufwerk entfernt, damit von der Platte gebootet wird.

![IP für SSH](images/15.00.31.ip-a-for-ssh.png)

Nach dem Start `ip a` ausgeführt, um die IP-Adresse für SSH zu sehen.
