# Crazymed - HackMyVM (Easy)

![Crazymed Icon](Crazymed.png)

## Übersicht

*   **VM:** Crazymed
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Crazymed)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 2022-11-02
*   **Original-Writeup:** https://alientec1908.github.io/Crazymed_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Die virtuelle Maschine "Crazymed" von HackMyVM (Schwierigkeitsgrad: Easy) wurde durch eine Reihe von Schwachstellen kompromittiert. Die Enumeration offenbarte einen unauthentifizierten Memcached-Dienst, aus dem ein Passwort für einen benutzerdefinierten Netzwerkdienst auf Port 4444 extrahiert werden konnte. Dieser Dienst erlaubte das Schreiben von Dateien über einen `echo`-Befehl, was genutzt wurde, um einen SSH-Schlüssel für den Benutzer `brad` zu platzieren und so initialen Zugriff zu erlangen. Die Privilegienerweiterung zu Root erfolgte durch PATH-Hijacking: Ein Cron-Job, der als Root lief, führte einen `chown`-Befehl ohne absoluten Pfad aus. Durch Erstellen eines bösartigen `chown`-Skripts in einem für `brad` schreibbaren Verzeichnis, das im `PATH` von Root lag, konnte das SUID-Bit auf `/bin/bash` gesetzt werden, was zu einer Root-Shell führte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `nikto`
*   `dirsearch`
*   `nc` (netcat)
*   `echo`
*   `memcdump`, `memccat`
*   `ssh`
*   `find`
*   `lsb_release`
*   `pspy64`
*   `sleep`
*   `grep`
*   `diff`
*   `ps`
*   `ss`
*   `vi`/`nano` (impliziert)
*   Standard Linux-Befehle (`cat`, `chmod`, `bash`, `id`, `whoami`, `pwd`, `cd`, `ls`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Crazymed" erfolgte in diesen Schritten:

1.  **Reconnaissance & Service Enumeration:**
    *   Ziel-IP (`192.168.2.107`, Hostname `crazymed.hmv`) via `arp-scan` identifiziert.
    *   `nmap` zeigte offene Ports: 22 (SSH 8.4p1), 80 (Apache 2.4.54 - "Crazymed Bootstrap Template"), 4444 (unbekannter Dienst, der eine Passwortabfrage zeigte) und 11211 (Memcached 1.6.9 - unauthentifiziert).
    *   Web-Enumeration auf Port 80 mit `gobuster`, `nikto`, `dirsearch` fand `/changelog.txt` und das Verzeichnis `/forms/`, aber keine direkten Schwachstellen.

2.  **Initial Access (Memcached Leak & Custom Service Exploit):**
    *   Mittels `memcdump` und `memccat` wurde der Memcached-Dienst (Port 11211) ausgelesen. Der Schlüssel `log` enthielt den Wert `password: cr4zyM3d`.
    *   Dieses Passwort (`cr4zyM3d`) wurde erfolgreich für den Login beim benutzerdefinierten Dienst auf Port 4444 verwendet (`nc 192.168.2.107 4444`). Dieser Dienst gewährte eine eingeschränkte Shell als Benutzer `brad`.
    *   Die eingeschränkte Shell erlaubte den `echo`-Befehl. Dieser wurde genutzt, um den öffentlichen SSH-Schlüssel des Angreifers in `/home/brad/.ssh/authorized_keys` zu schreiben.
    *   Erfolgreicher SSH-Login als `brad` mittels Schlüsselauthentifizierung.
    *   Die User-Flag wurde aus `/home/brad/user.txt` gelesen.

3.  **Privilege Escalation (brad zu root via PATH Hijacking & Cronjob):**
    *   Enumeration als `brad` (SUID, `sudo -l` etc.) ergab keine direkten Vektoren.
    *   `pspy64` wurde auf das Zielsystem hochgeladen und ausgeführt. Es deckte einen Cron-Job auf, der minütlich als `root` das Skript `/opt/check_VM` ausführt.
    *   Analyse von `/opt/check_VM` zeigte, dass es unter anderem den Befehl `chown -R www-data:www-data /var/www/html` ohne absoluten Pfad aufrief.
    *   **PATH Hijacking:**
        1.  Ein bösartiges Skript namens `chown` wurde in einem für `brad` schreibbaren Verzeichnis erstellt, das wahrscheinlich im `PATH` von Root vor `/bin` lag (z.B. `/usr/local/bin`):
            ```bash
            #!/bin/bash
            chmod u+s /bin/bash
            ```
        2.  Das Skript wurde ausführbar gemacht (`chmod +x /usr/local/bin/chown`).
        3.  Als der Cron-Job das nächste Mal `/opt/check_VM` ausführte, wurde das bösartige `chown`-Skript (als `root`) aufgerufen, welches das SUID-Bit auf `/bin/bash` setzte.
    *   Mit `/bin/bash -p` wurde eine Root-Shell erlangt.
    *   Die Root-Flag wurde aus `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Unsicherer Memcached-Dienst:** Preisgabe von Klartext-Credentials durch unauthentifizierten Zugriff.
*   **Fehlerhafte eingeschränkte Shell:** Erlaubte das Schreiben von Dateien über `echo` und Umleitungen, was das Platzieren eines SSH-Schlüssels ermöglichte.
*   **PATH Hijacking in Cronjob:** Ein als Root laufender Cronjob führte einen Befehl (`chown`) ohne absoluten Pfad aus, was die Ausführung eines bösartigen Skripts mit demselben Namen aus einem kontrollierbaren Verzeichnis im `PATH` erlaubte.
*   **SUID Bash:** Setzen des SUID-Bits auf `/bin/bash` als Mittel zur dauerhaften oder temporären Rechteerhöhung.
*   **Informationslecks:** Web-Enumeration lieferte Hinweise auf Benutzernamen.

## Flags

*   **User Flag (`/home/brad/user.txt`):** `f70a9801673220fb56f42cf9d5ddc28b`
*   **Root Flag (`/root/root.txt`):** `b9b38d9533ca00072eff46338bf21b43`

## Tags

`HackMyVM`, `Crazymed`, `Easy`, `Memcached`, `Credential Leak`, `Custom Service Exploit`, `SSH`, `Cronjob Exploitation`, `PATH Hijacking`, `SUID Bash`, `Privilege Escalation`, `Linux`
