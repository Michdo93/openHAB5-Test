# openHAB 5 Test
Tests mit openHAB 5 und GraalPy.

---

# Installation

Hier ist eine **vollständige Schritt-für-Schritt-Anleitung**, wie du **openHAB 5** auf einem **Ubuntu 24.04+** System mit dem **GraalVM JDK 21** installierst – inklusive Setup von Zeitzone, Performance-Optimierung, und Integration des **Python 3 (GraalPy) Scripting Bindings**.

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
sudo apt-get install apt-transport-https -y
```

### 🔹 Hinzufügen des Repositories

```bash
echo 'deb [signed-by=/usr/share/keyrings/openhab.gpg] https://openhab.jfrog.io/artifactory/openhab-linuxpkg stable main' | sudo tee /etc/apt/sources.list.d/openhab.list
```

### 🔹 openHAB + Add-ons installieren

```bash
sudo apt update && sudo apt install openhab openhab-addons -y
```

### 🔹 openHAB aktivieren und starten

```bash
sudo systemctl start openhab
sudo systemctl enable openhab
```

> Danach erreichbar unter: [http://localhost:8080](http://localhost:8080)

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
> Wir installieren das  **Python Scrupting Next** Binding und nicht das **Python Scripting** Binding, obwohl es experimentell ist. Es bietet aber die Möglichkeit pip-Pakete über ein venv zu installieren.

> [!WARNING]  
> Es darf nicht gleichzeitig JS Scripting installiert sein!

> In der **Main UI**:
> → *Settings* → *Add-on Store* → *Automation* →
> 🔍 **“Python Scripting Next”** → Installieren

**Alternativ per Konsole:**

Zunächst installieren wir sshpass:

```bash
sudo apt install sshpass -y
```

Dann fügen wir den Fingerprint automatisch hinzu und speichern ihn:

```bash
sshpass -p habopen ssh -p 8101 -o StrictHostKeyChecking=accept-new openhab@localhost 'exit'
```

Zuletzt installieren wir das Binding:

```bash
sshpass -p habopen ssh -p 8101 -o StrictHostKeyChecking=no openhab@localhost 'feature:install marketplace-openhab-automation-pythonscripting'
```

---

## 🚀 6. JVM für Performance optimieren

Wir führen folgendes aus:

```bash
sudo systemctl stop openhab
sudo sed -i '/^EXTRA_JAVA_OPTS=/d' /etc/default/openhab && echo -e 'EXTRA_JAVA_OPTS="\n  -XX:+UnlockExperimentalVMOptions\n  -XX:+EnableJVMCI\n  -XX:+UseG1GC\n  -Dpolyglot.engine.WarnInterpreterOnly=false\n  -Dfile.encoding=UTF-8\n  -Xms256m -Xmx1024m\n  -Duser.timezone=Europe/Berlin\n  -Dpython.NativeModules=false\n  -Dpython.ForceIsolation=false\n"' | sudo tee -a /etc/default/openhab
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
  -Dpython.NativeModules=false
  -Dpython.ForceIsolation=false
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
import requests
import base64

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

        response = requests.get(
            "http://127.0.0.1:8080/static/webapps/Image.jpg",
            auth=("openhab", "habopen")
        )
        response.raise_for_status()

        # Base64-kodieren und vorbereiten
        encoded_string = base64.b64encode(response.content).decode('utf-8').replace("\r\n", "")
        image_data_url = "data:image/jpg;base64," + encoded_string

        Registry.getItem("testImage").postUpdate(image_data_url)

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
import requests
import base64

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

        response = requests.get(
            "http://127.0.0.1:8080/static/webapps/Image.jpg",
            auth=("openhab", "habopen")
        )
        response.raise_for_status()

        # Base64-kodieren und vorbereiten
        encoded_string = base64.b64encode(response.content).decode('utf-8').replace("\r\n", "")
        image_data_url = "data:image/jpg;base64," + encoded_string

        Registry.getItem("testImage").postUpdate(image_data_url)


        self.logger.info("Cron rule executed successfully.")
EOF
```

Für die Rules benötigt man das `requests`-Paket:

```bash
sudo -u openhab /usr/bin/python3 -m pip install --target=$OH_SITE_PACKAGES requests
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

Im nächsten Schritt erlauben wir, dass System-Packages überschrieben werden dürfen.

```bash
sudo -u openhab python3 -m pip config set global.break-system-packages true
```

## ⚙️ 9. Python Scripting Service-Konfiguration

Normalerweise müsste man das Binding auch über die UI konfigurieren können. Bei mir erschien nur eine leere weiße Seite. Gott sei Dank kann man auch Service-Dateien konfigurieren. Ich denke dies könnte daran liegen, dass dieses Next Binding experimentell ist.

Wir erstellen also folgendes:

```
cat << 'EOF' | sudo tee /etc/openhab/services/pythonscripting.cfg > /dev/null
# Use scope and import wrapper
#
# This enables a scope module and and import wrapper.
# A scope module is an encapsulated module containing all openHAB jsr223 objects and can be imported with <code>import scope</code>
# Additionally you can run an import like <code>from org.openhab.core import OpenHAB</code>
#
#org.openhab.automation.pythonscripting:scopeEnabled = true

# Install openHAB Python helper module (requires scope module)
#
# Install openHAB Python helper module to support helper classes like rule, logger, Registry, Timer etc...
# If disabled, the openHAB python helper module can be installed manually by copying it to /conf/automation/python/lib/openhab"
#
#org.openhab.automation.pythonscripting:helperEnabled = true

# Inject scope and helper objects into rules (requires helper modules)
#
# This injects the scope and helper Registry and logger into rules.
#
# 2 => Auto injection enabled only for UI and Transformation scripts (preferred)
# 1 => Auto injection enabled for all scripts
# 0 => Disable auto injection and use 'import' statements instead
#
#org.openhab.automation.pythonscripting:injectionEnabled = 2

# Enable dependency tracking
#
# Dependency tracking allows your scripts to automatically reload when one of its dependencies is updated.
# You may want to disable dependency tracking if you plan on editing or updating a shared library, but don't want all
# your scripts to reload until you can test it.
#
#org.openhab.automation.pythonscripting:dependencyTrackingEnabled = true

# Cache compiled openHAB Python modules (.pyc files)
#
# Cache the openHAB python modules for improved startup performance.<br>
# Disable this option will result in a slower startup performance, because scripts have to be recompiled on every startup.
#
#org.openhab.automation.pythonscripting:cachingEnabled = true

# Enable jython emulation
#
# This enables Jython emulation in GraalPy. It is strongly recommended to update code to GraalPy and Python 3 as the emulation can have performance degradation.
# For tips and instructions, please refer to <a href="https://www.graalvm.org/latest/reference-manual/python/Modern-Python-on-JVM">Jython Migration Guide</a>.
#
#org.openhab.automation.pythonscripting:jythonEmulation = false
EOF
sudo chown openhab:openhab /etc/openhab/services/pythonscripting.cfg
```

Zugegeben ist diese Datei jetzt vollständig auskommentiert und so wird keine Konfiguration angewendet, aber wer weiß, was in Zukunft manuell konfiguriert werden muss.

## 👽 10. Nützliche Aliase konfigurieren

```
cat << 'EOF' >> ~/.bashrc

# OpenHAB Aliase und Variablen
alias karaf="sshpass -p habopen ssh -p 8101 openhab@localhost"
alias clean_oh="sudo rm -rf /var/lib/openhab/cache/* && sudo rm -rf /var/lib/openhab/tmp/*"
alias clean_start_oh="sudo systemctl stop openhab && clean_oh && sudo systemctl start openhab"
export OH_SITE_PACKAGES=/var/lib/openhab/.cache/org.graalvm.polyglot/python/python-home/e41832d1c8ba537cb2b83ce5a12dcd2887406294/lib/python3.11/site-packages
EOF
source ~/.bashrc
```

Also evtl. ist natürlich der Pfad für `$OH_SITE_PACKAGES` anders.

## 👨‍💻 11. VEnv (Virtual Environment) aktivieren

VEnv-basierte Python-Laufzeitumgebungen sind optional, werden jedoch benötigt, um zusätzliche Module über „pip” und native Module zu unterstützen.
So, der Vorteil unserer Aliase ist, dass wir jetzt einfach `karaf` ausführen und in der Konsole sind. Das macht vieles leichter.

Wir checken zuerst `pythonscripting info` um zu erfahren, wie es genau aussieht und was wir alles benötigen und konfigurieren müssen.

```bash
openhab> pythonscripting info
Python Scripting Environment:
======================================
  Runtime:
    Bundle version: 5.0.3.202511280836
    GraalVM version: 24.2.1
    Python version: 3.11.7
    Helper lib version: 1.0.14
    VEnv state: disabled

  Directories:
    Scripts: /etc/openhab/automation/python
    Libraries: /etc/openhab/automation/python/lib
    Temp: /var/lib/openhab/tmp
    VEnv: /var/lib/openhab/cache/org.openhab.automation.pythonscripting/venv

Python Scripting Add-on Configuration:
======================================
  Python-Umgebung
    scopeEnabled: true
    helperEnabled: true
    injectionEnabled: 2

  Systemverhalten
    pipModules: 
    dependencyTrackingEnabled: true
    cachingEnabled: true
    jythonEmulation: false


```

Diese Werte werden im nächsten Schritt benötigt. Nun laden wir Graalpy-Community herunter, was wir benötigen um unser venv zu erstellen.

```bash
cd ~
wget -qO- https://github.com/oracle/graalpython/releases/download/graal-25.0.1/graalpy-community-25.0.1-linux-amd64.tar.gz | gunzip | tar xvf -
sudo mv graalpy-community-25.0.1-linux-amd64/ /opt
sudo chown openhab:openhab /opt/graalpy-community-25.0.1-linux-amd64/
cd /opt/graalpy-community-25.0.1-linux-amd64/
# The venv target dir must match your "VEnv path" of openHAB
sudo -u openhab ./bin/graalpy -m venv /var/lib/openhab/cache/org.openhab.automation.pythonscripting/venv
```

Nun installieren wir „patchelf“, das man für die native Modulunterstützung in graalpy benötigt wird (optional). Auch wenn es optional ist, würde ich es empfehlen, weil man nie weiß, was man zukünftig noch alles an Packages benötigt. Demnach einfach installieren, bevor man später irgendwelche Fehler beim Installieren von Paketen bekommt und sich wundert.

```bash
sudo apt update && sudo apt-get install patchelf -y
```

Nach diesen Schritten ist die Einrichtung von venv abgeschlossen und wird beim nächsten Neustart von openHAB automatisch erkannt.

::: Tipp VEnv-Hinweis Theoretisch kann man venvs auch mit einer nativen Python-Installation erstellen. Es wird jedoch dringend empfohlen, dafür graalpy zu verwenden. Dadurch wird eine „spezielle” Version von pip in diesem venv installiert, die gepatchte Python-Module installiert, sofern verfügbar. Dies erhöht die Kompatibilität von Python-Modulen mit graalpython.

In Containerumgebungen sollte man den Ordner „graalpy” mounten, da venv symbolische Links verwendet. :::

Meine persönliche Erfahrung ist, dass die normalen Installationen mit `pip` und `python3` nur bedingt funktionieren. Einige Python-Module nutzen halt CPython und die dann in Java einzubinden, wenn dann im Hintergrund C genutzt wird, ist schwierig. Je mehr ein Paket mit der Hardware interagieren muss, desto unwahrscheinlicher, dass das Modul dann unter Java läuft. Das Coole an dem VEnv für graalpython ist, dass wenn man hier pip nutzt, dass Pakete herunter geladen werden, die dann in für Java zu nutzenden `jar`-Bibliotheken wandelt.

Ich musste openHAB neustarten, damit das venv aktiviert ist. Hierzu führt man folgendes aus:

```bash
sudo systemctl restart openhab
```

Nachdem Neustart von openhab sieht man in der Karaf-Konsole nun folgendes:

```bash
openhab> pythonscripting info
Python Scripting Environment:
======================================
  Runtime:
    Bundle version: 5.0.3.202511280836
    GraalVM version: 24.2.1
    Python version: 3.11.7
    Helper lib version: 1.0.14
    VEnv state: enabled

  Directories:
    Scripts: /etc/openhab/automation/python
    Libraries: /etc/openhab/automation/python/lib
    Temp: /var/lib/openhab/tmp
    VEnv: /var/lib/openhab/cache/org.openhab.automation.pythonscripting/venv

Python Scripting Add-on Configuration:
======================================
  Python-Umgebung
    scopeEnabled: true
    helperEnabled: true
    injectionEnabled: 2

  Systemverhalten
    pipModules: 
    dependencyTrackingEnabled: true
    cachingEnabled: true
    jythonEmulation: false
```

Yay, hat geklappt. Also können wir weiter machen mit pip.

## 📦 12. Verwendung von pip zur Installation externer Module

Die Ausgaben, die spare ich mir. Aber man kann folgendes durchführen, um bspw. die `requests`-Library zu installieren.

```bash
openhab> pythonscripting pip install requests
```

Man kann das Ergebnis hinterher auch anzeigen lassen:

```bash
openhab> pythonscripting pip show requests
Name: requests                 
Version: 2.32.5
Summary: Python HTTP for Humans.
Home-page: https://requests.readthedocs.io
Author: Kenneth Reitz
Author-email: me@kennethreitz.org
License: Apache-2.0
Location: /var/lib/openhab/cache/org.openhab.automation.pythonscripting/venv/lib/python3.11/site-packages
Requires: certifi, charset_normalizer, idna, urllib3
Required-by: tiktoken
```

Und es geht natürlich auch pip list, um unsere installierten Pakete aufzulisten:

```
openhab> pythonscripting pip list
Package            Version     
------------------ ----------
annotated-types    0.7.0
anyio              4.12.0
blinker            1.9.0
certifi            2025.11.12
charset-normalizer 3.4.4
click              8.3.1
distro             1.9.0
Flask              3.1.2
h11                0.16.0
hpy                0.9.0
httpcore           1.0.9
httpx              0.28.1
idna               3.11
itsdangerous       2.2.0
Jinja2             3.1.6
jiter              0.12.0
MarkupSafe         3.0.3
openai             2.11.0
pip                23.2.1
pydantic           2.10.3
pydantic_core      2.27.1
PyYAML             6.0.3
regex              2025.11.3
requests           2.32.5
setuptools         65.5.0
sniffio            1.3.1
tiktoken           0.7.0
tqdm               4.67.1
typing_extensions  4.15.0
urllib3            2.6.2
Werkzeug           3.1.4
```

Das coole ist, dass man auch außerhalb der Karaf-Konsole nun ebenfalls pip nutzen kann, um Python-Module zu installieren:

```bash
sudo -u openhab /var/lib/openhab/cache/org.openhab.automation.pythonscripting/venv/bin/pip install requests
```

## 🚗 13. Python Auto-Completion aktivieren

Bevor man die Autovervollständigung aktivieren kann, muss man die erforderlichen Typ-Hinweis-Stub-Dateien generieren. Melden Sie sich bei der openHAB-Konsole an und führen Sie folgenden Befehl aus:

```bash
pythonscripting typing
```

Man sieht dann bspw. folgendes:

```bash
openhab> pythonscripting typing

You are about creating python type hint stub files in '/etc/openhab/automation/python/typings'.

Press Enter to confirm or Ctrl+C to cancel.

INFO: BUNDLE: org.openhab.core_5.0.3 [155] with 269 classes processed
INFO: BUNDLE: org.openhab.core.addon_5.0.3 [156] with 15 classes processed
INFO: BUNDLE: org.openhab.core.addon.marketplace_5.0.3 [157] with 6 classes processed
INFO: BUNDLE: org.openhab.core.audio_5.0.3 [159] with 27 classes processed
INFO: BUNDLE: org.openhab.core.automation_5.0.3 [162] with 102 classes processed
...
INFO: BUNDLE: org.openhab.core.config.discovery.usbserial_5.0.3 [278] with 4 classes processed
INFO: BUNDLE: org.openhab.core.io.transport.serial_5.0.3 [283] with 10 classes processed
INFO: BUNDLE: org.openhab.core.io.transport.serial.rxtx_5.0.3 [284] with 1 classes processed
INFO: BUNDLE: org.openhab.ui.basic_5.0.3 [313] with 2 classes processed
INFO: 1647 bundle and 365 java classes processed
INFO: Total of 2012 type hint files create in '/etc/openhab/automation/python/typings'
```

Dadurch wird die aktuelle openHAB-Instanz, einschließlich aller installierten Add-ons, nach öffentlichen Java-Klassenmethoden durchsucht und es werden entsprechende Python-Typ-Hinweis-Stub-Dateien erstellt.

Nun müsste bei der Info `Typing` und `Type hints` hinzugekommen sein:

```
openhab> pythonscripting info
Python Scripting Environment:
======================================
  Runtime:
    Bundle version: 5.0.3.202511280836
    GraalVM version: 24.2.1
    Python version: 3.11.7
    Helper lib version: 1.0.14
    VEnv state: enabled
    Type hints: available

  Directories:
    Scripts: /etc/openhab/automation/python
    Libraries: /etc/openhab/automation/python/lib
    Typing: /etc/openhab/automation/python/typings
    Temp: /var/lib/openhab/tmp
    VEnv: /var/lib/openhab/cache/org.openhab.automation.pythonscripting/venv

Python Scripting Add-on Configuration:
======================================
  Python-Umgebung
    scopeEnabled: true
    helperEnabled: true
    injectionEnabled: 2

  Systemverhalten
    pipModules: 
    dependencyTrackingEnabled: true
    cachingEnabled: true
    jythonEmulation: false


```

Die Dateien werden im Ordner `/etc/openhab/automation/python/typings/` gespeichert.
Als letzten Schritt müssen die Ordner `/etc/openhab/automation/python/libs/` und `/etc/openhab/automation/python/typings/` als „extraPaths” in deiner IDE hinzugefügt werden.

![Auto-Completion](https://github.com/openhab/openhab-addons/raw/main/bundles/org.openhab.automation.pythonscripting/doc/ide_autocompletion.png)

## 👽 14. Logging aktivieren

In der Karaf-Konsole muss man explizit das Logging aktivieren. Ohne das Logging sieht man nicht die internen Fehler von Python. Oder in anderen Worten ausgedrückt: Ich würde nur Events in openHAB sehen, wenn dies ausgeführt werden oder nicht ausgeführt werden, ob aber eine Rule vorher schon Fehler wirft, sehe ich nicht. Ich sehe in openHAB vielleicht, ob die Rule vorhanden ist, aber beim Hinzufügen der Rule sehe ich nicht, warum es scheitert, wenn eine Rule doch nicht hinzugefügt werden kann.

In `Karaf` setzen wir entsprechend das `Log Level` für uns `Python Scripting Next Binding`:

```
openhab> log:set DEBUG org.openhab.automation.pythonscripting
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
