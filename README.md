# Grotesque3 - HackMyVM (Medium) - Writeup

![Grotesque3 Icon](Grotesque3.png)

Dieses Repository enthält einen zusammengefassten Bericht über die Kompromittierung der HackMyVM-Maschine "Grotesque3" (Schwierigkeitsgrad: Medium).

## Metadaten

*   **Maschine:** Grotesque3 (HackMyVM - Medium)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Grotesque3](https://hackmyvm.eu/machines/machine.php?vm=Grotesque3)
*   **Autor des Writeups:** DarkSpirit
*   **Datum:** 16. November 2022
*   **Original Writeup:** [https://alientec1908.github.io/Grotesque3_HackMyVM_Medium/](https://alientec1908.github.io/Grotesque3_HackMyVM_Medium/)

## Zusammenfassung des Angriffspfads

Die initiale Erkundung mit `nmap` offenbarte einen SSH-Dienst (Port 22) und einen Apache-Webserver (Port 80). Versuche, Informationen über Steganographie aus einer Datei `atlasg.jpg` (mittels `steghide`, `stegsnow`, `stegseek`) zu gewinnen, schlugen zunächst fehl. Ein `gobuster`-Scan auf die IP-Adresse des Webservers lieferte nur die `index.html`.

Nach dem Hinzufügen des Hostnamens `grotek.hmv` zur lokalen `/etc/hosts`-Datei wurde mittels `wfuzz` eine Local File Inclusion (LFI)-Schwachstelle in einem versteckten PHP-Skript (`.../f66b22bf020334b04c7d0d3eb5010391.php`) über den Parameter `purpose` entdeckt. Durch Ausnutzen dieser LFI (`?purpose=/etc/passwd`) wurde der Benutzername `freddie` identifiziert.

Ein `hydra`-Brute-Force-Angriff auf den SSH-Dienst mit dem Benutzernamen `freddie` und der `rockyou.txt`-Liste war erfolgreich und enthüllte das Passwort `61a4e3e60c063d1e472dd780f64e6cad` (ein MD5-Hash, der als Klartextpasswort verwendet wurde). Dies ermöglichte den initialen SSH-Zugriff als `freddie`.

Als `freddie` wurde die User-Flag gelesen. Die lokale Enumeration (`ss`, `smbclient -L`) zeigte einen SMB-Share namens `grotesque`. Eine PHP-Reverse-Shell (`rev.php`) wurde auf das Zielsystem übertragen und über `smbclient put` auf den `grotesque`-Share hochgeladen. Später wurde ein dediziertes Bash-Reverse-Shell-Skript (`shell.sh`) erstellt, per HTTP-Server bereitgestellt, auf das Zielsystem heruntergeladen und ebenfalls auf den `grotesque`-Share hochgeladen.

Nach dem Hochladen von `shell.sh` auf den SMB-Share wurde eine Verbindung von einem Netcat-Listener auf dem Angreifer-System empfangen, die eine Root-Shell (`uid=0`) gewährte. Dies deutet darauf hin, dass der Inhalt des `grotesque`-Shares von einem Prozess mit Root-Rechten ausgeführt wurde. Abschließend wurde die Root-Flag gelesen.

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `steghide`
*   `stegsnow`
*   `stegseek`
*   `gobuster`
*   `nmap`
*   `vi` / Editor
*   `wfuzz`
*   `curl`
*   `hydra`
*   `ssh`
*   `ss`
*   `smbclient`
*   `wget`
*   `nbtscan`
*   `Metasploit` (msfconsole - erfolglos)
*   `python3 -m http.server`
*   `nc` (netcat)
*   `cat`, `ls`, `cd`, `echo`, `chmod`, `rm`, `su` (impliziert)

## Angriffsschritte (Übersicht)

1.  **Reconnaissance:** Ziel-IP (`arp-scan`). Steganographie-Versuche (`steghide`, `stegsnow`, `stegseek` auf `atlasg.jpg`) erfolglos. `gobuster` auf IP findet nur `index.html`. `nmap` -> Port 22 (SSH), Port 80 (HTTP/Apache).
2.  **VHost & LFI Discovery:** Hostname `grotek.hmv` zu `/etc/hosts` hinzufügen. `wfuzz` auf `http://grotek.hmv/f66b...php?FUZZ=/etc/passwd` -> Parameter `purpose` gefunden.
3.  **LFI Exploitation:** `curl "http://grotek.hmv/...php?purpose=/etc/passwd"` -> User `freddie` identifiziert.
4.  **Password Crack:** `hydra -l freddie -P rockyou.txt ssh://grotek.hmv` -> Passwort `61a4e3e60c063d1e472dd780f64e6cad` gefunden.
5.  **Initial Access:** Login via `ssh freddie@grotek.hmv`.
6.  **Enumeration als Freddie:** `ss` -> SMB (445), MySQL (3306 local). `smbclient -L 127.0.0.1` -> Share `grotesque`. User-Flag lesen (`cat user.txt`).
7.  **Vorbereitung Reverse Shell:** `shell.sh` (Bash Reverse Shell zu Port 4444) erstellen. Via `python3 -m http.server` bereitstellen. Auf Zielsystem mit `wget` nach `/tmp` herunterladen.
8.  **SMB Upload:** Mit `smbclient \\\\127.0.0.1\\grotesque` verbinden. `put shell.sh` aus `/tmp` auf den Share hochladen.
9.  **Privilege Escalation & Root Access:** `nc -lvnp 4444` Listener starten. Warten auf Ausführung von `shell.sh` vom SMB-Share (durch Root-Prozess) -> Root-Shell erhalten.
10. **Root Flag:** `cat root.txt`.

## Flags

*   **User Flag (`/home/freddie/user.txt`):** `35A7EB682E33E89606102A883596A880f`
*   **Root Flag (`/root/root.txt`):** `5C42D6BB0EE9CE4CB7E7349652C45C4A`

---

## Disclaimer

Die hier dargestellten Informationen und Techniken dienen ausschließlich Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Methoden dürfen nur auf Systemen angewendet werden, für die eine ausdrückliche Genehmigung vorliegt (z.B. in CTF-Umgebungen wie HackMyVM, Penetrationstests mit schriftlicher Erlaubnis). Das unbefugte Eindringen in fremde Computersysteme ist illegal und strafbar. Die Autoren übernehmen keine Haftung für missbräuchliche Verwendung der bereitgestellten Informationen. Handeln Sie stets legal und ethisch verantwortlich.

---
