# build-mainline-kernel
Build mainline Linux kernel for Ubuntu and Linux Mint

**Der aktuelle Linux-Kernel 5.15 bietet interessante Funktionen für alle Nutzer. Wer möchte, kann den Kernel schon jetzt auch unter Ubuntu 20.04 oder Linux Mint 20 einsetzen.**

Bei der Kernel-Version 5.15 sind zwei spannende Funktionen hinzugekommen. Neu ist ein eigenes Modul für Dateifreigaben über das SMB/CIFS-Protokoll, wofür bisher der Samba-Server diente. Das Kernel-Modul soll vor allem für mehr Geschwindigkeit bei der Datenübertragung sorgen.

Eine weitere Neuerung ist ein Treiber für das NTFS-Dateisystem mit Lese- und Schreibzugriff und mit verbesserter Leitung. Das bisherige Kernel-Modul bietet nur Lesezugriff, weshalb zusätzlich das Paket NTFS-3G erforderlich ist.

## Risiken und Nebenwirkungen
Die Installation eines neueren Kernels ist nur selten mit Problemen verbunden. Da ältere Kernel-Versionen stets erhalten bleiben, kann man bei Problemen schnell auf die vorherige Version wechseln. Dazu hält man beim Start des Rechners die Shift-Taste gedrückt, um in das Grub-Bootmenü zu gelangen. Unter „Erweiterte Optionen für Ubuntu“ beziehungsweise „Erweiterte Optionen für Linux Mint“ lässt sich ein älterer Kernel wählen.

Bei relativ aktuellen Kerneln können zusätzliche Treiber Probleme verursachen. Ist beispielsweise der proprietäre Treiber für einen Nvidia-Grafikchip aktiv, wird dieser bei einem Kernel-Update neu kompiliert. Sollte der Treiber den Kernel noch nicht unterstützen, schlägt das fehl und der Treiber kann nicht geladen werden. In diesem Fall muss man eine neuere Version des Treiberpakets installieren, die mit dem Kernel zusammenarbeitet.

Der Kernel 15.5 wird bei www.kernel.org als stabile Version geführt. Allerdings ist der Code für den NTFS-Treiber noch recht neu, auch wenn er auf einem seit langem genutzten Treiber von Paragon basiert (www.paragon-software.com). Fehler sind daher nicht ausgeschlossen, weshalb vor der Nutzung ein Backup der NTFS-Partition anzuraten ist.

Beim neuen Samba-Modul besteht zwar nicht die Gefahr von Datenverlust, dafür sind aber bisher noch nicht alle Funktionen des Samba-Servers implementiert. Man muss daher ein paar kleine Einschränkungen und Umwege bei der Konfiguration einplanen.

**Hinweis:** Selbst erstellte Kernel sind nicht digital signiert. Damit die Uefi-Firmware den Kernel laden kann, muss im Firmware-Setup die Secure-Boot-Funktion deaktiviert sein.

## 1. Vorbereitungen für den neuen Kernel

Aktuelle Kernel für Ubuntu und Linux Mint gibt es zum Download bei https://kernel.ubuntu.com/~kernel-ppa/mainline. Neuere Kernel lassen sich allerdings nur in Ubuntu etwa ab Version 21.10 installieren. Deshalb muss man die Kernel für Ubuntu 20.04 oder Linux Mint 20 selbst erstellen.

**Schritt 1:** Ubuntu-Nutzer öffnen „Anwendungen & Aktualisierungen“, setzen auf der Registerkarte „Ubuntu-Anwendungen“ ein Häkchen vor „Quelltext“, klicken auf „Schließen“ und danach auf „Neu laden“. Unter Linux Mint starten Sie „Systemverwaltung -> Anwendungspaketquellen“, aktivieren „Quellcodepaketquelle“ und klicken auf „OK“.

**Schritt 2:** Öffnen Sie ein Terminal (Strg-Alt-T) und richten Sie zuerst die Pakete zum Erzeugen des aktuellen Kernels ein:
```
sudo apt-get build-dep linux linux-image-$(uname -r)
```
Zusätzlich sind diese Pakete erforderlich:
```
sudo apt install git build-essential fakeroot libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf dwarves zstd debhelper
```
**Schritt 3:** Erstellen Sie einen Arbeitsordner in Ihrem Homeverzeichnis und wechseln Sie in diesen Ordner, beispielsweise mit diesen zwei Befehlszeilen:
```
mkdir ~/kernel
cd ~/kernel
```
Danach laden Sie den Quellcode des Ubuntu-Kernels herunter:
```
git clone --depth=1 -b cod/mainline/v5.15.6 git://git.launchpad.net/~ubuntu-kernel-test/ubuntu/+source/linux/+git/mainline-crack
```
Die Versionsnummer „v5.15.6“ passen Sie für noch neuere Kernel an. Welche Versionen verfügbar sind, erfahren Sie unter https://kernel.ubuntu.com/~kernel-ppa/mainline.

**Schritt 4:** Führen Sie die folgenden sechs Befehlszeilen aus:
```
cd mainline-crack
chmod a+x debian/rules
chmod a+x debian/scripts/*
chmod a+x debian/scripts/misc/*
LANG=C fakeroot debian/rules clean
LANG=C fakeroot debian/rules editconfigs
```
Nach der letzten Zeile bestätigen Sie die Frage „Do you want to edit config: amd64/config.flavour.generic? [Y/n]“ mit der Enter-Taste. Das Modul ntfs3 („NTFS Read-Write file system support“ und ksmbd („SMB3 server support“) sind bereits aktiviert, weshalb Sie nur auf „Exit“ gehen und mit „Yes“ bestätigen müssen. Danach erfolgt in der gleichen Weise die Konfiguration für „amd64/config.flavour.lowlatency“.

![](https://github.com/Myria-de/build-mainline-kernel/blob/main/106_01_Config.png?raw=true)

**Schritt 5:** Die Zeile
```
LANG=C fakeroot debian/rules binary-headers binary-generic binary-perarch
```
erzeugt die deb-Pakete für den neuen Kernel. Für die Installation verwenden Sie diese zwei Zeilen:
```
cd ~/kernel
sudo dpkg -i *.deb
```
Danach starten Sie Linux neu, um das System mit dem neuen Kernel zu laden.

![](https://github.com/Myria-de/build-mainline-kernel/blob/main/106_02_dpkg.png?raw=true)

## 2. NTFS-Modul verwenden
Laden Sie das NTFS-Modul im Terminal mit
```
sudo modprobe ntfs3
```
Damit der Kernel das Modul beim Systemstart automatisch lädt, führen Sie die folgende Befehlszeile aus:
```
echo ntfs3 | sudo tee -a /etc/modules
```
Verschaffen Sie sich mit 
```
sudo parted -l
```
einen Überblick über die Partitionen. Hängen Sie eine NTFS-Partition beispielsweise mit
```
sudo mount -t ntfs3 -o uid=1000,gid=1000 /dev/sdd3 /mnt
```
in das Dateisystem ein.

**Datenträger über den Dateimanager einbinden:** Über den Dateimanager werden NTFS-Partitionen weiterhin mithilfe von ntfs-3g eingebunden. Damit das auch mit dem ntfs3-Modul funktioniert, sind Änderungen in der Konfiguration nötig.

Erstellen Sie die Textdatei "75-ntfs3.rules" mit diesem Inhalt:
```
SUBSYSTEM=="block", ENV{ID_FS_TYPE}=="ntfs", ENV{ID_FS_TYPE}="ntfs3", ENV{UDISKS_FILESYSTEM_SHARED}="0"
```
Kopieren Sie die Datei (als root) in den Ordner "/etc/udev/rules.d"

**Anmerkung:** "UDISKS_FILESYSTEM_SHARED" bietet diese Optionen:

"1": Dateisystem als gemeinsam genutztes Verzeichnis mounten (/media/VolumeName)

"0": Ubuntu Standard. Dateisystem als privates Verezichnis mounten (/media/$USER/VolumeName)

Erstellen Sie die Textdatei "mount_options.conf" mit diesem Inhalt:
```
[defaults]
ntfs3_defaults=uid=$UID,gid=$GID,noacsrules,discard
ntfs3_allow=uid=$UID,gid=$GID,umask,dmask,discard,fmask,locale,norecover,ignore_case,compression,nocompression,big_writes,nls,nohidden,sys_immutable,sparse,showmeta,prealloc,noacsrules
```
Kopieren Sie die Datei (als root) in den Ordner "/etc/udisks2".

**Anmerkungen:** Durch "noacsrules" werden keine speziellen Rechte genutzt werden. Es kann daher nicht zu Zugriffsproblemen kommen, wenn ein anderer Nutzer ein Elemnet im Dateisystem erstellt hat.

"discard" für SSDs (wird bei anderen Laufwerken ignoriert): Online-Discard, da ntfs3 weiterhin (Kernel 5.16.3) kein fstrim unterstützt und deshalb nicht regelmäßig (durch cronjob, systemd-timer) getrimmt werden kann.

**Starten Sie Linux neu.**

Binden Sie einen NTFS-Datenträger über den Dateimanager ein (Ubuntu: "Andere Orte").

Im Terminal lässt sich prüfen, dass tatsächlich der NTFS3-Treiber zum Einsatz kommt. Die Ausgabe des Befehls "mount" kann dann beispielsweise so aussehen:
```
/dev/sda3 on /media/te/904C9F0E4C9EEE6A type ntfs3 (rw,nosuid,nodev,relatime,uid=1000,gid=1000,iocharset=utf8,discard,noacsrules,uhelper=udisks2)
```
Hinter "type" steht jetzt "ntfs3" statt vorher "fuseblk".

## 3. Samba-Server konfigurieren
Für das smb3-Server-Modul benötigen Sie zusätzliche Tools, die Sie erst kompilieren müssen. Dazu verwenden Sie die folgenden sechs Zeilen:
```
sudo apt install autoconf libtool pkg-config libnl-3-dev libnl-genl-3-dev libglib2.0-dev
mkdir ~/ksmbd && cd ~/ksmbd
git clone https://github.com/cifsd-team/ksmbd-tools.git
cd ksmbd-tools
./autogen.sh && ./configure
make && sudo make install
```
Falls der Samba SMB Daemon (Paket: samba) bereits installiert ist, deaktivieren Sie den Dienst mit diesen zwei Zeilen

```
sudo systemctl stop smbd
sudo systemctl disable smbd
```
Laden Sie das Modul mit 
```
sudo modprobe ksmbd
```
Für den automatischen Start bauen Sie den Modulnamen in die Datei "/etc/modules" ein. Erstellen Sie die Datei "/etc/ksmbd/smb.conf". Hier ein Beispiel (passen Sie die Bezeichnungen und Pfadangaben für Ihr System an):
```
[global]
workgroup = WORKGROUP
netbios named = MeinRechner

[data]
   path = /data
   guest ok = no,
   writable = yes

[Downloads]
   path=/home/sepp/Downloads
   guest ok = no
   writable = yes

[homes]
   comment = Home Directories
   path = /home
   browseable = yes
   read only = no
   create mask = 0700
   directory mask = 0700
   valid users = te
```
Eine Dokumetation der möglichen Optionen finden Sie unter https://github.com/cifsd-team/ksmbd-tools/blob/master/Documentation/configuration.txt.

Legen Sie das Samba-Passwort für den Benutzer fest:
```
sudo ksmbd.adduser -a [Benutzername]
```
Den Platzhalter „[Benutzername]“ ersetzen Sie durch Ihren Benutzernamen. Beenden Sie den ksmb-Daemon und starten Sie ihn neu:
```
sudo ksmbd.control -s
sudo ksmbd.mountd
```
Danach können Sie im Linux-Dateimanager über die Adressleiste (einblenden mit Strg-L) und einer Adress in der Form
```
smb://[Server]/[Freigabe]
```
auf die Freigabe zugreifen. Windows-Nutzer verwenden
```
\\[Server]\[Freigabe]
```
Der ksmbd-Daemon startet nicht automatisch. Die passende Dienstdefinition "ksmbd.service" sieht so aus:
```
[Unit]
Description=ksmbd userspace daemon
Wants=network-online.target
After=network.target network-online.target

[Service]
Type=oneshot
User=root
Group=root
RemainAfterExit=yes
ExecStartPre=-/sbin/modprobe ksmbd
ExecStart=/usr/local/sbin/ksmbd.mountd -s
ExecReload=/bin/sh -c '/usr/local/sbin/ksmbd.control -s && /usr/local/sbin/ksmbd.mountd -s'
ExecStop=/usr/local/sbin/ksmbd.control -s

[Install]
WantedBy=multi-user.target
```

Kopieren Sie die Datei "ksmbd.service" (als root) in den Ordner "/etc/systemd/system".

Aktivieren Sie den Dienst mit
```
systemctl daemon-reload
systemctl enable ksmbd.service
systemctl start ksmbd.service
```
