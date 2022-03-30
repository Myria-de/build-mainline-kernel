# build-mainline-kernel
Build mainline Linux kernel for Ubuntu and Linux Mint

## 1. Vorbereitungen für den neuen Kernel

Aktuelle Kernel für Ubuntu und Linux Mint gibt es zum Download bei https://kernel.ubuntu.com/~kernel-ppa/mainline. Neuere Kernel lassen sich allerdings nur in Ubuntu etwa ab Version 21.10 installieren. Deshalb muss man die Kernel für Ubuntu 20.04 oder Linux Mint 20 selbst erstellen.

**Schritt 1:** Ubuntu-Nutzer öffnen „Anwendungen & Aktualisierungen“, setzen auf der Registerkarte „Ubuntu-Anwendungen“ ein Häkchen vor „Quelltext“, klicken auf „Schließen“ und danach auf „Neu laden“. Unter Linux Mint starten Sie „Systemverwaltung -> Anwendungspaketquellen“, aktivieren „Quellcodepaketquelle“ und klicken auf „OK“.

**Schritt 2:** Öffnen Sie ein Terminal (Strg-Alt-T) und richten Sie zuerst die Pakete zum Erzeugen des aktuellen Kernels ein:
```
sudo apt-get build-dep linux linux-image-$(uname -r)
```
Zusätzlich sind diese Pakete erforderlich:
```
sudo apt install git build-essential fakeroot libncurses-dev gawk flex bison openssl libssl-dev dkms libelf-dev libudev-dev libpci-dev libiberty-dev autoconf dwarves zstd
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

## NTFS-Modul verwenden
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

Über den Dateimanager werden NTFS-Partitionen weiterhin mithilfe von ntfs-3g eingebunden. Damit das auch mit dem ntfs3-Modul funktioniert, sind einige Änderungen an der Konfiguration nötig.

## Samba-Server konfigurieren
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
