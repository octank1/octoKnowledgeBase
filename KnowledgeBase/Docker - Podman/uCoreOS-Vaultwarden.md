# Vaultwarden auf uCoreOS

## Installation als Container

Die Installation des Containers direkt in Podman ist denkbar einfach. Mit dem folgenden Befehl wird der Container erzeugt und gestartet. Nach wenigen Sekunden ist er einsatzbereit.

Hinweise hierzu und die Quelle, um den Container selber zu bauen, findet man hier:

[dani-garcia/vaultwarden](https://github.com/dani-garcia/vaultwarden/blob/main/README.md)

### HTTPS

Hinweis: hier wurde die **nicht empfohlene** SSL-Version über Rocket aktiviert, da ein https-Zugriff über 'localhost' auf der Sophos ohne GUI nicht funktioniert.
Dazu bitte die Doku lesen, wie sich dies sicherer per NGINX umsetzen lässt.

Für HTTPS benötigt man eine Domain, für die man mit **Let's Encrypt** ein kostenloses Zertifikat anlegen kann. Dazu bitte die einschlägige Dokumentation lesen.
Dies ist insbesondere notwendig, wenn man mit der Handy-App auf seinen Vault zugreifen möchte, da man dort keine eigenen Zertifikate hinterlegen kann, so dass self-signed nicht funktioniert.

[Enabling HTTPS](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS)

### Besonderheiten für uCoreOS

1. Bei Containern innerhalb des User-Kontextes auf uCoreOS immer darauf achten, dass der Owner der Volume-Verzeichnisse der eigene User ist (also nicht per sudo anlegen oder ändern, falls man Verzeichnisse von einem anderen User kopiert hat)
2. Bei der Angabe der Volume-Parameter unbedingt am Ende `:Z,U` hinzufügen, damit die Verzeichnisse und Dateien die ACL `system_container_t` zugewiesen bekommen, da podman ansonsten keinen Zugriff auf die Verzeichnisse und Dateien hat.

```sh
podman pull vaultwarden/server:latest
podman run --detach --name vaultwarden \
  --env DOMAIN="https://deine-domain:port" \
  --env ROCKET_TLS='{certs="/ssl/certificate.crt",key="/ssl/private.key"}' \
  --volume /var/home/vaultwarden/vwdata/certificates/:/ssl/:Z,U \
  --volume /var/home/vaultwarden/vwdata/:/data/:Z,U \
  --restart unless-stopped \
  --publish 443:80 \
  vaultwarden/server:latest
```

## Installation als Service

Um in uCoreOS einen Container als Service zu installieren, was notwendig ist, damit er nach einem Neustart automatisch gestartet wird, sind folgende Schritt notwendig:

### User anlegen und berechtigen

```bash
# User anlegen
sudo useradd vaultwarden
# User mit sudo Rechten versehen (können später entzogen werden)
sudo usermod -aG sudo vaultwarden
# Passwort vergeben
sudo passwd vaultwarden  
```
Dann unbedingt als dieser User anmelden, damit eine Usersession gestartet wird. Ein Wechsel der CLI mit `su vaultwarden` reicht nicht aus.

### Verzeichnisstruktur anlegen

Um die Containerdateien anlegen zu können, damit diese beim Systemstart ausgeführt werden können, müssen diese im Verzeichnis `~/.config/containers/systemd` liegen. 
Dieses Verzeichnis im User-Kontext anlegen (.config gibt es schon)

### Containerdatei anlegen

Nun muss man eine Container-Datei (Quadlet) anlegen, in der die Containerdetails beschrieben sind. Inhaltlich entspricht sie einem Compose-File oder dem Aufruf oben, allerdings in einer anderen Syntax.

Nachfolgend eine Beispieldatei, die als Kopiervorlage verwendet werden kann.
Diese ist unter dem Namen `vaultwarden.container` im systemd-Verzeichnis anzulegen.

```
[Unit]
Description=Vaultwarden Container
Wants=network-online.target
After=network-online.target

[Container]
Image=docker.io/vaultwarden/server:latest
ContainerName=vaultwarden
HostName=vaultwarden
#AutoUpdate=registry
Timezone=Europe/Berlin

Environment=DOMAIN="https://deine-domain:port"
Environment=ROCKET_TLS='{certs="/ssl/certificate.crt",key="/ssl/private.key"}'

Volume=/var/home/vaultwarden/vwdata/certificates/:/ssl/:Z,U
Volume=/var/home/vaultwarden/vwdata/:/data/:Z,U

PublishPort=443:80/tcp

UserNS=auto
NoNewPrivileges=true

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target

```

### Starten des Containers als Service

Wenn die Datei fehlerfrei erstellt wurde, kann man sie als Service starten. Dazu führt man zwei Befehle aus (`daemon-reload`)

```bash
# Geänderte systemd Informationen laden (nach jeder Änderung an der Datei notwendig)
systemctl --user daemon-reload
# Starten des Containers als Service
systemctl --user start vaultwarden.service
```

### Automatisches Starten nach Reboot

Damit die Container im Userkontext nach einem Neustart ausgeführt werden, muss über den folgenden Befehl eine User-Session angelegt werden, auch, wenn sich der User natürlich nicht am Server anmeldet.

Der Befehl sorgt dafür, dass beim Systemstart eine Session unter dem Usernamen geöffnet wird, wodurch die Container-Services ausgeführt werden.

```bash
# Den User zum lingering hinzufügen
loginctl enable-linger vaultwarden 
# Liste der User anzeigen, für die Sessions automatisch gestartet werden
loginctl list-users
```

