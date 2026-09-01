# Plattformübergreifender Observability Stack

Lokaler, reproduzierbarer Observability-Stack für Docker Desktop oder Docker
Engine:

- **Prometheus** sammelt und speichert Metriken.
- **Grafana** visualisiert Metriken und Logs.
- **Loki** speichert Logs.
- **Grafana Alloy** liest, verarbeitet und versendet lokale Beispiel- und
  Anwendungslogs.
- **OpenTelemetry Collector** ist als optionales Compose-Profil vorbereitet.
- Ein kleiner **Log-Generator** erzeugt sofort sichtbare Testdaten.

Alle Datenbanken verwenden benannte Docker-Volumes. Ein normales `docker compose
down` behält die Daten daher bei.

## Versionsstand

| Komponente | Image/Version |
|---|---|
| Prometheus | `prom/prometheus:v3.12.0` |
| Grafana | `grafana/grafana:13.1.0` |
| Loki | `grafana/loki:3.6.11` |
| Grafana Alloy | `grafana/alloy:v1.19.2` |
| OpenTelemetry Collector Contrib | `otel/opentelemetry-collector-contrib:0.157.0` |

Promtail ist seit dem 2. März 2026 EOL und wurde deshalb durch Grafana Alloy
ersetzt. Die Alloy-Konfiguration bildet die bisherige JSON-Pipeline vollständig
nach. Im Normalbetrieb liest Alloy neue Dateien vom Anfang. Für den einmaligen
Wechsel von Promtail ist weiter unten ein eigener, duplikatfreier
Migrationsablauf dokumentiert.

## Voraussetzungen

1. Docker Desktop oder Docker Engine in einer aktuellen Version
2. Docker Compose v2 und Unterstützung für Linux-Container
3. Mindestens 4 GB für Docker verfügbarer Arbeitsspeicher
4. Freie lokale Ports 3000, 3100 und 9090

Prüfen:

```bash
docker version
docker compose version
```

Getestete Entwicklungsumgebung ist macOS Tahoe auf Apple Silicon. Die
Konfiguration verwendet ausschließlich Linux-Container und wird in CI auf
Ubuntu validiert. Sie ist daher ebenso für Windows 11 mit WSL2 sowie für
native Linux-Systeme vorgesehen.

## Plattformhinweise

### Windows 11

Docker Desktop muss mit dem WSL2-Backend im Linux-Container-Modus laufen. Die
Beispiele in dieser Dokumentation verwenden eine POSIX-Shell; am einfachsten
werden Repository und Befehle deshalb innerhalb einer WSL2-Distribution
verwendet. Native PowerShell benötigt für Befehle wie `cp`, `sed`, `chmod`,
`test`, `rm` und für `$PWD` entsprechende PowerShell-Varianten.

Ein Repository im WSL-Dateisystem bietet außerdem verlässlichere
Dateiberechtigungen und meist bessere Bind-Mount-Performance als ein Checkout
unter `C:\`.

### Linux

Prometheus erreicht Anwendungen auf dem Docker-Host über
`host.docker.internal`. Der dafür nötige `host-gateway`-Eintrag ist in
`docker-compose.yml` explizit gesetzt, weil native Docker-Engines den Namen im
Gegensatz zu Docker Desktop nicht immer automatisch bereitstellen.

Der Log-Generator kann `sample-logs/demo.log` auf nativen Linux-Systemen als
`root` anlegen. Das beeinträchtigt die Verarbeitung nicht, kann aber für eine
spätere Bearbeitung oder Löschung auf dem Host eine Rechteanpassung erfordern.
Auf Systemen mit SELinux, insbesondere Fedora und RHEL, benötigen die
Bind-Mounts je nach lokaler Policy zusätzlich ein passendes `:z`- oder
`:Z`-Label. Die persistenten Daten von Prometheus, Grafana, Loki und Alloy
liegen in benannten Volumes und sind davon nicht betroffen.

## Verzeichnisstruktur

```text
observability-stack/
├── docker-compose.yml
├── .env.example
├── prometheus/prometheus.yml
├── grafana/
│   ├── dashboards/stack-overview.json
│   └── provisioning/
│       ├── dashboards/dashboards.yml
│       └── datasources/datasources.yml
├── loki/config.yml
├── alloy/config.alloy
├── otel-collector/config.yml
└── sample-logs/
```

## Erster Start

Im Repository-Verzeichnis:

```bash
cp .env.example .env
```

Danach in `.env` mindestens `GRAFANA_ADMIN_PASSWORD` ändern.

Prometheus benötigt denselben `METRICS_TOKEN`, der im ISD-Backend konfiguriert
ist. Bei der empfohlenen Verzeichnisstruktur mit beiden Repositories als
Nachbarn wird die geschützte Token-Datei so angelegt:

```bash
mkdir -p secrets
sed -n 's/^METRICS_TOKEN=//p' ../isd/.env > secrets/isd_metrics_token
chmod 600 secrets/isd_metrics_token
test -s secrets/isd_metrics_token && echo "Metrics-Token vorhanden"
```

`chmod 600` schützt die Datei auf POSIX-Dateisystemen. Bei einem nativen
Windows-Checkout müssen die Zugriffsrechte stattdessen über Windows-ACLs
gesetzt werden; innerhalb von WSL2 gilt der gezeigte Befehl unverändert.

Liegt das Backend an einem anderen Ort, muss der Pfad zu dessen `.env`
entsprechend angepasst werden. Compose bricht absichtlich mit einer klaren
Fehlermeldung ab, wenn die Token-Datei fehlt; es wird kein gleichnamiges
Verzeichnis mehr automatisch erzeugt.

Anschließend:

```bash
docker compose config --quiet
docker compose pull
docker compose up -d
docker compose ps
```

Oberflächen:

- Grafana: <http://localhost:3000>
- Prometheus: <http://localhost:9090>
- Loki-Readiness: <http://localhost:3100/ready>

Die Grafana-Zugangsdaten stehen in `.env`. Unter **Dashboards → Observability**
ist das Dashboard **Observability Stack – Übersicht** automatisch vorhanden.
Prometheus und Loki sind bereits als Datasources eingerichtet. Alloy läuft
ohne veröffentlichten Host-Port; seine interne UI und Metriken sind nur im
Compose-Netz unter `alloy:12345` erreichbar.

## Funktion testen

### 1. Containerstatus

```bash
docker compose ps
```

Prometheus, Grafana und Alloy sollten nach kurzer Zeit `healthy` anzeigen.
Loki und der Log-Generator sollten `running` sein.

### 2. HTTP-Endpunkte

```bash
curl -fsS http://localhost:9090/-/ready
curl -fsS http://localhost:3000/api/health
curl -fsS http://localhost:3100/ready
```

### 3. Prometheus-Targets

<http://localhost:9090/targets> öffnen. Die Jobs `prometheus`, `grafana`,
`loki` und `alloy` sollten `UP` sein. `otel-collector` ist ohne das
optionale Profil erwartungsgemäß `DOWN`.

Der mitgelieferte ISD-Scrape sendet den Bearer-Token per HTTP ausschließlich
über den lokalen Docker-Host-Zugang an `host.docker.internal`. Diese
Konfiguration ist nur für eine lokale Entwicklungsumgebung vorgesehen. Für
einen entfernten, gemeinsam genutzten oder produktiven Metrics-Endpunkt muss
in `prometheus/prometheus.yml` `scheme: https` gesetzt und eine gültige
TLS-Konfiguration verwendet werden.

### 4. Logs in Loki

Der Dienst `log-generator` schreibt alle 15 Sekunden eine JSON-Zeile nach
`sample-logs/demo.log`. Alloy parst die JSON-Zeilen und sendet sie an Loki. Im
mitgelieferten Grafana-Dashboard erscheinen die Meldungen normalerweise nach
spätestens 30 Sekunden.

Alternativ in Grafana **Explore** öffnen, Loki auswählen und abfragen:

```logql
{job="sample-logs"}
```

Eigene JSON-Logs können in `sample-logs/*.log` geschrieben werden. Das
vorgesehene Format ist:

```json
{"timestamp":"2026-07-27T12:00:00Z","level":"info","service":"fastapi","message":"Import abgeschlossen"}
```

## OpenTelemetry Collector aktivieren

Der Collector ist standardmäßig deaktiviert und verändert den Basis-Stack
nicht. Aktivieren:

```bash
docker compose --profile otel up -d
docker compose --profile otel ps
```

Anwendungen können OTLP an folgende lokalen Endpunkte senden:

- gRPC: `localhost:4317`
- HTTP: `localhost:4318`
- aus anderen Compose-Containern: `otel-collector:4317` beziehungsweise
  `otel-collector:4318`

Der vorbereitete Collector nimmt Metriken, Logs und Traces an. Metriken werden
auf Port 8889 für Prometheus bereitgestellt; Logs und Traces erscheinen zur
Demonstration im Collector-Containerlog. Für eine Produktionsumgebung sollten
dedizierte Backends, TLS, Authentifizierung und passende Limits ergänzt werden.

Nur den optionalen Collector wieder stoppen:

```bash
docker compose --profile otel stop otel-collector
docker compose rm -f otel-collector
```

## Eigene Anwendungen anbinden

### Prometheus-Metriken

Wenn eine Anwendung auf dem Docker-Host läuft, ist sie aus dem Stack unter
`host.docker.internal` erreichbar. Beispiel für
`prometheus/prometheus.yml`:

```yaml
  - job_name: vulnprocessing
    metrics_path: /metrics
    static_configs:
      - targets: ["host.docker.internal:8000"]
```

Danach Konfiguration prüfen und Prometheus neu laden:

```bash
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
curl -fsS -X POST http://localhost:9090/-/reload
```

Für Anwendungen in einem anderen Compose-Projekt ist ein gemeinsames externes
Docker-Netzwerk oder ein Scrape über den veröffentlichten Host-Port sinnvoll.

### Logs

Für den lokalen Einstieg kann eine Anwendung strukturierte JSON-Logs in eine
Datei unter `sample-logs/` schreiben. Bei containerisierten Anwendungen kann
Alloy um eine passende Docker- oder OTLP-Quelle erweitert werden. Das Mounten
von `/var/lib/docker/containers` ist zwischen Docker Desktop, nativen Engines
und rootless Installationen nicht zuverlässig portabel und wird hier deshalb
bewusst vermieden.

## Stoppen und neu starten

Stoppen, Daten behalten:

```bash
docker compose down
```

Neu starten:

```bash
docker compose up -d
```

Container neu erzeugen, Daten behalten:

```bash
docker compose up -d --force-recreate
```

## Update

Die Versionen werden in `.env` festgelegt. Nicht blind `latest` verwenden.

1. Release Notes und Breaking Changes der jeweiligen Komponente lesen.
2. Backup der Volumes erstellen.
3. Eine Version in `.env` ändern.
4. Konfiguration und neues Image testen:

```bash
docker compose config --quiet
docker compose pull
docker compose up -d --remove-orphans
docker compose ps
docker compose logs --since=5m
```

5. Dashboard, Prometheus-Targets und Loki-Abfrage prüfen.

Alloy folgt einem eigenen Release-Zyklus. Vor einem Versionssprung müssen daher
die Alloy-Release-Notes und mögliche Änderungen an den `loki.*`-Komponenten
separat von Loki geprüft werden.

### Einmalige Migration von Promtail zu Alloy

Dieser Abschnitt gilt nur für Installationen, die noch mit Promtail liefen und
noch keine Alloy-Lesepositionen im Volume `observability-stack_alloy-data`
besitzen. Bereits auf Alloy migrierte Installationen verwenden den normalen
Update-Ablauf oben.

Beim ersten Alloy-Start werden vorhandene Dateien einmalig am Ende geöffnet,
weil ihre älteren Einträge bereits von Promtail an Loki übertragen wurden.
`--remove-orphans` entfernt dabei den nicht mehr definierten, sonst weiterhin
laufenden Promtail-Container:

```bash
ALLOY_TAIL_FROM_END=true docker compose up -d --remove-orphans
docker compose ps alloy
```

Sobald Alloy `healthy` ist, wird ausschließlich Alloy mit der normalen
Einstellung neu erzeugt. Gespeicherte Positionen werden dabei beibehalten;
neu entdeckte oder rotierte Dateien beginnen künftig wieder am Anfang:

```bash
docker compose up -d --force-recreate alloy
docker compose ps alloy
```

`ALLOY_TAIL_FROM_END` darf nicht dauerhaft in `.env` auf `true` gesetzt werden,
da sonst Zeilen übersprungen werden können, die vor der nächsten Dateisuche in
einer neuen Logdatei geschrieben wurden.

## Backup und Wiederherstellung

Die persistenten Daten liegen in den benannten Volumes
`observability-stack_prometheus-data`, `observability-stack_grafana-data`,
`observability-stack_loki-data` und `observability-stack_alloy-data`.

Das frühere Volume `observability-stack_promtail-data` wird nicht mehr
eingebunden. Es enthält keine Logdaten, sondern nur die alten Datei-Offsets und
kann nach erfolgreicher Prüfung der Alloy-Pipeline manuell entfernt werden.

Für ein einfaches lokales Backup zuerst Schreibzugriffe stoppen:

```bash
docker compose stop
mkdir -p backups
```

Beispiel für Grafana:

```bash
docker run --rm \
  -v observability-stack_grafana-data:/source:ro \
  -v "$PWD/backups:/backup" \
  busybox:1.37.0 \
  tar -czf /backup/grafana-data.tgz -C /source .
```

Danach:

```bash
docker compose start
```

Für wichtige Umgebungen sollten alle Volumes separat gesichert und
Wiederherstellungen regelmäßig getestet werden.

## Daten vollständig löschen

> Der folgende Befehl löscht alle Metriken, Loki-Logs, Grafana-Änderungen und
> Alloy-Lesepositionen dieses Stacks dauerhaft.

```bash
docker compose down --volumes
```

Die Datei `sample-logs/demo.log` ist ein Bind-Mount und kann separat gelöscht
werden:

```bash
rm sample-logs/demo.log
```

## Troubleshooting

### Ein Port ist bereits belegt

Fehlermeldungen wie `address already in use` bedeuten, dass ein lokaler Port
belegt ist. In `.env` beispielsweise ändern:

```dotenv
GRAFANA_PORT=3001
```

Danach `docker compose up -d` erneut ausführen.

### Grafana startet nicht oder meldet ein ungültiges Passwort

Prüfen, ob `.env` existiert und `GRAFANA_ADMIN_PASSWORD` gesetzt ist:

```bash
docker compose config --quiet
docker compose logs grafana
```

Das in `.env` gesetzte Admin-Passwort wird nur beim erstmaligen Anlegen der
Grafana-Datenbank verwendet. Bei vorhandenem Volume ändert ein neuer `.env`-
Wert das bereits gespeicherte Passwort nicht. Lokal kann es so zurückgesetzt
werden:

```bash
docker compose exec grafana grafana cli admin reset-admin-password 'NEUES_PASSWORT'
```

### Prometheus-Target ist DOWN

1. <http://localhost:9090/targets> öffnen und die Fehlermeldung ansehen.
2. Container und interne Namensauflösung prüfen:

```bash
docker compose ps
docker compose logs prometheus
```

`otel-collector` ist ohne Profil absichtlich `DOWN`. Alle anderen mitgelieferten
Targets sollten `UP` sein.

### Keine Logs in Grafana

```bash
docker compose logs log-generator alloy loki
tail -n 5 sample-logs/demo.log
```

Zusätzlich kontrollieren:

- In Grafana wirklich die Loki-Datasource verwenden.
- Zeitraum auf „Letzte 30 Minuten“ stellen.
- LogQL-Abfrage `{job="sample-logs"}` verwenden.
- Nach dem ersten Start bis zu 30 Sekunden warten.

Alloys Komponentenstatus ist innerhalb des Compose-Netzes unter
`http://alloy:12345` verfügbar. Die laufende Konfiguration lässt sich ohne
Host-Port so prüfen:

```bash
docker compose exec alloy alloy validate /etc/alloy/config.alloy
docker compose exec alloy /bin/bash -ec \
  "exec 3<>/dev/tcp/127.0.0.1/12345; printf 'GET /-/healthy HTTP/1.0\\r\\n\\r\\n' >&3; cat <&3"
```

### Loki meldet Berechtigungsfehler

Der Stack verwendet ein Docker-Volume statt eines Host-Bind-Mounts für `/loki`;
dadurch sind typische UID-Probleme bereits vermieden. Falls ein altes
Volume falsche Rechte enthält und die Daten entbehrlich sind, den Stack samt
Volumes neu anlegen:

```bash
docker compose down --volumes
docker compose up -d
```

### CPU-Architektur

Die verwendeten Images sind Multi-Architecture-Images für AMD64 und ARM64. Ein
erzwungenes `platform: linux/amd64` ist nicht nötig und würde auf ARM64-Systemen
wie Apple Silicon unnötige Emulation verursachen.

### Diagnoseübersicht erzeugen

```bash
docker compose ps
docker compose config
docker compose logs --since=10m
docker system df
```

Keine Zugangsdaten aus `.env` in Issues oder öffentliche Logs kopieren.

## Sicherheit

- Alle veröffentlichten Ports sind standardmäßig nur an `127.0.0.1` gebunden.
- Grafana-Self-Sign-up und Telemetrie sind deaktiviert.
- `.env` wird nicht versioniert.
- Das Setup ist für lokale Entwicklung gedacht, nicht für unveränderten
  Internetbetrieb.
- Für produktiven Betrieb sind mindestens TLS, Authentifizierung, Secret
  Management, Netzwerkregeln, Backups und Ressourcenlimits erforderlich.

## Lizenzhinweis

Dieses Gerüst enthält nur Konfiguration. Die verwendeten Komponenten und
Container-Images unterliegen ihren jeweiligen Open-Source-Lizenzen.
