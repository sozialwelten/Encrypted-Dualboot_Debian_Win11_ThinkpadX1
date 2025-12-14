# 🛡️ Fully Encrypted Dualboot Cheatsheet (Debian/LUKS & Windows/VeraCrypt)

#### System Details

| Komponente | Details                              |
| :--- |:-------------------------------------|
| **Hardware** | ThinkPad X1 Yoga Gen 8               |
| **Windows** | Windows 11 (mit integrierter Lizenz) |
| **Linux** | Debian 12                            |
| **Windows-Verschlüsselung** | VeraCrypt System Encryption          |
| **Linux-Verschlüsselung** | LUKS/LVM                             |
| **Bootloader** | GRUB                                 |

---

### Der Ablauf: Die Strategie, die funktioniert hat

Der Schlüssel zu diesem Setup ist die Reihenfolge. **GRUB muss die Kontrolle über den Bootvorgang übernehmen**, aber unverschlüsselt bleiben.

1.  **Vorbereitung:**
    * Komplettes Backup aller Daten (obligatorisch).
    * Windows 11 installieren. Nur die notwendige Partition erstellen (z.B. 50% der Platte). **Wichtig:** Den restlichen Speicherplatz als unpartitionierten freien Speicher lassen.
2.  **Debian/LUKS-Installation:**
    * Debian installieren und den freien Speicherplatz nutzen.
    * **Partitionierung:** Eine verschlüsselte LVM-Partition (mit LUKS) für `/` und `swap` erstellen.
    * **Bootloader-Installation:** GRUB in die **unverschlüsselte EFI-Partition** installieren (wo auch der Windows Boot Manager liegt).
3.  **VeraCrypt-Verschlüsselung (Windows):**
    * In Windows booten.
    * VeraCrypt starten: `System` → `Encrypt System Partition/Drive`.
    * Die **VeraCrypt Rescue Disk** (ISO) erstellen und diese unbedingt auf einen USB-Stick kopieren.
4.  **Der Pretest:**
    * VeraCrypt startet das System neu für den Pretest.
    * Beim Neustart das VeraCrypt-Passwort eingeben.
    * Der Test sollte erfolgreich sein und dich anschließend zum **GRUB-Menü** führen, wo du Windows wieder starten kannst.

---

### 🚨 Die typischen Fallstricke (Der "Leidensweg"-Fix)

Der Hauptstolperstein (und der Grund für dieses Cheatsheet) war, dass VeraCrypt nach dem erfolgreichen Pretest nicht automatisch mit der eigentlichen Systemverschlüsselung begann.

#### Fallstrick 1: VeraCrypt fragt nicht nach Verschlüsselung

Nach dem erfolgreichen Pretest startet Windows, aber VeraCrypt fragt nicht, ob die Verschlüsselung beginnen soll.

| ❌ Problem | ✅ Lösung (Der Fix) |
| :--- | :--- |
| VeraCrypt-Status ist unklar, die Verschlüsselung startet nicht von selbst. | 1. VeraCrypt **als Administrator** öffnen. |
| | 2. Im Hauptfenster: **`System` → `Resume Interrupted Process`** wählen. |
| | 3. Alternativ: **`System` → `Encrypt System Partition/Drive`** wählen. |

VeraCrypt sollte nun erkennen, dass der Pretest erfolgreich war, und die eigentliche, langwierige Verschlüsselung starten.

#### Fallstrick 2: Der Boot-Mechanismus

* **Der Boot-Ablauf:** BIOS/UEFI → GRUB (unverschlüsselt) → VeraCrypt (Passwort) → Windows
* **Wichtig:** Der Windows-Boot-Manager darf nicht das Standard-Boot-Target sein, da er die Debian/GRUB-Partition nicht sehen kann. GRUB muss das erste Target in der EFI-Partition sein.

---

### 💾 Nützliche Kommandos / GUI-Pfade

Da es sich hier um einen Prozess handelt, der stark auf GUIs (Debian-Installer, VeraCrypt) beruht, sind die entscheidenden "Befehle" die Klicks:

| Aktion | Ort / Kommando |
| :--- | :--- |
| **Start der Verschlüsselung fortsetzen** | VeraCrypt: `System` → `Resume Interrupted Process` |
| **Neuinstallation Windows/Debian** | Sicherstellen, dass GRUB in der **EFI-Partition** installiert wird. |
| **Rettung** | VeraCrypt Rescue Disk (falls das Passwort nicht angenommen wird) |