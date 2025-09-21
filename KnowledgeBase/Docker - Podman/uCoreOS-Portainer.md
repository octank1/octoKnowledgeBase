# Portainer auf uCoreOS 

## Vorbemerkung

Die Installation von Portainer geht über den normalen podman-Aufruf problemlos. Jedoch müsste man für jeden Userkontext einen eigenen Portainer-Container anlegen, um auf die Container des jeweiligen Users zugreifen zu können.

Eine interessante Alternative könnte sein, einen Container als root anzulegen, der Zugriff auf alle anderen User-Container hat. So hat man nur eine Oberfläche, um die Container schnell und einfach zu verwalten.

Dies ist jedoch nicht ganz so einfach.

## Installation als Container

Zum ersten Testen kann man einen normalen Container anlegen und schauen, ob das funktioniert. Das geht mit folgendem Befehl:

```bash
# Den podman-Socket aktivieren
sudo systemctl --user enable --now podman.socket
# Ein Volume für die Daten anlegen
sudo podman volume create portainer_data
# Den Container starten
sudo podman run -d \
-p 8001:8000 \
-p 9443:9443 \
--name portainer \
--restart=always --privileged \
-v /run/podman/podman.sock:/var/run/docker.sock \
-v portainer_data:/data \
docker.io/portainer/portainer-ce:sts
```

### Installation als Service

Die Installation als Container-Service (wie z.B. bei den paperless- oder vaultwarden-Containern) per Quadlet ist leider nicht möglich, da `--privileged` damit nicht gesetzt werden kann.

_Als Referenz die Quadlet-Datei portainer.container_

```
# FUNKTIONIERT NICHT, DA KEIN PRIVILEGED MÖLGICH
[Unit]
Description=Portainer
Wants=network-online.target
After=network-online.target

[Container]
Image=docker.io/portainer/portainer-ce:sts
ContainerName=portainer
#Privileged=true
AddCapability=ALL
AutoUpdate=registry
Timezone=Europe/Berlin
Environment=TZ=Europe/Berlin

PublishPort=8000:8000/tcp
PublishPort=8443:9443/tcp

# Root-Containers
Volume=/run/podman/podman.sock:/var/run/docker.sock:Z,U
# Core-Containers
# Datencontainer
Volume=portainer:/data:Z,U

UserNS=auto
NoNewPrivileges=false
#ReadOnly=true

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target
```

**Nun hätte man zwei Möglichkeiten:**
1. Entweder legt man einen normalen Service an, der den Container erstellt und startet (das hat bei mir nicht funktioniert, weil er den Container immer schon kennt, obwohl er gelöscht wurde)
2. Den Container einmal selbst mit obigem Befehl anlegen und dann nur noch per Service starten

_Variante 1:_

Service in `/etc/systemd/system/portainer.service`

```
[Unit]
Description=Portainer-Container
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=podman run --rm --privileged \
    -v /run/podman/podman.sock:/var/run/docker.sock \
    -v /run/user/1000/podman/podman.sock:/var/run/1000/docker.sock \
    -v /run/user/1002/podman/podman.sock:/var/run/1002/docker.sock \
    -v /run/user/1005/podman/podman.sock:/var/run/1005/docker.sock \
    -v portainer:/data
    -p 8000:8000 \
    -p 8443:9443 \
    --name portainer-root \
    docker.io/portainer/portainer-ce:latest

Restart=always

[Install]
WantedBy=multi-user.target
```

_Variante 2:_

Service in `/etc/systemd/system/portainer.service`

```
[Unit]
Description=Portainer
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/podman start portainer
ExecStop=/usr/bin/podman stop portainer
Restart=always

[Install]
WantedBy=multi-user.target
```

### Zugriff auf die Container anderer User erhalten

Der Zugriff auf die Container anderer User kann über **Environments** innerhalb von Portainer gelöst werden. Dazu müssen diese User aber den podman.socket starten.
D.h. innerhalb jedes User-Kontextes muss auch der Befehl `systemctl --user enable --now podman.socket` ausgeführt werden, damit der Socket gestartet wird.
Dieser ist dann unter `/run/user/UID/podman` zu finden.

Dann kann man über weitere Volumes diese Sockets im Container bekannt machen `-v /run/user/UID/podman/podman.sock:/var/run/UID/docker.sock:Z,U`. Der hintere Teil der Anweisung (`/var/run/UID/docker.sock`, also der Pfad innerhalb des Containers) kann dann als Socket für das Environment verwendet werden.

## Sonstige Hinweise

Wenn man versucht, auf die Webseite zuzugreifen und den Fehler "Not found" erhält, ist der Port vermutlich nicht in der Firewall freigeschaltet, insbesondere, wenn man einen Nicht-Standard-Port verwendet.
Dies behebt man mit den folgenden zwei Befehlen:
```bash
sudo firewall-cmd --permanent --add-port 9443/tcp
sudo firewall-cmd --reload
```






