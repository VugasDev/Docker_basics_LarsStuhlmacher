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

# Die vier Serveranwendungen

Jede der vier Anwendungen erfüllt eine andere Rolle. Zwei davon sind
*Nutzanwendungen*, die einen Dienst nach außen anbieten (Pi-hole, nginx), zwei
sind *Helfer*, die den Docker-Betrieb selbst unterstützen (Portainer,
Watchtower).

| Anwendung      | Rolle                        | Weboberfläche | Braucht den Docker-Socket |
| -------------- | ---------------------------- | ------------- | ------------------------- |
| **Pi-hole**    | DNS-Server mit Werbefilter   | ja            | nein                      |
| **Portainer**  | Verwaltungsoberfläche        | ja            | **ja**                    |
| **Watchtower** | Automatische Image-Updates   | nein          | **ja**                    |
| **nginx**      | Webserver                    | ja (Inhalt)   | nein                      |

---

## Pi-hole — DNS-Server mit Werbefilter

**Pi-hole** ist ein DNS-Server, der Werbung und Tracker netzwerkweit blockiert.
Statt Werbung erst im Browser auszublenden, setzt Pi-hole eine Ebene tiefer an:
Es beantwortet DNS-Anfragen selbst.

### Funktionsprinzip

Jedes Gerät muss vor dem Aufruf einer Website deren Namen in eine IP-Adresse
übersetzen lassen. Genau an dieser Stelle greift Pi-hole ein:

1. Ein Gerät fragt: *Welche IP hat `doubleclick.net`?*
2. Pi-hole prüft den Namen gegen seine **Blocklisten**.
3. Steht der Name auf der Liste, antwortet Pi-hole mit `0.0.0.0` — die
   Verbindung läuft ins Leere, die Werbung wird nie geladen.
4. Steht er nicht darauf, reicht Pi-hole die Frage an einen **Upstream-DNS**
   weiter (hier `8.8.8.8` / `8.8.4.4`) und gibt die echte Antwort zurück.

Der Vorteil gegenüber einem Browser-Adblocker: Es wirkt für **alle** Geräte im
Netz, auch für Smart-TVs und Handys, auf denen sich keine Erweiterung
installieren lässt.

### Konfiguration in diesem Projekt

```yaml
ports:
  - "5353:53/tcp"    # DNS
  - "5353:53/udp"
  - "8081:80/tcp"    # Weboberfläche
cap_add:
  - NET_ADMIN        # Rechte für Netzwerkoperationen
```

Die Fähigkeit `NET_ADMIN` ist nötig, weil Pi-hole Netzwerkdienste bereitstellt.
Die beiden Volumes `./etc-pihole` und `./etc-dnsmasq.d` sichern Konfiguration,
Blocklisten und Statistiken über einen Neustart des Containers hinaus — ohne
sie wäre nach jedem `docker compose down` alles zurückgesetzt.

### Zugang

Weboberfläche: <http://localhost:8081/admin>

Das Passwort steht bewusst **nicht** im Repository. Es wird nach dem ersten
Start gesetzt:

```bash
docker exec -it pihole pihole setpassword <passwort>
```

> **Beobachtung beim Testlauf:** Das Blocken funktioniert nachweislich —
> `doubleclick.net` wurde mit `0.0.0.0` beantwortet. Die Weiterleitung an den
> Upstream schlug dagegen fehl. Ursache ist nicht Pi-hole, sondern die
> WSL2-Umgebung: Ausgehende DNS-Anfragen auf UDP-Port 53 werden dort auch
> direkt vom Host aus blockiert. Auf einem Raspberry Pi oder einem normalen
> Linux-Server tritt das nicht auf.

---

## Portainer — Verwaltungsoberfläche für Docker

**Portainer** ist eine grafische Oberfläche für Docker. Alles, was sonst über
`docker ps`, `docker logs` oder `docker stop` auf der Kommandozeile passiert,
lässt sich hier im Browser erledigen: Container starten und stoppen, Logs
mitlesen, eine Shell im Container öffnen, Images, Volumes und Netzwerke
verwalten.

### Funktionsprinzip

Der entscheidende Punkt steht in der Compose-Datei:

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

`/var/run/docker.sock` ist der **Docker-Socket** — die Schnittstelle, über die
der Docker-Daemon Befehle entgegennimmt. Auch das Kommando `docker ps` spricht
im Hintergrund mit genau diesem Socket. Indem der Socket in den Container
hineingereicht wird, kann Portainer alle anderen Container sehen und steuern.

> ⚠️ **Sicherheitshinweis:** Der Docker-Socket ist gleichbedeutend mit
> Root-Rechten auf dem Host. Wer Zugriff auf ihn hat, kann beliebige Container
> mit beliebigen Rechten starten. Solche Container gehören deshalb niemals
> ungeschützt ins Internet.

Das Volume `./portainer_data` speichert Benutzerkonten und Einstellungen.
Ohne dieses Volume müsste nach jedem Neustart ein neuer Administrator angelegt
werden.

### Zugang

Weboberfläche: <http://localhost:9000>

Beim allerersten Aufruf legt man den Administrator an. Aktuelle
Portainer-Versionen verlangen dabei ein **Setup-Token**, das beim Start in die
Container-Logs geschrieben wird:

```bash
docker logs portainer 2>&1 | grep setup_token
```

Danach im Setup-Bildschirm Token, Benutzername und ein Passwort mit mindestens
12 Zeichen eingeben. Anschließend die lokale Umgebung *(Get Started → local)*
auswählen — dort erscheinen alle laufenden Container.

---

## Watchtower — automatische Container-Updates

**Watchtower** hält Container aktuell. Es besitzt keine Weboberfläche und
keinen offenen Port, sondern arbeitet still im Hintergrund.

### Funktionsprinzip

In regelmäßigen Abständen (Standard: alle 24 Stunden) führt Watchtower für jeden
überwachten Container folgende Schritte aus:

1. Nachsehen, ob im Registry ein **neueres Image** unter demselben Tag liegt.
2. Falls ja: das neue Image herunterladen.
3. Den alten Container stoppen und entfernen.
4. Einen neuen Container mit **identischer Konfiguration** starten — gleiche
   Ports, gleiche Volumes, gleiche Umgebungsvariablen.

Weil die Volumes dabei erhalten bleiben, gehen keine Daten verloren. Wie
Portainer benötigt Watchtower dafür Zugriff auf den Docker-Socket.

### Wichtig: Reichweite einschränken

Ohne weitere Angabe fasst Watchtower **jeden** Container auf dem Host an — auch
solche aus völlig fremden Projekten. In dieser Übung hat ein unbedachter
Testlauf prompt eine projektfremde Datenbank neu gestartet.

Deshalb wurde die Reichweite hier bewusst begrenzt:

```yaml
command: --label-enable
```

Mit dieser Option beachtet Watchtower ausschließlich Container, die das Label
`com.centurylinklabs.watchtower.enable=true` tragen. Dieses Label ist in
`pihole.yml`, `portainer.yml` und `nginx.yml` gesetzt — alle übrigen Container
auf dem Rechner bleiben unangetastet.

> ⚠️ **Bekannte Einschränkung in dieser Umgebung:** Das Image
> `containrrr/watchtower` aus dem Video stammt vom **11.11.2023** und spricht
> die Docker-API-Version **1.25**. Die hier eingesetzte Docker Engine 29.1.3
> verlangt mindestens **API 1.44**. Der Container startet deshalb und beendet
> sich sofort wieder mit Exit-Code 1:
>
> ```
> client version 1.25 is too old.
> Minimum supported API version is 1.44
> ```
>
> Das Projekt `containrrr/watchtower` wird seit 2023 nicht mehr gepflegt. Die
> Compose-Datei bleibt hier aufgabengemäß beim Original aus dem Gist. Für den
> produktiven Einsatz weicht man auf einen aktiven Fork aus, etwa
> `nickfedor/watchtower`; ein Testlauf damit funktionierte einwandfrei
> (*Watchtower 1.22.1 using Docker API v1.52*).
>
> Das ist übrigens genau das Problem, das Watchtower selbst lösen soll:
> ein Image, das zu lange nicht aktualisiert wurde.

---

## nginx — Webserver

**nginx** (gesprochen *„engine x“*) ist ein Webserver: Er nimmt HTTP-Anfragen
entgegen und liefert Dateien aus. Er gilt als besonders ressourcenschonend und
gehört zu den meistgenutzten Webservern im Internet. Neben dem Ausliefern
statischer Seiten wird er häufig als **Reverse Proxy** eingesetzt, der Anfragen
an dahinterliegende Anwendungen verteilt.

### Selbst erstellte Compose-Datei

Diese Datei stammt als einzige nicht aus dem Video. Sie ist das
Compose-Gegenstück zum dortigen Befehl:

```bash
docker run -p 80:80 nginx
```

Gegenüber diesem Einzeiler bietet die Compose-Fassung vier Vorteile:

| Aspekt        | `docker run`              | `nginx.yml`                          |
| ------------- | ------------------------- | ------------------------------------ |
| Wiederholbar  | Befehl muss man sich merken | Konfiguration steht in der Datei    |
| Containername | zufällig vergeben         | fest `nginx`                         |
| Nach Neustart | bleibt aus                | `restart: unless-stopped`            |
| Inhalt        | nginx-Standardseite       | eigene Seite aus `./html`            |

### Der Bind-Mount

```yaml
volumes:
  - ./html:/usr/share/nginx/html:ro
```

Der Ordner `./html` des Hosts wird an die Stelle im Container gehängt, an der
nginx seine Webinhalte sucht. Das Kürzel `:ro` steht für *read-only*: Der
Container darf die Dateien lesen, aber nicht verändern.

Praktisch daran: Änderungen an `html/index.html` sind sofort im Browser
sichtbar, ohne dass ein neues Image gebaut oder der Container neu gestartet
werden muss.

### Zugang

Weboberfläche: <http://localhost>

Ein Login gibt es hier nicht — ein Webserver liefert Inhalte öffentlich aus.
Angezeigt wird die eigene Startseite aus `html/index.html`, die auf die
Oberflächen der übrigen Dienste verlinkt.

---

# Beobachtungen aus dem Praxisteil

Die vier Stacks wurden nacheinander gestartet und nach jedem Schritt mit
`docker ps` kontrolliert. Endstand:

```
NAMES        STATUS                     PORTS
nginx        Up 28 seconds              0.0.0.0:80->80/tcp
watchtower   Exited (1) 28 seconds ago
portainer    Up 30 seconds              8000/tcp, 9443/tcp, 0.0.0.0:9000->9000/tcp
pihole       Up 30 seconds (healthy)    0.0.0.0:5353->53/tcp, 0.0.0.0:5353->53/udp, 0.0.0.0:8081->80/tcp
```

Ergebnis der Anmeldung an den Weboberflächen:

| Dienst     | Adresse                          | Ergebnis                                  |
| ---------- | -------------------------------- | ----------------------------------------- |
| nginx      | <http://localhost>               | ✅ HTTP 200, eigene Startseite             |
| Pi-hole    | <http://localhost:8081/admin>    | ✅ Anmeldung erfolgreich                   |
| Portainer  | <http://localhost:9000>          | ✅ Anmeldung erfolgreich                   |
| Watchtower | —                                | ⚠️ Exit-Code 1, siehe Abschnitt oben       |

Drei Dinge sind dabei aufgefallen:

1. **Ports sind endliche Ressourcen.** Pi-hole und nginx wollen beide Port 80,
   und Port 53 war unter WSL2 schon von `systemd-resolved` belegt. Genau dafür
   ist die Trennung von Host-Port und Container-Port da: Innen bleibt alles beim
   Standard, außen weicht man aus.
2. **Der Docker-Socket ist mächtig.** Portainer und Watchtower brauchen ihn
   beide — und wer ihn hat, kontrolliert den ganzen Host.
3. **Ein Image ohne Pflege veraltet.** Watchtower ist daran gescheitert, dass
   sein eigenes Image zwei Jahre alt ist.

## Container mit Docker Desktop steuern

Docker Desktop unter Windows zeigt dieselben Container grafisch an, sofern die
**WSL-Integration** für die verwendete Distribution aktiviert ist:

> *Settings → Resources → WSL Integration* → gewünschte Distribution einschalten
> → *Apply & Restart*

Anschließend erscheinen die Container unter **Containers**. Über die Schaltflächen
in der Zeile lassen sie sich starten (▶), stoppen (■) und wieder entfernen (🗑).
Ein Klick auf den Namen öffnet Logs, Umgebungsvariablen und eine Shell im
Container. Die Spalte *Port(s)* führt direkt zur jeweiligen Weboberfläche.

Ohne aktivierte WSL-Integration betreibt Docker Desktop einen **eigenen**
Docker-Daemon. Die hier gestarteten Container laufen dann im Daemon der
WSL-Distribution und tauchen in Docker Desktop nicht auf.

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
