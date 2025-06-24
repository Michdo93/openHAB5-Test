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

> [!WARNING]  
> Es darf nicht gleichzeitig JS Scripting installiert sein!

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
sudo systemctl stop openhab
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
sudo systemctl start openhab
```

---

## 🧪 7. Testen

### 🔹 Beispielregel (Datei: `hello.py` im scripts-Ordner)

```bash
sudo mkdir -p /etc/openhab/automation/python/personal
sudo touch /etc/openhab/automation/python/personal/hello.py
sudo chown -R openhab:openhab /etc/openhab/automation/python/personal/hello.py
```

```python
cat << 'EOF' | sudo tee /etc/openhab/automation/python/personal/hello.py > /dev/null
from openhab import rule
from openhab.triggers import GenericCronTrigger

@rule( triggers = [ GenericCronTrigger("*/5 * * * * ?") ] )
class Test:
    def execute(self, module, input):
        self.logger.info("Rule was triggered")
EOF
```

> Logausgabe prüfen:

```bash
sudo journalctl -u openhab -f
```

### 🔹 openHAB Static Examples

Im nächsten Schritt installieren wie die openHAB Static Examples:

```bash
cd ~
git clone https://github.com/Michdo93/openhab_static_examples
cd openhab_static_examples
rm -r rules/*
sudo cp -r * /etc/openhab
cd ..
sudo rm -r openhab_static_examples
sudo chown -R openhab:openhab /etc/openhab
```

Da die Rules dort als DSL Rules geschrieben sind löschen wir diese und erstellen unsere Python-Rules manuell neu:

```bash
sudo mkdir -p /etc/openhab/automation/python/personal
sudo touch /etc/openhab/automation/python/personal/static.py
sudo touch /etc/openhab/automation/python/personal/cron.py
sudo chown -R openhab:openhab /etc/openhab/automation/python/personal
```

Für den Systemstart:

```python
cat << 'EOF' | sudo tee /etc/openhab/automation/python/personal/static.py > /dev/null
from datetime import datetime
from openhab           import rule, Registry
from openhab.triggers  import SystemStartlevelTrigger   # Startlevel-Trigger
from scope             import NULL                      # Konstanten (ON, OFF, NULL)

# --------------------------------------------------------------------

@rule(triggers=[SystemStartlevelTrigger(100)])          # Level 100 = „System ready“
class Started:
    def execute(self, module, input):

        # 1) Items auf NULL setzen
        demo_items = (
            "testColor", "testContact", "testDateTime", "testDimmer",
            "testNumber", "testPlayer", "testRollershutter", "testString",
            "testSwitch", "testLocation", "testImage"
        )
        for name in demo_items:
            Registry.getItem(name).postUpdate(NULL)

        # 2) Beispiel-Werte schreiben
        Registry.getItem("testColor").postUpdate("120,100,100")
        Registry.getItem("testContact").postUpdate("CLOSED")
        Registry.getItem("testDateTime").postUpdate(
            datetime.now().strftime("%Y-%m-%dT%H:%M:%S.%f%z")
        )
        Registry.getItem("testDimmer").postUpdate("90")
        Registry.getItem("testNumber").postUpdate("50")
        Registry.getItem("testPlayer").postUpdate("PAUSE")
        Registry.getItem("testRollershutter").postUpdate("0")
        Registry.getItem("testString").postUpdate("Hello World")
        Registry.getItem("testSwitch").postUpdate("OFF")

        # Location als einfacher lat,lon,alt-String genügt
        Registry.getItem("testLocation").postUpdate(
            "48.051437316054006,8.207755911376244,857.0"
        )

        self.logger.info("Started rule executed successfully.")
EOF
```

Für den Cronjob:

```python
cat << 'EOF' | sudo tee /etc/openhab/automation/python/personal/cron.py > /dev/null
from datetime import datetime
from openhab           import rule, Registry
from openhab.triggers  import GenericCronTrigger   # Startlevel-Trigger
from scope             import NULL                      # Konstanten (ON, OFF, NULL)

# --------------------------------------------------------------------

@rule(triggers=[GenericCronTrigger("0 0/1 * * * ?")])
class Cron:
    def execute(self, module, input):
        Registry.getItem("testColor").postUpdate("0,100,100")
        Registry.getItem("testContact").postUpdate("OPEN")
        Registry.getItem("testDateTime").postUpdate(
            datetime.now().strftime("%Y-%m-%dT%H:%M:%S.%f%z")
        )
        Registry.getItem("testDimmer").postUpdate("30")
        Registry.getItem("testNumber").postUpdate("0")
        Registry.getItem("testPlayer").postUpdate("PLAY")
        Registry.getItem("testRollershutter").postUpdate("100")
        Registry.getItem("testString").postUpdate("Hello")
        Registry.getItem("testSwitch").postUpdate("ON")

        # Location als einfacher lat,lon,alt-String genügt
        Registry.getItem("testLocation").postUpdate(
            "48.051437316054006,8.207755911376244,857.0"
        )

        self.logger.info("Cron rule executed successfully.")
EOF
```

---

## 🧪 8. Python Libraries hinzufügen

Finde heraus, wo openHAB seine Automation-Umgebung installiert hat:

```bash
sudo find /var/lib/openhab -type d -name "site-packages"
```

Ein typischer Pfad ist zum Beispiel:

```bash
/var/lib/openhab/.cache/org.graalvm.polyglot/python/python-home/3fe684bec8ba537cb2b83ce56e79ca177f1e0290/lib/python3.11/site-packages
```

Wenn du mehrere Pakete brauchst oder öfter arbeitest, kannst du dir z. B. eine Umgebungsvariable setzen:

```bash
export OH_SITE_PACKAGES=/var/lib/openhab/.cache/org.graalvm.polyglot/python/python-home/3fe684bec8ba537cb2b83ce56e79ca177f1e0290/lib/python3.11/site-packages
```

Im nächsten Schritt stellen wir sicher, dass `pip3` auch wirklich systemweit installiert ist:

```bash
sudo apt install -y python3-pip
```

Neue Pakete kannst du absofort wie folgt installieren:

```bash
sudo -u openhab /usr/bin/python3 -m pip install --target=$OH_SITE_PACKAGES <paketname>
```

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
