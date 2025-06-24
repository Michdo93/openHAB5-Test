# openHAB 5 Test
Tests mit openHAB 5, vor allem Jython und GraalPy

---

# Installation

Hier ist eine **vollständige Schritt-für-Schritt-Anleitung**, wie du **openHAB 5** auf einem **Ubuntu 22.04+** System mit dem **GraalVM JDK 21** installierst – inklusive Setup von Zeitzone, Performance-Optimierung, und Integration des **Python 3 (GraalPy) Scripting Bindings**.

---

## 🛠️ 1. System vorbereiten

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl unzip wget gnupg lsb-release -y
```

---

## ☕ 2. GraalVM JDK 21 installieren (Community Edition)

### 🔹 Variante: Manuell installieren (empfohlen für Server)

```bash
cd /tmp
wget https://download.oracle.com/graalvm/21/archive/graalvm-jdk-21.0.6_linux-x64_bin.tar.gz

sudo mkdir -p /usr/lib/jvm
sudo tar -xzf graalvm-jdk-21.0.6_linux-x64_bin.tar.gz -C /usr/lib/jvm
sudo ln -s /usr/lib/jvm/graalvm-jdk-21.0.6+8.1 /usr/lib/jvm/graalvm21
```

### 🔹 Java-Systempfade setzen

```bash
sudo update-alternatives --install /usr/bin/java java /usr/lib/jvm/graalvm21/bin/java 100
sudo update-alternatives --install /usr/bin/javac javac /usr/lib/jvm/graalvm21/bin/javac 100
sudo update-alternatives --config java     # → GraalVM auswählen
```

> Prüfen:

```bash
java -version
java 21.0.6 2025-01-21 LTS
Java(TM) SE Runtime Environment Oracle GraalVM 21.0.6+8.1 (build 21.0.6+8-LTS-jvmci-23.1-b55)
Java HotSpot(TM) 64-Bit Server VM Oracle GraalVM 21.0.6+8.1 (build 21.0.6+8-LTS-jvmci-23.1-b55, mixed mode, sharing)
```

### 🔹 JAVA_HOME-Variable setzen

```bash
sudo sed -i '/^JAVA_HOME=/d' /etc/environment && echo 'JAVA_HOME="/usr/lib/jvm/graalvm21"' | sudo tee -a /etc/environment
sudo reboot
```

---

## 🧱 3. openHAB 5 installieren

### 🔹 Repository Key hinzufügen

```bash
curl -fsSL "https://openhab.jfrog.io/artifactory/api/gpg/key/public" | gpg --dearmor > openhab.gpg
sudo mkdir /usr/share/keyrings
sudo mv openhab.gpg /usr/share/keyrings
sudo chmod u=rw,g=r,o=r /usr/share/keyrings/openhab.gpg
```

### 🔹 Hinzufügen des HTTPS-Transports für APT

```bash
sudo apt-get install apt-transport-https
```

### 🔹 Hinzufügen des Repositories

```bash
echo 'deb [signed-by=/usr/share/keyrings/openhab.gpg] https://openhab.jfrog.io/artifactory/openhab-linuxpkg testing main' | sudo tee /etc/apt/sources.list.d/openhab.list
```

### 🔹 openHAB + Add-ons installieren

```bash
sudo apt update
sudo apt install openhab openhab-addons -y
```

### 🔹 openHAB aktivieren und starten

```bash
sudo systemctl start openhab
sudo systemctl enable openhab
```

> Danach erreichbar unter: [http://localhost:8080](http://localhost:8080)

### 🔹 Repository umstellen

```bash
sudo rm /etc/apt/sources.list.d/openhab.list
echo "deb [signed-by=/usr/share/keyrings/openhab.gpg] https://openhab.jfrog.io/artifactory/openhab-linuxpkg testing main" | \
  sudo tee /etc/apt/sources.list.d/openhab.list
```

### 🔹 Paketliste aktualisieren und openHAB installieren

```bash
sudo apt update
sudo apt install openhab
```

---

## 🌍 4. Zeitzone setzen (Europe/Berlin)

### 🔹 Systemweit

```bash
sudo timedatectl set-timezone Europe/Berlin
```

### 🔹 openHAB-spezifisch (optional)

Wir fügen nun `-Duser.timezone=Europe/Berlin` in der Variable `EXTRA_JAVA_OPTS` in der `/etc/default/openhab`-Datei hinzu:

```ini
sudo sed -i '/^EXTRA_JAVA_OPTS=/d' /etc/default/openhab && echo 'EXTRA_JAVA_OPTS="-Duser.timezone=Europe/Berlin"' | sudo tee -a /etc/default/openhab
```

---

## 🧠 5. Python 3 Binding installieren (GraalPy)

> In der **Main UI**:
> → *Settings* → *Add-on Store* → *Automation* →
> 🔍 **“Python 3 Scripting”** → Installieren

**Alternativ per Konsole:**

Zunächst installieren wir sshpass:

```bash
sudo apt install -y sshpass
```

Dann fügen wir den Fingerprint automatisch hinzu und speichern ihn:

```bash
sshpass -p habopen ssh -p 8101 -o StrictHostKeyChecking=accept-new openhab@localhost 'exit'
```

Zuletzt installieren wir das Binding:

```bash
sshpass -p habopen ssh -p 8101 -o StrictHostKeyChecking=no openhab@localhost 'feature:install openhab-automation-pythonscripting'
```

---

## 🚀 6. JVM für Performance optimieren

Wir führen folgendes aus:

```bash
sudo sed -i '/^EXTRA_JAVA_OPTS=/d' /etc/default/openhab && echo -e 'EXTRA_JAVA_OPTS="\n  -XX:+UnlockExperimentalVMOptions\n  -XX:+EnableJVMCI\n  -XX:+UseG1GC\n  -Dpolyglot.engine.WarnInterpreterOnly=false\n  -Dfile.encoding=UTF-8\n  -Xms256m -Xmx1024m\n  -Duser.timezone=Europe/Berlin\n"' | sudo tee -a /etc/default/openhab
```

Dies erstellt folgendes:

```ini
EXTRA_JAVA_OPTS="
  -XX:+UnlockExperimentalVMOptions
  -XX:+EnableJVMCI
  -XX:+UseG1GC
  -Dpolyglot.engine.WarnInterpreterOnly=false
  -Dfile.encoding=UTF-8
  -Xms256m -Xmx1024m
  -Duser.timezone=Europe/Berlin
"
```

Dann:

```bash
sudo rm -rf /var/lib/openhab/tmp/* && sudo rm -rf /var/lib/openhab/cache/*
sudo systemctl restart openhab
```

---

## 🧪 7. Testen

### 🔹 Beispielregel (Datei: `hello.py` im scripts-Ordner)

```python
from core.rules import rule
from core.triggers import when
from core.log import logging

log = logging.getLogger("HelloPython")

@rule("Hello Rule")
@when("System started")
def hello(event):
    log.info("✅ Python 3.11 läuft mit GraalVM & openHAB 5!")
```

Speichern unter:
`/etc/openhab/automation/jsr223/python/personal/hello.py`

> Logausgabe prüfen:

```bash
sudo journalctl -u openhab -f
```

---

## ✅ Zusammenfassung

| Komponente      | Version / Status                     |
| --------------- | ------------------------------------ |
| Java-Umgebung   | GraalVM JDK 21 (inkl. JVM + GraalPy) |
| openHAB         | 5.x (stable)                         |
| Python Binding  | „Python 3 Scripting“ (GraalPy 3.11)  |
| Zeitzone        | Systemweit + JVM: Europe/Berlin      |
| JVM optimiert?  | ✅ Via `EXTRA_JAVA_OPTS`              |
| Zukunftssicher? | ✅ Voll auf Java 21 + Python 3        |

---

Wenn du magst, helfe ich dir im nächsten Schritt beim **Migrieren von Jython-Skripten** oder beim **Profiling langsamer Regeln**.
