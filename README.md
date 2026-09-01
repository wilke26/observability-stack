# Observability Stack für macOS

Lokaler, reproduzierbarer Observability-Stack für Docker Desktop:

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
nach. Beim ersten Start beginnt Alloy am Ende vorhandener Dateien: Historische
Einträge bleiben in Loki erhalten, neue Zeilen werden ohne Doppelimport
verarbeitet.

## Voraussetzungen

1. macOS Tahoe auf Apple Silicon oder Intel
2. Docker Desktop in einer aktuellen Version
3. Mindestens 4 GB für Docker verfügbare Arbeitsspeicher
4. Freie lokale Ports 3000, 3100 und 9090

Prüfen:

```bash
docker version
docker compose version
```

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

Danach in `.env` mindestens `GRAFANA_ADMIN_PASSWORD` ändern. Anschließend:

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

Wenn eine Anwendung auf dem Mac läuft, ist sie aus Docker Desktop gewöhnlich
unter `host.docker.internal` erreichbar. Beispiel für
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
von `/var/lib/docker/containers` ist unter Docker Desktop
für macOS nicht zuverlässig portabel und wird hier deshalb bewusst vermieden.

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
docker compose up -d
docker compose ps
docker compose logs --since=5m
```

5. Dashboard, Prometheus-Targets und Loki-Abfrage prüfen.

Alloy folgt einem eigenen Release-Zyklus. Vor einem Versionssprung müssen daher
die Alloy-Release-Notes und mögliche Änderungen an den `loki.*`-Komponenten
separat von Loki geprüft werden.

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

Der Stack verwendet ein Docker-Volume statt eines macOS-Bind-Mounts für
`/loki`; dadurch sind typische UID-Probleme bereits vermieden. Falls ein altes
Volume falsche Rechte enthält und die Daten entbehrlich sind, den Stack samt
Volumes neu anlegen:

```bash
docker compose down --volumes
docker compose up -d
```

### Apple-Silicon-Kompatibilität

Die verwendeten Images sind Multi-Architecture-Images. Ein erzwungenes
`platform: linux/amd64` ist nicht nötig und würde auf Apple Silicon unnötige
Emulation verursachen.

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
