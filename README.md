# Bestandsplan-Editor (Server-Variante mit Live-Karte)

Editor für Bestandspläne von Straßenabschnitten. Luftbild und Karte kommen direkt aus den amtlichen Diensten, die Lage ist in ETRS89 / UTM 32N, das PDF ist maßstäblich.

## Inhalt

| Datei | Zweck |
|---|---|
| `html/index.html` | der Editor (eine Datei, kein Build nötig) |
| `nginx.conf` | Webserver und Proxy für Kartendienste und Ortssuche |
| `proxy-tiles.inc` | gemeinsame Proxy- und Cache-Einstellungen |
| `docker-compose.yml` | Container-Definition (nginx 1.27) |

## Installation auf dem VPS

1. Ordner auf den Server kopieren, z. B. nach `/opt/bestandsplan`.
2. Im Ordner starten: `docker compose up -d`
3. Im Cloudflare Tunnel einen Public Hostname anlegen: `bestandsplan.hxmobility.work` → `http://localhost:8088`
   Läuft `cloudflared` selbst als Container, dann den Dienst in dasselbe Docker-Netz hängen und `http://bestandsplan:80` eintragen.

Wenn die Subdomain anders heißt: in `nginx.conf` die zwei Zeilen mit `bestandsplan.hxmobility.work` anpassen (Kennung gegenüber der Ortssuche).

## Nach dem Start prüfen

Diese Konfiguration wurde ohne Zugriff auf die echten Dienste erstellt. Bitte einmal testen:

- Seite: `https://bestandsplan.hxmobility.work/`
- Luftbild-Kachel (muss ein Bild von Stuttgart liefern):
  `/wms/dop?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&LAYERS=IMAGES_DOP_20_RGB&STYLES=&CRS=EPSG:25832&BBOX=513000,5403000,513512,5403512&WIDTH=512&HEIGHT=512&FORMAT=image/jpeg`
- Karten-Kachel:
  `/wms/basemap?SERVICE=WMS&VERSION=1.3.0&REQUEST=GetMap&LAYERS=de_basemapde_web_raster_farbe&STYLES=&CRS=EPSG:25832&BBOX=513000,5403000,513512,5403512&WIDTH=512&HEIGHT=512&FORMAT=image/png`
- Ortssuche: `/geocode?format=jsonv2&limit=3&countrycodes=de&q=Rosensteinstraße Stuttgart`

Fehler zeigt `docker logs bestandsplan`.

## Zugriffsschutz

Die Seite hat keine Anmeldung. Der Proxy spricht zwar nur drei feste Dienste an, trotzdem sollte nicht jeder über deinen Server Karten abrufen können. Empfehlung: in Cloudflare Zero Trust eine Access-Regel für die Subdomain setzen.

## Datenquellen und Pflichten

| Dienst | Quelle | Lizenz | Quellenvermerk |
|---|---|---|---|
| Luftbild DOP20 | LGL Baden-Württemberg, WMS `WMS_LGL-BW_ATKIS_DOP_20_C` | Datenlizenz Deutschland - Namensnennung - Version 2.0 | `LGL-BW (Jahr) Datenlizenz Deutschland - Namensnennung - Version 2.0, www.lgl-bw.de` |
| Karte | BKG, basemap.de Web Raster, WMS `wms_basemapde` | CC BY 4.0 | `© GeoBasis-DE / BKG (Jahr) CC BY 4.0` |
| Ortssuche | Nominatim, OpenStreetMap | ODbL | `© OpenStreetMap-Mitwirkende` |

Der Editor zeigt den Quellenvermerk am Bildschirm und druckt ihn ins PDF.

Das Luftbild deckt nur Baden-Württemberg ab. Für andere Länder gibt es eigene Dienste, die sich in `nginx.conf` und in der `MAPS`-Liste in `index.html` ergänzen lassen.

### Nutzungsregeln der Ortssuche (wichtig)

Die Suche nutzt den öffentlichen Nominatim-Server der OpenStreetMap Foundation. Dessen Nutzungsregeln gelten für dich als Betreiber: https://operations.osmfoundation.org/policies/nominatim/

- höchstens 1 Anfrage pro Sekunde (der Proxy begrenzt das)
- keine Massenabfragen, keine Autovervollständigung (der Editor sucht nur auf Enter)
- Anfragen müssen die Anwendung benennen (User-Agent und Referer sind in `nginx.conf` gesetzt)
- der Dienst läuft auf gespendeten Servern und kann ohne Vorwarnung gesperrt werden

Für einzelne Suchen im Büroalltag passt das. Wenn du das nicht willst, die `location = /geocode` in `nginx.conf` löschen; der Editor funktioniert dann ohne Suche weiter.

## Speicherung

Der Plan liegt im Browser (localStorage) und wird beim Neuladen wiederhergestellt. Verlässlich und geräteübergreifend ist nur „Projekt speichern" (JSON-Datei).

Kartenkacheln puffert der Server bis zu 90 Tage in einem Docker-Volume (höchstens 3 GB). Leeren: `docker compose down && docker volume rm bestandsplan_tiles`.

## Aktualisieren

Neue `index.html` in `html/` legen, fertig. Ein Neustart ist nur nach Änderungen an `nginx.conf` nötig: `docker compose restart`.
