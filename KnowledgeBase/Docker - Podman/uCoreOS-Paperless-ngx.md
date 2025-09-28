# Paperless-ngx auf uCoreOS

## Installation als Podman-Container

Die Installation unter Docker ist einfach und funktioniert problemlos anhand der Dokumentation unter

[docs.paperless-ngx.com](https://docs.paperless-ngx.com/setup/)

In diesem Artikel werde ich nur auf die **Installation in podman** eingehen. Dabei gibt es zwei Versionen. Die Installation direkt in podman, die bei uCoreOS jedoch nach jedem Reboot verschwindet. Um eine Installation reboot-sicher zu machen, müssen die Container als services (entweder rootless oder als root) eingerichtet werden.

Eine Installation als root ist nur per ignition bei der Installalation des Betriebssystems möglich. Daher wird hier **nur die rootless-Variante** näher beleuchtet.

**Hinweis**: die Installation mit sqlite funktioniert für den ersten Test gut und einfach. Aber wenn man vorhat, das System zu nutzen und 2000+ Dokumente dort abzulegen, sollte man direkt eine Datenbankversion in Betracht ziehen. Eine Datenmigration ist möglich, aber schwierig.
Die Problematik mit sqlite: man erhält beim Hochladen und Verarbeiten von Dokumenten ständig lock-Fehler, weil die Datenbank-Dateien gelockt werden und konkurrierende Zugriffe dann nicht möglich sind. Eine Datenbank, wie z.B. mariadb hat solche Probleme nicht.

## Installation per Installationsscript

Die Installation als Podman-Container über `podman-compose` ist ziemlich einfach und lässt sich über das Install-Script von Paperless bewerkstelligen. Dazu muss man allerdings im Script Änderungen vornehmen, damit `podman-compose` und nicht docker verwendet wird.

Aber das Script erzeugt alle relevanten Dateien und fragt die benötigten Parameter ab.

<mark>Diese Variante eigent sich gut, um schnell und einfach die Performance des Servers zu testen</mark>

## Installation über Compose

Man kann sich von Git-Hub das passende Compose-File herunterladen und über `docker-compose` alle Container automatisch anlegen lassen.

Quelle: [Git-Hub paperless-ngx](https://github.com/paperless-ngx/paperless-ngx/tree/dev/docker/compose)

Die Problematik liegt in der Netzwerk-Konfiguration.

In einem uCore-OS ist Podman mit einem default-netzwerk vorinstalliert. Dieses bietet jedoch keine DNS-Funktionalität. D.h. die Referenzen auf die anderen Container funktionieren nicht. Das kann man umgehen, indem man den Containern feste IP-Adressen gibt und diese in der Config referenziert.

Eine andere Möglichkeit ist, ein eigenes Netzwerk mit aktiviertem DNS anzulegen, damit die Container untereinander kommunizieren können. Natürlich muss das Netzwerk außerhalb des Podman Adressbereichs liegen.

Das lässt sich mit `podman network inspect podman` ermitteln. Wenn man ein eigenes Netzwerk aufbaut, muss dieses in einem anderen Adressbereich als das Podman-Netzwerk liegen (beachte: Das Standard-Netzwerk umfasst mehrere subnetz-Bereiche, also z.B. 10.88.0.0/16, also 10.88.0.1 bis 10.88.225.225). Ein eigenes Netzwerk kann also erst bei 10.89.0.1 beginnen.

<mark>Beachte: Diese Installationsart ist nach einem reboot wieder weg. Aber zum Testen der Einstellung durchaus geeignet</mark>

## Installation als User-Service

Hierzu müssen sog. Quadlet-Dateien erzeugt werden, die von `systemctl` als Dienste installiert und ausgeführt werden. Die Erstellung ist zwar einfach, aber Fehler innerhalb der Datei werden beim Starten mit unscheinbaren Fehlermeldungen quittiert. Hier ist ein gutes Auge gefragt, Fehler zu erkennen und zu beheben.

### User anlegen und berechtigen

```bash
# User anlegen
sudo useradd paperless
# User mit sudo Rechten versehen (können später entzogen werden)
sudo usermod -aG sudo paperless
# Passwort vergeben
sudo passwd paperless  
```
Dann unbedingt als dieser User anmelden, damit eine Usersession gestartet wird. Ein Wechsel der CLI mit `su paperless` reicht nicht aus.

### Vorbereitung

Um die Containerdateien anlegen zu können, damit diese beim Systemstart ausgeführt werden können, müssen diese im Verzeichnis `~/.config/containers/systemd` liegen. 
Dieses Verzeichnis im User-Kontext anlegen (.config gibt es schon)

Man sollte auch einen guten Editor installieren, mit dem man gut zurecht kommt. Standardmäßig sind `vi` und `nano` auf einem uCore enthalten. Ich bevorzuge inzwischen `micro`, da mich `nano` mit seinen Tastenkombinationen langsam um den Verstand gebracht hat.

### Containerdateien anlegen

Nun muss man die Container-Dateien (Quadlet) für die einzelnen Container anlegen, in der die Containerdetails beschrieben sind. Inhaltlich entspricht sie einem Compose-File oder dem Aufruf oben, allerdings in einer anderen Syntax.

Nachfolgend die Inhalte als Beispieldateien, die als Kopiervorlage verwendet werden kann.
Diese ist unter den im Kommentar genannten Namen als `<name>.container` im systemd-Verzeichnis anzulegen.

```
# paperless-broker.container
[Unit]
Description=Paperless Broker
Wants=network-online.target
After=network-online.target paperless-mariadb.container

[Container]
Image=docker.io/library/redis:8
ContainerName=paperless-broker
HostName=broker
#AutoUpdate=registry
Timezone=Europe/Berlin
Environment=HOME=/root
Environment=HOSTNAME=broker
Environment=REDIS_DOWNLOAD_SHA=e2c1cb9dd4180a35b943b85dfc7dcdd42566cdbceca37d0d0b14c21731582d3e
Environment=REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-8.2.1.tar.gz

Network=paperless
NetworkAlias=broker
IP=10.89.2.2

Volume=paperless_redisdata:/data:Z,U

UserNS=auto
NoNewPrivileges=true

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target paperless-webserver.container


# paperless-gotenberg.container

[Unit]
Description=Paperless Gotenberg
Wants=network-online.target
After=network-online.target paperless-broker.container

[Container]
Image=docker.io/gotenberg/gotenberg:8.22
ContainerName=paperless-gotenberg
HostName=gotenberg
#AutoUpdate=registry
Timezone=Europe/Berlin
Exec=/bin/bash -c "gotenberg --chromium-disable-javascript=true --chromium-allow-list=file:///tmp/.*"

Network=paperless
NetworkAlias=gotenberg
IP=10.89.2.5

UserNS=auto
NoNewPrivileges=true

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target paperless-webserver.container


# paperless-mariadb.container

[Unit]
Description=Paperless MariaDB
Wants=network-online.target
After=network-online.target

[Container]
Image=docker.io/library/mariadb:11.8
#Image=github.com/MariaDB/mariadb-docker
ContainerName=paperless-db
HostName=mariadb
#AutoUpdate=registry
Timezone=Europe/Berlin
Environment=MARIADB_HOST=mariadb
Environment=MARIADB_DATABASE=paperless
Environment=MARIADB_USER=paperless
Environment=MARIADB_PASSWORD=paperless
Environment=MARIADB_ROOT_PASSWORD=paperless
Environment=LANG=C.UTF-8
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

Network=paperless
NetworkAlias=mariadb
IP=10.89.2.3
PublishPort=3306:3306/tcp

Volume=/var/home/paperless/paperless-data/db:/var/lib/mysql:Z,U
UserNS=auto
NoNewPrivileges=true

[Service]
Restart=always
TimeoutStartSec=900
[Install]
WantedBy=multi-user.target default.target paperless-broker.container paperless-webserver.container paperless-tika.container     


# paperless-tika.container

[Unit]
Description=Paperless Tika
Wants=network-online.target
After=network-online.target paperless-gotenberg.container

[Container]
Image=docker.io/apache/tika:latest
ContainerName=paperless-tika
HostName=tika
#AutoUpdate=registry
Timezone=Europe/Berlin

Network=paperless
NetworkAlias=tika
IP=10.89.2.4
UserNS=auto
NoNewPrivileges=true
[Service]
Restart=always
TimeoutStartSec=900
[Install]                                                                                                            
WantedBy=multi-user.target default.target paperless-webserver.container


# paperless-webserver.container

[Unit]
Description=Paperless Webserver
Wants=network-online.target
Requires=paperless-broker.service paperless-gotenberg.service paperless-mariadb.service
#Requires=paperless-tika.service
After=network-online.target paperless-tika.service

[Container]
Image=docker.io/paperlessngx/paperless-ngx:latest
ContainerName=paperless-webserver
HostName=webserver
Timezone=Europe/Berlin
Environment=HOSTNAME=webserver
# Diese Einstellungen sind für SMB wichtig
User=1002
Group=1002
GroupAdd=keep-groups
Environment=LANG=C.UTF-8
Environment=PAPERLESS_DBENGINE=mariadb
Environment=PAPERLESS_DBHOST=10.89.2.3
Environment=PAPERLESS_DBPASS=paperless
Environment=PAPERLESS_DBPORT=3306
Environment=PAPERLESS_DBUSER=paperless
Environment=PAPERLESS_OCR_LANGUAGE=deu
Environment=PAPERLESS_REDIS=redis://10.89.2.2:6379
Environment=PAPERLESS_SECRET_KEY=F8,hx^RZ%$WJR+CxIQjcAr7/(w{_hsJ8#8b2H]e]E>?7<tWP]R9on6hxF|l~szql
Environment=PAPERLESS_TIKA_ENABLED=1
Environment=PAPERLESS_TIKA_GOTENBERG_ENDPOINT=http://10.89.2.5:3000
Environment=PAPERLESS_TIKA_ENDPOINT=http://10.89.2.4:9998
Environment=PAPERLESS_TIME_ZONE=Europe/Berlin
Environment=S6_BEHAVIOUR_IF_STAGE2_FAILS=0                                                                           
Environment=S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0                                                                       
Environment=S6_VERBOSITY=1                                                                                           
Environment=UV_CACHE_DIR=/cache/uv/
Environment=UV_LINK_MODE=copy
Environment=UV_TOOL_BIN_DIR=/usr/local/bin
Network=paperless
NetworkAlias=webserver
IP=10.89.2.10
PublishPort=8090:8000/tcp

Volume=/var/home/paperless/data/data:/usr/src/paperless/data:Z,U
Volume=/var/home/paperless/data/media:/usr/src/paperless/media:Z,U
Volume=/var/home/paperless/data/export:/usr/src/paperless/export:Z,U
Volume=/var/home/paperless/data/consume:/usr/src/paperless/consume:Z,U
# Wenn man eine UID/GID angibt, funktioniert das S6-Overlay nicht mehr
# Daher muss man hier ein virtuelles Volume zuweisen, damit S6 funktioniert
Volume=run.volume:/run:Z,U


#UserNS=auto  -- dies ist falsch für SMB
# Diese Einstellung wird benötigt, damit der SMB Folder Schreibrechte behält
UserNS=keep-id:uid=1002,gid=1002
NoNewPrivileges=true
#ReadOnly=true

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target


# paperless.network

[Network]
Subnet=10.89.2.0/24
Gateway=10.89.2.1

```

### Starten des Containers als Service

Wenn die Datei fehlerfrei erstellt wurde, kann man sie als Service starten. Dazu führt man zwei Befehle aus (`daemon-reload`)

```bash
# Geänderte systemd Informationen laden (nach jeder Änderung an der Datei notwendig)
systemctl --user daemon-reload
# Starten des Containers als Service
systemctl --user start paperless-mariadb.service
systemctl --user start paperless-broker.service
systemctl --user start paperless-gotenberg.service
systemctl --user start paperless-tika.service
systemctl --user start paperless-webserver.service
```

### Automatisches Starten nach Reboot

Damit die Container im Userkontext nach einem Neustart ausgeführt werden, muss über den folgenden Befehl eine User-Session angelegt werden, auch, wenn sich der User natürlich nicht am Server anmeldet.

Der Befehl sorgt dafür, dass beim Systemstart eine Session unter dem Usernamen geöffnet wird, wodurch die Container-Services ausgeführt werden.

```bash
# Den User zum lingering hinzufügen
loginctl enable-linger paperless 
# Liste der User anzeigen, für die Sessions automatisch gestartet werden
loginctl list-users
```

### Fehleranalyse

Die wichtigsten Befehle zur Fehleranalyse und zum Starten sind:

`systemctl --user daemon-reload` # Muss nach jeder Änderung an der Config vorgenommen werden

`systemctl --user start name.service` # Der name.service entspricht dem Dateinamen name.container

`journalctl --user -xe` # Fehleranalyse der letzten Einträge im Journal

`journalctl --user -xeu name.service` # Hiermit werden nur die Fehler eines Containers/Services angezeigt

`podman ps` # Listet die aktuell laufenden Container auf

`podman network ls` # Listet die podman Netzwerke auf, die Parameter `rm` oder `prune` sind auch sehr wichtig

### Zugriff auf den consume-Folder per SMB

Um Zugriff auf das SMB-Share zu gewähren, muss unter SELinux ein Kontext hinzugefügt werden.

```sh
sudo semanage fcontext --add --type "samba_share_t" /var/home/paperless/data
sudo restorecon -R /var/home/paperless/data
```

Und in der SMB-Konfiguration `/etc/samba/smb.conf` muss der Share deklariert sein.

```sh
[paperless_share]
        comment = Paperless Datenverzeichnis
        path = /var/home/paperless/data
        writeable = yes
        browseable = yes
        public = yes
        create mask = 0644
        directory mask = 0755
        write list = user
```

Dann muss der Paperless-User für SMB berechtigt und der Service aktiviert und gestartet werden.

```sh
sudo smbpasswd -a paperless
sudo systemctl enable --now smb.service
sudo systemctl status smb.service
```

### Disclaimer

Auch, wenn ich die Informationen sorffältig zusammengetragen und bei mir verwende, kann ich keine Garantie übernehmen, dass es bei Dir 1:1 auch so funktioniert. Dazu sind die Systeme alle zu unterschiedlich und viele Dinge hängen von der individuellen Konfiguration ab.
Aber vielleicht findest Du trotzdem Hinweise auf weitere Recherchen oder bekommst Ideen, was man manchen muss, damit es letztendlich funktioniert.

Viel Vergnügen
Oliver
