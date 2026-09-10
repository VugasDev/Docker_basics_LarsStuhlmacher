# Changelog

Alle nennenswerten Änderungen an diesem Projekt. Das Format orientiert sich an
[Keep a Changelog](https://keepachangelog.com/de/1.1.0/), die Datumsangaben
folgen ISO 8601.

---

## [ABGABE] — 2026-09-10

Stand der Abgabe. Alle vier Stacks laufen, alle Weboberflächen sind erreichbar.

### Behoben

#### Watchtower startet nicht mehr — Image gegen gepflegten Fork getauscht

Das im c't-3003-Video verwendete Image `containrrr/watchtower` ist mit
aktuellen Docker-Engines nicht mehr lauffähig.

| | |
| --- | --- |
| Image-Stand | 11.11.2023 |
| Docker-API des Images | 1.25 |
| Von der Engine verlangt | mindestens 1.44 |
| Folge | Container beendet sich sofort, **Exit-Code 1** |

```
client version 1.25 is too old.
Minimum supported API version is 1.44
```

Das Projekt `containrrr/watchtower` wird seit 2023 nicht mehr gepflegt. Ersetzt
durch den aktiv gepflegten Fork **`nickfedor/watchtower`**, der dieselben
Optionen versteht:

```
Watchtower 1.22.1 using Docker API v1.55
```

Die Original-Zeile aus dem Gist steht als Kommentar in `watchtower.yml`, damit
die Herkunft nachvollziehbar bleibt.

> Bemerkenswert daran: Watchtower ist an genau dem Problem gescheitert, das es
> selbst lösen soll — einem Image, das zu lange nicht aktualisiert wurde.

#### Pi-hole belegte einen unter Windows besetzten Port

Der DNS-Port des Pi-hole-Stacks lag auf Host-Port `5353`. Das ist der
mDNS-Standardport und unter Windows von `svchost` und Chrome belegt, weshalb
der Stack im Docker-Desktop-Daemon nicht startete:

```
ports are not available: exposing port UDP 0.0.0.0:5353 -> 127.0.0.1:0
```

Geändert auf **`5335`** — in beiden Umgebungen frei.

### Geändert

- **Watchtower-Optionen erweitert:**

  | Option | Wirkung |
  | --- | --- |
  | `--interval 3600` | Stündliche Prüfung statt der voreingestellten 24 Stunden |
  | `--cleanup` | Ersetzte Images werden entfernt, statt sich anzusammeln |

- Watchtower läuft mit `restart: unless-stopped` und meldet sich als `healthy`.

### Hinzugefügt

- **Readme-Abschnitt „Zwei Docker-Daemons: WSL und Docker Desktop"** —
  nativer WSL-Docker und Docker Desktop führen getrennte Bestände; Container aus
  dem einen Daemon sind im anderen unsichtbar. Mit Diagnosebefehlen.
- **Readme-Abschnitt „Stolperfalle: Docker-Socket-Mounts nach einem Neustart"** —
  Container, die `/var/run/docker.sock` aus einer WSL-Distribution mounten
  (Portainer, Watchtower), brechen nach jedem Docker-Desktop-Start mit
  **Exit-Code 127** ab. Docker Desktop legt den Bind-Mount unter
  `/mnt/wsl/docker-desktop-bind-mounts/` erst Sekunden später an. Die
  Neustart-Richtlinie greift nicht, weil der Fehler beim Erstellen der Task
  auftritt und nicht im laufenden Prozess. Abhilfe: die Container kurz darauf
  erneut starten.

---

## 2026-09-09

### Hinzugefügt

- **Grundstruktur** nach Aufgabenstellung: je ein Ordner mit eigener
  Compose-Datei für `pihole`, `portainer`, `watchtower` und `nginx`.
- **`nginx.yml`** — selbst erstellt, als Compose-Gegenstück zum Video-Befehl
  `docker run -p 80:80 nginx`, mit eigener Startseite unter `nginx/html/`.
- **Erläuterungsabschnitte in der Readme** zu allen vier Serveranwendungen:
  Funktionsprinzip, Konfiguration in diesem Projekt und Zugang.
- **Beobachtungen aus dem Praxisteil** und eine Anleitung zur Bedienung der
  Container in Docker Desktop.
- **`.gitignore`** für die Laufzeitdaten, die Docker beim Start anlegt
  (`etc-pihole/`, `portainer_data/`) — sie enthalten Datenbanken und
  Zugangsdaten und gehören nicht in die Versionsverwaltung.

### Geändert

- **Watchtower auf dieses Projekt begrenzt.** Ohne Einschränkung aktualisiert
  Watchtower *jeden* Container auf dem Host — bei einem Testlauf traf das prompt
  eine projektfremde Datenbank. Ergänzt um `--label-enable`; die zugehörigen
  Labels sind in `pihole.yml`, `portainer.yml` und `nginx.yml` gesetzt. Ein
  Prüflauf erfasst seitdem genau die drei gelabelten Container.
- **Pi-hole-Ports angepasst.** Port 53 ist unter WSL2 von `systemd-resolved`
  belegt, Port 80 gehört in diesem Projekt dem nginx-Stack. Das Webinterface
  liegt deshalb auf `8081`. Port 67 (DHCP) entfällt, Pi-hole wird hier nicht als
  DHCP-Server betrieben.

### Bekannte Einschränkung

- **Pi-holes DNS-Weiterleitung funktioniert unter WSL2 nicht.** Das Blocken
  greift nachweislich (`doubleclick.net` → `0.0.0.0`), die Weiterleitung an den
  Upstream läuft in einen Timeout. Ursache ist nicht Pi-hole, sondern die
  Umgebung: Ausgehende DNS-Anfragen auf UDP-Port 53 werden unter WSL2 auch
  direkt vom Host aus blockiert. Auf einem Raspberry Pi oder einem regulären
  Linux-Server tritt das nicht auf.

---

## Herkunft

Die Compose-Dateien für Pi-hole, Portainer und Watchtower sowie der Befehlsteil
der Readme stammen aus dem Video *Docker für c’t 3003* und dem zugehörigen
Gist: <https://gist.github.com/jamct/2e6c03f60319423bc4bc6c23fc0aa359>

Die Datei für nginx wurde selbst erstellt. Abweichungen vom Original sind in
den Compose-Dateien als `ANPASSUNG` bzw. `ABWEICHUNG` kommentiert.
