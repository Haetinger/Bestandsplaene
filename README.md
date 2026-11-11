# Bestandsplan-Editor (Server-Variante mit Live-Karte)

Editor für Bestandspläne von Straßenabschnitten als Grundlage für Verkehrszeichenpläne. „Grundlage erzeugen" übernimmt für einen Ausschnitt das Kataster (Flurstücke, Gebäude, Verkehrsflächen) als Strichzeichnung und schlägt die Straßen mit Fahrstreifen und Markierung vor. Luftbild und Karten sind als Hintergrund zuschaltbar. Lage in ETRS89 / UTM 32N, PDF maßstäblich.

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
- Kataster (muss JSON mit einem Gebäude liefern):
  `/wfs/alkis?SERVICE=WFS&VERSION=2.0.0&REQUEST=GetFeature&TYPENAMES=nora:v_al_gebaeude&COUNT=1&SRSNAME=EPSG:25832&BBOX=513000,5403000,513200,5403200,EPSG:25832&OUTPUTFORMAT=application/json`
- Straßen aus OpenStreetMap (muss JSON mit `elements` liefern):
  `/overpass?data=[out:json];way[highway=motorway](48.78,9.17,48.79,9.19);out body geom 1;`

Fehler zeigt `docker logs bestandsplan`.

## Zugriffsschutz

Die Seite hat keine Anmeldung. Der Proxy spricht zwar nur drei feste Dienste an, trotzdem sollte nicht jeder über deinen Server Karten abrufen können. Empfehlung: in Cloudflare Zero Trust eine Access-Regel für die Subdomain setzen.

## Datenquellen und Pflichten

| Dienst | Quelle | Lizenz | Quellenvermerk |
|---|---|---|---|
| Luftbild DOP20 | LGL Baden-Württemberg, WMS `WMS_LGL-BW_ATKIS_DOP_20_C` | Datenlizenz Deutschland - Namensnennung - Version 2.0 | `LGL-BW (Jahr) Datenlizenz Deutschland - Namensnennung - Version 2.0, www.lgl-bw.de` |
| Karte | BKG, basemap.de Web Raster, WMS `wms_basemapde` | CC BY 4.0 | `© GeoBasis-DE / BKG (Jahr) CC BY 4.0` |
| Kataster (Vektor und Bild) | LGL Baden-Württemberg, WFS `WFS_LGL-BW_ALKIS`, WMS `WMS_LGL-BW_ALKIS_Basis_transparent` | Datenlizenz Deutschland - Namensnennung - Version 2.0 | wie Luftbild |
| Straßenachsen | OpenStreetMap über die Overpass API | ODbL | `© OpenStreetMap-Mitwirkende` |
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

### Nutzungsregeln der Overpass API

Die Straßenachsen kommen vom öffentlichen Server overpass-api.de: https://wiki.openstreetmap.org/wiki/Overpass_API

- Für regelmäßig genutzte Anwendungen gilt als Richtwert: unter 100 Abfragen und unter 10 MB pro Tag.
- Der Editor stellt eine Abfrage pro „Grundlage erzeugen", der Server puffert sie 7 Tage.
- Server-Adressen können bei Überlast gesperrt werden. Dann liefert „Grundlage erzeugen" nur das Kataster, und die Straßen zeichnest du von Hand.

### Was die automatische Grundlage kann und was nicht

- Das Kataster ist amtlich und lagegenau, zeigt aber Grundstücke und Gebäude, keine Fahrbahnränder.
- Die Straßen sind ein Vorschlag: Achse und Fahrstreifenzahl aus OpenStreetMap, Breiten nach Regelwerten. Vor der Verwendung gegen das Luftbild prüfen und anpassen.
- Exakte Fahrbahnränder und Markierungen gibt es nur aus einer Bestandsvermessung.
- Flurstücksnummern, Hausnummern und Straßennamen kommen mit amtlicher Lage und Drehung aus dem Kataster.
- Die Bestandsbeschilderung ist ein Vorschlag aus OpenStreetMap (Verkehrszeichen, Stopp, Vorfahrt gewähren, Ampeln). Sie ist lückenhaft, die Standorte sind ungefähr und ohne Blickrichtung. Vor Ort oder im Luftbild prüfen.

## Speicherung

Der Plan liegt im Browser (localStorage) und wird beim Neuladen wiederhergestellt. Verlässlich und geräteübergreifend ist nur „Projekt speichern" (JSON-Datei).

Kartenkacheln puffert der Server bis zu 90 Tage in einem Docker-Volume (höchstens 3 GB). Leeren: `docker compose down && docker volume rm bestandsplan_tiles`.

## Aktualisieren

Geänderte Dateien ins GitHub-Repo hochladen, dann auf dem Server:

```bash
cd /opt/bestandsplan && git pull && docker compose restart
```

Im Browser danach einmal mit Strg+F5 neu laden.
