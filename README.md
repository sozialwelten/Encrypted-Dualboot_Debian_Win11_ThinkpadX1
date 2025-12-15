# 🛡️ Fully Encrypted Dualboot 
## Cheatsheet für Debian/LUKS & Windows/VeraCrypt

#### System Details

| Komponente                  | Details                              |
|:----------------------------|:-------------------------------------|
| **Hardware**                | ThinkPad X1 Yoga Gen 8               |
| **Festplatte**              | 512 GB SSD (einzelne Platte)         |
| **Windows**                 | Windows 11 (mit integrierter Lizenz) |
| **Linux**                   | Debian 12                            |
| **Windows-Verschlüsselung** | VeraCrypt System Encryption          |
| **Linux-Verschlüsselung**   | LUKS/LVM                             |
| **Bootloader**              | GRUB                                 |

---

## Ergebnis

- GRUB startet ohne Passwort-Abfrage
- Beim Start von **Windows**: VeraCrypt fragt nach Passwort → Windows startet entschlüsselt
- Beim Start von **Debian**: LUKS fragt nach Passwort → Debian startet entschlüsselt
- Beide Systeme vollständig verschlüsselt (außer Boot-Partitionen)

## Voraussetzungen

- UEFI-System (kein Legacy/CSM)
- Secure Boot **deaktiviert** (BIOS-Einstellung)
- TPM kann aktiviert bleiben
- Backup aller Daten
- Windows 11 Installationsmedium (USB)
- Debian Installationsmedium (USB)
- VeraCrypt für Windows herunterladen

---

## Phase 1: Windows 11 Installation (unverschlüsselt)

### 1.1 BIOS-Einstellungen prüfen

```
Security:
├─ Secure Boot: OFF ✓
├─ TPM 2.0: ON (kann bleiben)
└─ Boot Mode: UEFI Only

Boot:
└─ Fast Boot: Disabled (während Installation)
```

### 1.2 Windows installieren

1. Von USB booten
2. **Wichtig bei Partitionierung:**
   - **NICHT die ganze Festplatte verwenden!**
   - Beispiel bei 512GB SSD:
     - EFI: ~512MB (automatisch)
     - Windows: ~200-250GB
     - **Rest unallocated lassen für Debian**
3. Installation durchführen
4. **BitLocker NICHT aktivieren**
5. System vollständig einrichten und Updates installieren

### 1.3 Netzwerk-Umgehung bei Installation

Falls "Internet erforderlich" blockiert:

```cmd
Shift + F10
OOBE\BYPASSNRO
```

System startet neu, dann Option "Ich habe kein Internet" verfügbar.

### 1.4 Treiber nachinstallieren

Nach Offline-Installation:
- USB-Tethering über Smartphone, oder
- LAN-Kabel anschließen
- Windows Update ausführen
- Lenovo System Update installieren (für alle Treiber)

---

## Phase 2: Debian mit Verschlüsselung installieren

### 2.1 Kritische Punkte bei der Partitionierung

**⚠️ WICHTIG: Die richtige Reihenfolge beachten!**

#### Schritt 1: Boot-Partition erstellen

```
Aus freiem Speicher:
├─ Größe: 1 GB
├─ Typ: Primär
├─ Position: Anfang
├─ Dateisystem: ext4
├─ Einbindungspunkt: /boot
├─ Boot-Flag: AUS (gehört auf EFI!)
└─ Reservierte Blöcke: 0-1% (bei 1GB nicht nötig)
```

#### Schritt 2: Verschlüsselte Partition erstellen

```
Aus verbleibendem freiem Speicher:
├─ Größe: Alle verbleibenden GB
├─ Typ: Primär
├─ Position: Anfang
└─ Benutzen als: Physikalisches Volume für Verschlüsselung
    └─ KEIN Einbindungspunkt!
    └─ NICHT als ext4 formatieren!
```

#### Schritt 3: Verschlüsselung konfigurieren

1. Menü: **"Verschlüsselte Datenträger konfigurieren"**
2. Änderungen auf Festplatte schreiben: **Ja**
3. **"Verschlüsselten Datenträger erzeugen"**
4. Die große Partition auswählen
5. **Starkes Passwort eingeben** (bei jedem Boot erforderlich!)
6. Warten (Partition wird mit Zufallsdaten überschrieben, 15-45 Min)
7. **"Fertigstellen"**

#### Schritt 4: LVM auf verschlüsselter Partition

Zurück in Partitionsliste sollte jetzt erscheinen:
- Verschlüsselter Container (~277GB oder größer)

```
Verschlüsselten Container auswählen:
└─ Benutzen als: Physikalisches Volume für LVM
    └─ KEIN Einbindungspunkt!
```

Änderungen schreiben: **Ja**

#### Schritt 5: Logical Volume Manager konfigurieren

**⚠️ WARNUNG:** Diese Meldung ist normal und KORREKT:

> "Nachdem die Partitionen geschrieben wurden, sind weitere Änderungen an der physischen Partition nicht mehr erlaubt"

Das betrifft nur die physische 277GB Partition - LVM innerhalb kann trotzdem konfiguriert werden!

1. Menü: **"Logical Volume Manager konfigurieren"**
2. Änderungen schreiben: **Ja**
3. **"Volume-Gruppe anlegen"**
   - Name: z.B. "vg0" oder "debian-vg"
   - Das verschlüsselte LVM Physical Volume auswählen
4. **Logisches Volume anlegen** (Swap):
   - Volume-Gruppe: vg0
   - Name: swap
   - Größe: 16-20GB (= RAM-Größe oder etwas mehr)
5. **Logisches Volume anlegen** (Root):
   - Volume-Gruppe: vg0
   - Name: root
   - Größe: Alle verbleibenden GB
6. **"Fertig"**

#### Schritt 6: Logical Volumes formatieren

Zurück in Partitionsliste:

```
LV swap:
└─ Benutzen als: Auslagerungsspeicher (Swap)

LV root:
├─ Benutzen als: Ext4-Journaling-Dateisystem
├─ Formatieren: Ja
├─ Einbindungspunkt: /
├─ Einbindungsoptionen: defaults
└─ Reservierte Blöcke: 1-5%
```

#### Schritt 7: EFI-Partition einbinden

**⚠️ NICHT neu formatieren!**

```
Vorhandene EFI-Partition (von Windows):
├─ Typ: ESP oder "EFI System Partition"
├─ Größe: ~100-512MB
├─ Dateisystem: FAT32/vfat
├─ Benutzen als: EFI System Partition
├─ Formatieren: NEIN! ✗
└─ Einbindungspunkt: /boot/efi
```

### 2.2 Partitions-Übersicht (Endergebnis)

```
/dev/nvme0n1p1    512 MB    ESP (EFI)           [von Windows, nicht formatiert]
/dev/nvme0n1p2    ~220 GB   Windows (NTFS)      [später mit VeraCrypt verschlüsselt]
/dev/nvme0n1p3    1 GB      /boot (ext4)        [unverschlüsselt]
/dev/nvme0n1p4    ~277 GB   LUKS verschlüsselt
  └─ LVM Volume Group (vg0)
      ├─ swap   16-20 GB   Swap
      └─ root   ~257 GB    / (ext4)
```

### 2.3 Software-Auswahl

```
✓ Debian Desktop Environment
✓ GNOME (oder KDE/XFCE/Mate)
✓ Standard-Systemwerkzeuge
✗ Webserver
✗ SSH-Server (außer explizit benötigt)
✗ SQL-Datenbank
✗ Print-Server
```

### 2.4 GRUB Installation

- **Ja** - GRUB in Master Boot Record installieren
- Gerät: **/dev/nvme0n1** (ganze Festplatte, NICHT eine Partition!)
- GRUB erkennt Windows automatisch

---

## Phase 3: Windows mit VeraCrypt verschlüsseln

### 3.1 VeraCrypt installieren

1. Windows starten (über GRUB)
2. VeraCrypt herunterladen: https://www.veracrypt.fr/en/Downloads.html
3. Als Administrator installieren
4. **Screenshot-Schutz aktiviert lassen** (Standard)
5. **Windows-Schnellstart deaktivieren:** Ja (VeraCrypt fragt automatisch)

### 3.2 System-Verschlüsselung starten

1. VeraCrypt starten (als Administrator)
2. **System** → **Encrypt System Partition/Drive**
3. Optionen:
   ```
   Typ: Normal (nicht Hidden)
   Bereich: Encrypt the Windows system partition
   Betriebssysteme: Multi-boot ← WICHTIG wegen Debian!
   Algorithmus: AES (Standard)
   Hash: SHA-512 (Standard)
   ```
4. **Passwort wählen:**
   - Starkes Passwort
   - **⚠️ WICHTIG:** Kein Z/Y (sind vertauscht im Bootloader!)
   - **⚠️ WICHTIG:** Keine Umlaute (ä,ö,ü,ß)
   - Nur: a-z, A-Z, 0-9, einfache Sonderzeichen
   - VeraCrypt-Bootloader nutzt US-Tastatur-Layout!
5. **PIM:** Nicht verwenden (leer lassen)
6. **Rescue Disk:**
   - ISO wird erstellt
   - **Unbedingt auf USB-Stick speichern!**
   - Im Notfall benötigt
7. **Löschmodus:** Ohne (am schnellsten) - ausreichend bei Verschlüsselung
8. **Pretest:** System startet neu

### 3.3 Pretest und Verschlüsselung

**Nach Neustart:**
1. VeraCrypt-Bootloader erscheint
2. Passwort eingeben (US-Tastatur-Layout beachten!)
3. GRUB startet
4. Windows wählen
5. Windows sollte normal starten

**Pretest erfolgreich:**
1. VeraCrypt meldet sich (oder manuell öffnen)
2. **System** → **Resume Interrupted Process**
3. Verschlüsselung starten: **Ja**
4. Dauer: 30-60 Minuten (nicht unterbrechen!)

---

## Häufige Probleme und Lösungen

### Problem: Windows bootet nach Debian-Installation nicht mehr

**Symptom:** Nach Debian-Installation startet Windows in "Automatische Reparatur"

**Ursache:** Windows-Bootloader durch GRUB irritiert

**Lösung:**
1. Automatische Reparatur durchlaufen lassen
2. Falls fehlgeschlagen: Windows-Installations-USB
3. Computer reparieren → Problembehandlung → Eingabeaufforderung
4. EFI-Partition Laufwerksbuchstaben zuweisen:
   ```cmd
   diskpart
   list volume
   select volume [EFI-Partition, FAT32, ~200MB]
   assign letter=S
   exit
   bcdboot C:\Windows /s S: /f UEFI /l de-de
   ```

### Problem: GRUB erscheint nach Windows-Installation nicht mehr

**Symptom:** Nach Windows-Installation bootet System direkt in Windows

**Ursache:** Windows hat GRUB als primären Bootloader überschrieben

**Lösung:** Von Debian-USB im Rescue Mode booten

```bash
# Verschlüsselte Partition entsperren
sudo cryptsetup luksOpen /dev/nvme0n1p4 cryptroot

# Partitionen mounten
sudo mount /dev/mapper/vg0-root /mnt
sudo mount /dev/nvme0n1p3 /mnt/boot
sudo mount /dev/nvme0n1p1 /mnt/boot/efi

# In System wechseln
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt

# GRUB neu installieren
grub-install /dev/nvme0n1
update-grub
exit

# Aufräumen
sudo umount /mnt/boot/efi /mnt/boot /mnt/dev /mnt/proc /mnt/sys /mnt
sudo reboot
```

**Alternative:** Boot-Reihenfolge im BIOS ändern (GRUB/debian vor Windows Boot Manager)

### Problem: Debian bootet nicht, hängt beim Start

**Symptom:** Nach LUKS-Passwort hängt System bei "Timed out waiting for device" oder Emergency Mode

**Häufigste Ursache:** Problem in `/etc/fstab` (meist Swap-Eintrag)

**Lösung:** Von Debian-USB im Rescue Mode booten

```bash
# In Root-Shell
mount -o remount,rw /
nano /etc/fstab

# Problematische Zeile (meist swap) auskommentieren:
# /dev/mapper/debian--vg-swap none swap sw 0 0

# Speichern (Ctrl+O), Beenden (Ctrl+X)
exit
# Neustart
```

**Swap später als Datei nachträglich einrichten** (optional):
```bash
sudo fallocate -l 16G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
# Dauerhaft: echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Problem: WLAN funktioniert nach Debian-Installation nicht

**Symptom:** `iwlwifi firmware failed to load`

**Lösung:** Firmware nachinstallieren

```bash
sudo apt update
sudo apt install firmware-iwlwifi
sudo reboot
```

Falls kein Internet verfügbar: USB-Tethering über Smartphone nutzen

### Problem: Basissystem-Installation schlägt fehl

**Symptom:** `debootstrap wurde mit einem Fehler beendet (Rückgabewert 1)`

**Ursache:** Meist Netzwerkproblem oder Dateisystem-Problem

**Lösungen:**
1. Anderen Mirror-Server wählen (z.B. ftp.de.debian.org)
2. Mit USB-Tethering stabiles Internet sicherstellen
3. Bei wiederholten Fehlern: Partitionierung nochmal komplett neu (alle LVM/LUKS-Partitionen löschen und neu erstellen)

### Problem: VeraCrypt "Wrong password" beim Pretest

**Ursache:** VeraCrypt-Bootloader nutzt US-Tastatur-Layout

**Wichtige Unterschiede DE → US:**
- Z und Y sind vertauscht
- Umlaute (ä,ö,ü,ß) existieren nicht
- Sonderzeichen an anderen Positionen

**Lösung:** Passwort ohne diese Zeichen wählen oder mit US-Layout eingeben

**Tastatur-Test:** Im VeraCrypt-Bootloader **F5** drücken

---

## Boot-Reihenfolge (Endergebnis)

```
UEFI/BIOS
    ↓
GRUB (unverschlüsselt, von /boot)
    ├─→ Debian gewählt
    │   ↓
    │   LUKS Passwort-Abfrage
    │   ↓
    │   Debian startet (entschlüsselt)
    │
    └─→ Windows Boot Manager gewählt
        ↓
        VeraCrypt Passwort-Abfrage
        ↓
        Windows startet (entschlüsselt)
```

---

## Sicherheitshinweise

### Passwörter

- Zwei verschiedene Passwörter: LUKS (Debian) und VeraCrypt (Windows)
- Können identisch oder unterschiedlich sein
- VeraCrypt-Passwort: Keine Z/Y/Umlaute wegen US-Tastatur
- Beide Passwörter gut merken - bei Verlust keine Datenrettung möglich!

### Rescue Disk

- VeraCrypt Rescue Disk auf USB-Stick sichern
- Im Notfall kann damit der Bootloader repariert werden
- Ohne Rescue Disk: Bei Boot-Problemen schwierige Datenrettung

### Updates

- Windows-Updates können Boot-Probleme verursachen
- Bei Windows-Updates: Immer Rescue Disk griffbereit haben
- Nach größeren Windows-Updates: `sudo update-grub` in Debian ausführen

### TPM

- TPM kann aktiviert bleiben
- VeraCrypt nutzt TPM nicht (Passwort-Eingabe immer erforderlich)
- BitLocker NICHT parallel zu VeraCrypt verwenden

---

## Tipps und Optimierungen

### Boot-Zeit optimieren

In Debian:
```bash
# Timeout in GRUB reduzieren
sudo nano /etc/default/grub
# GRUB_TIMEOUT=5 → GRUB_TIMEOUT=2
sudo update-grub
```

### Swap-Größe

- Mindestens RAM-Größe (für Suspend-to-Disk/Hibernate)
- Ohne Hibernate: 8-16GB ausreichend
- Bei wenig Speicher: Swap als Datei statt LV flexibler

### Alternative ohne LVM

Einfachere Variante (ohne Swap-Partition):
```
/boot             1 GB        ext4 (unverschlüsselt)
Verschlüsselt     277 GB      ext4 direkt als /
```
Swap später als Datei: siehe "Problem: Debian bootet nicht"

Vorteile: Einfacher, weniger Fehlerquellen
Nachteil: Swap nicht als eigene Partition

---

## Checkliste

### Vor der Installation

- [ ] Backup aller Daten erstellt
- [ ] Windows 11 USB-Stick erstellt
- [ ] Debian USB-Stick erstellt
- [ ] VeraCrypt heruntergeladen
- [ ] BIOS-Einstellungen geprüft (Secure Boot OFF)

### Nach der Installation

- [ ] Beide Systeme booten erfolgreich
- [ ] GRUB zeigt beide Betriebssysteme
- [ ] Windows vollständig eingerichtet und aktualisiert
- [ ] Debian vollständig aktualisiert (`sudo apt update && sudo apt upgrade`)
- [ ] WLAN/Treiber funktionieren
- [ ] VeraCrypt Rescue Disk auf USB gesichert
- [ ] Boot-Zeit akzeptabel
- [ ] Alle Passwörter sicher notiert

---

## Quellen und weiterführende Links

- VeraCrypt: https://www.veracrypt.fr/
- Debian Wiki: https://wiki.debian.org/FullDiskEncryption
- Debian Installer: https://www.debian.org/distrib/
- Windows 11: https://www.microsoft.com/software-download/windows11

---

**Erstellt:** Dezember 2024  
**Getestet auf:** ThinkPad X1 Yoga Gen 8, 512GB SSD  
**Systeme:** Windows 11, Debian 12  

Bei Fragen oder Problemen: Issue auf GitHub öffnen!