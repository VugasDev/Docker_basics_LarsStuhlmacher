# Docker Basics — Lars Stuhlmacher

Übungsrepository zu den Docker-Grundlagen (SWD-Unterricht). Es enthält vier
Docker-Compose-Stacks, die nacheinander gestartet und über ihre jeweilige
Weboberfläche bedient werden.

Die Compose-Dateien für **Pi-hole**, **Portainer** und **Watchtower** sowie der
Befehlsteil dieser Readme stammen aus dem Video *Docker für c’t 3003* und dem
zugehörigen Gist. Die Datei für **nginx** wurde selbst erstellt.

> Quelle: <https://gist.github.com/jamct/2e6c03f60319423bc4bc6c23fc0aa359>

## Aufbau des Repositorys

```
.
├── Readme.md
├── .gitignore
├── pihole/
│   └── pihole.yml
├── portainer/
│   └── portainer.yml
├── watchtower/
│   └── watchtower.yml
└── nginx/
    ├── nginx.yml
    └── html/index.html
```

Jeder Dienst liegt in einem eigenen Ordner mit eigener Compose-Datei. Das
entspricht der Empfehlung aus dem Video: So lässt sich mit `docker compose down`
gezielt ein einzelner Teil der Umgebung herunterfahren, ohne die übrigen
Container zu beeinflussen.

## Ports in diesem Projekt

| Dienst     | Host-Port    | Weboberfläche                     |
| ---------- | ------------ | --------------------------------- |
| nginx      | 80           | <http://localhost>                |
| Pi-hole    | 8081, 5353   | <http://localhost:8081/admin>     |
| Portainer  | 9000         | <http://localhost:9000>           |
| Watchtower | —            | keine, arbeitet im Hintergrund    |

Zwei Abweichungen von den Original-Dateien aus dem Video waren nötig, weil
Ports auf dem Testsystem (Ubuntu unter WSL2) bereits belegt waren:

* **Pi-hole DNS** liegt auf Host-Port `5353` statt `53`, weil `systemd-resolved`
  Port 53 belegt.
* **Pi-hole Web** liegt auf Host-Port `8081` statt `80`, weil Port 80 in diesem
  Projekt vom nginx-Stack verwendet wird.

## Stacks starten

Jeder Stack wird einzeln gestartet. Der Schalter `-f` gibt die Compose-Datei an,
`-d` startet die Container im Hintergrund (*detached*).

```bash
docker compose -f pihole/pihole.yml up -d
docker compose -f portainer/portainer.yml up -d
docker compose -f watchtower/watchtower.yml up -d
docker compose -f nginx/nginx.yml up -d
```

Laufende Container beobachten:

```bash
docker ps
```

Das Pi-hole-Passwort steht bewusst nicht im Repository. Es wird nach dem ersten
Start gesetzt:

```bash
docker exec -it pihole pihole setpassword <passwort>
```

Alles wieder herunterfahren:

```bash
docker compose -f nginx/nginx.yml down
docker compose -f watchtower/watchtower.yml down
docker compose -f portainer/portainer.yml down
docker compose -f pihole/pihole.yml down
```

---

# Hier alle im Video erwähnten Befehle:

## Docker installieren:

```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```

Falls kein curl installiert: `apt install curl`

Wenn man nicht mit dem Nutzer Root arbeitet, sollte man den aktuellen Benutzer berechtigen:

```
sudo usermod -aG docker $USER
```

## Mit Docker arbeiten

* Laufende Container auflisten: `docker ps`
* Alle Container auflisten (auch gestoppte): `docker ps -a`
* Einen Container anhalten: `docker stop <Containername>` (den Namen findet man mit `docker ps` heraus)
* Einen gestoppten Container endgültig löschen: `docker rm <Containername>`

## Einen simplen Webserver starten

Der Container aus dem Image nginx fährt mit folgendem Befehl hoch:

```
docker run -p 80:80 nginx
```
Die eigene IP-Adresse erhält man mit `ip a`

## Arbeiten mit Docker-Compose

Legt euch am besten einen eigenen Ordner für das Docker-Projekt an, um Ordnung zu halten. Die Datei docker-compose.yml enthält die Definition der Container.

Bearbeitet wird die Datei mit:

```
nano docker-compose.yml
```

Die Inhalte findet ihr unten in diesem GitHub-Gist. Den Texteditor Nano beendet man mit: `Strg+X`, dann `Y`

Die Compose-Zusammenstellung hochfahren:

```
docker compose up -d
```

Will man die Container updaten, lädt man die neuen Images mit

```
docker compose pull
```

## Eine oder mehrere Docker-Compose-Dateien?

Das ist definitiv Geschmackssache und hängt von der Umgebung ab. Wenn man mehr als ein Projekt (zum Beispiel einen Blog und ein Pihole) auf einem Server betreibt, sollte man für jedes einen Ordner anlegen und darin eine Docker-Compose-Datei ablegen. Die nützlichen Helfer wie Portainer und Watchtower kommen zusammen in eine weitere Datei. Dann kann man mit `docker compose down` gezielt Teile der Umgebung herunterfahren.
