Hier is de volledige reproductieprompt, inclusief alle geleerde lessen:

---

# BAG Adresselectie Tool — Reproductieprompt

Bouw een **single-file HTML/JS/CSS webapplicatie** zonder server of API-key waarmee Nederlandse BAG-adressen op een kaart geselecteerd en als CSV geëxporteerd kunnen worden.

---

## Stack

- **Leaflet.js 1.9.4** via CDN (`unpkg.com`)
- **Fonts:** Inter + JetBrains Mono via Google Fonts
- **Kaart tiles:** CartoDB Light (`basemaps.cartocdn.com/light_all`)
- Alles client-side, geen dependencies buiten CDN

---

## Layout

Flexbox: `#sidebar` (340px, `flex-shrink: 0`) links + `#map-pane` (`flex: 1`) rechts. Sidebar is **inklapbaar** via een blauwe tab-knop (`#sidebar-toggle`) die aan de rechterzijde van de sidebar hangt, met CSS `transition: transform .28s`. Na toggle: `map.invalidateSize()` na 300ms.

Op mobiel (`max-width: 600px`): sidebar `position: absolute`, neemt volle breedte in en schuift over de kaart.

**Kleurenpalet:**
```
--blue: #1B4F8A       (header, selectieknoppen actief, polygoon)
--blue-light: #2563B0
--orange: #E8610A     (markers, export-knop, chip-hover)
--bg: #F2F4F7
--surface: #FFFFFF
--border: #D8DCE6
--text: #1A1A2E
--muted: #6B7280
--radius: 6px
```

---

## Sidebar structuur (van boven naar beneden)

### 1. Header
Blauwe balk met titel, subtitel, en een ronde **info-knop** (`i`) rechts. Info-knop opent een modal met uitleg over de app en kaartlagen (zie Info Modal).

### 2. Zoekbalk
Adres-autocomplete via PDOK Locatieserver suggest. Resultaten in dropdown, klik → zoom naar adres + voeg toe.

### 3. Toolbar — twee aparte rijen

**Rij 1 — Selectie** (blauw thema, class `sel-btn`):
- `#btn-click` — Klik (standaard actief)
- `#btn-poly` — Polygoon
- `#btn-pc6sel` — PC6

Onder de knoppen: `#mode-label` als eigen regel in blauw cursief met de uitleg van de actieve modus.

**Rij 2 — Lagen** (teal thema `#0E7490`, class `layer-btn`, visueel onderscheiden van selectieknoppen):
- `#btn-pc6` — PC6-labels (standaard actief)
- `#btn-luchtfoto` — Luchtfoto

### 4. Adreslijst
Scrollbare lijst met chips. Elke chip toont: straatnaam+huisnummer (JetBrains Mono), postcode · woonplaats, en een blauwe gebruiksdoel-badge. Badge toont "ophalen…" tijdens async laden. Klik op chip → zoom naar marker. × verwijdert het adres.

### 5. Footer
"Wis alles" knop + oranje "Exporteer CSV" knop.

---

## Kaartlagen

```javascript
// Basiskaart 1: CartoDB Light
const cartoLayer = L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', ...);

// Basiskaart 2: PDOK Luchtfoto HR
const luchtfotoLayer = L.tileLayer(
  'https://service.pdok.nl/hwh/luchtfotorgb/wmts/v1_0/Actueel_orthoHR/EPSG:3857/{z}/{x}/{y}.jpeg'
);

// BAG panden (visueel, WMS)
const bagLayer = L.tileLayer.wms('https://service.pdok.nl/lv/bag/wms/v2_0', {
  layers: 'pand', transparent: true, opacity: 0.55, minZoom: 14
});

// CBS Postcode6 (WFS, laden bij moveend/zoomend, min zoom 14)
```

Luchtfoto toggle: wissel `cartoLayer` ↔ `luchtfotoLayer`, `bagLayer.setOpacity(0.7)` op luchtfoto.

---

## PDOK API-endpoints — exact deze gebruiken

### Autocomplete
```
GET https://api.pdok.nl/bzk/locatieserver/search/v3_1/suggest
  ?q=<term>&fq=type:adres&rows=8&fl=weergavenaam,id,centroide_ll
```

### Adresdetail na suggest-keuze
```
GET https://api.pdok.nl/bzk/locatieserver/search/v3_1/lookup
  ?id=<id>&fl=weergavenaam,nummeraanduiding_id,straatnaam,huisnummer,
            huisletter,huisnummertoevoeging,postcode,woonplaatsnaam,centroide_ll
```

### Klik → dichtstbijzijnd adres
```
GET https://api.pdok.nl/bzk/locatieserver/search/v3_1/reverse
  ?lat=<lat>&lon=<lon>&type=adres&rows=1&fl=...
```
⚠️ Gebruik altijd `reverse`. NOOIT `free?q=*&fq=bboxField:[...]` — dat werkt niet.

### Polygoon / PC6 → adressen in gebied (max 500)
```
GET https://api.pdok.nl/bzk/locatieserver/search/v3_1/reverse
  ?lat=<center.lat>&lon=<center.lng>&type=adres&rows=50
  &distance=<straal in meters, max 10000>&start=<offset>&fl=...
```
Pagineer in blokken van 50 (`start=0,50,...500`), stop als page < 50. Bereken straal: Haversine van center naar hoek van bounding box.

### CBS Postcode6 WFS (vlakken + labels)
```
GET https://service.pdok.nl/cbs/postcode6/2023/wfs/v1_0
  ?service=WFS&version=2.0.0&request=GetFeature
  &typeName=postcode6&outputFormat=application/json&srsName=EPSG:4326
  &bbox=<minLat>,<minLng>,<maxLat>,<maxLng>,EPSG:4326   ← lat eerst!
  &count=500
```
Labels via `L.divIcon` class `pc6-label` op `layer.getBounds().getCenter()`. Styling: transparante achtergrond, tekst `#1B4F8A` bold, `text-shadow: 0 0 3px #fff` (3×). Laden bij `moveend`/`zoomend` met 300ms debounce, min zoom 14.

### Gebruiksdoel (async na adres toevoegen)
```
GET https://service.pdok.nl/lv/bag/wfs/v2_0?service=WFS&version=2.0.0
  &request=GetFeature&typeName=bag:pand
  &outputFormat=application/json
  &propertyName=gebruiksdoel,oppervlakte_min,oppervlakte_max,status
  &count=1&FILTER=<OGC INTERSECTS>
```

OGC INTERSECTS filter (punt in WGS84, volgorde `lat,lon`):
```xml
<Filter xmlns="http://www.opengis.net/ogc"
        xmlns:gml="http://www.opengis.net/gml">
  <Intersects>
    <PropertyName>geom</PropertyName>
    <gml:Point srsName="urn:ogc:def:crs:EPSG::4326">
      <gml:coordinates>52.0923,5.2148</gml:coordinates>
    </gml:Point>
  </Intersects>
</Filter>
```
⚠️ **Gebruik INTERSECTS, geen CQL_FILTER op postcode/huisnummer — dat werkt niet betrouwbaar.**
⚠️ Controleer altijd of response begint met `{` vóór `JSON.parse`.
Fallback: bbox `lon-0.0001,lat-0.0001,lon+0.0001,lat+0.0001,EPSG:4326` op dezelfde laag.
Snapshot `lat`/`lon` per async-aanroep zodat parallelle calls elkaar niet overschrijven.

---

## Selectiemodi

### `click`
Klik op kaart → `reverse` geocode → adres toevoegen. Minimum zoom 14.

### `poly` — Vrije polygoon
- Enkele klik: voeg hoekpunt toe als `L.circleMarker` (blauw, witte kern, r=4)
- Rubber-band preview: gestippelde lijn volgt cursor via `map.on('mousemove')`
- Dubbelklik: sluit polygoon (`L.DomEvent.stop(e)` om zoom te voorkomen), haal adressen op
- Filter: bounding box als groffilter, daarna **ray-casting punt-in-polygoon**
- Nieuwe polygoon starten wist automatisch de vorige voltooide polygoon

```javascript
function pointInPolygon(point, vs) {
  const [py, px] = point;
  let inside = false;
  for (let i = 0, j = vs.length - 1; i < vs.length; j = i++) {
    const [iy, ix] = vs[i], [jy, jx] = vs[j];
    if (((iy > py) !== (jy > py)) && px < ((jx - ix) * (py - iy)) / (jy - iy) + ix)
      inside = !inside;
  }
  return inside;
}
```

### `pc6sel` — Postcodegebied
- Klik op kaart → OGC INTERSECTS op CBS Postcode6 WFS → haal `postcode6` op
- Highlight het vlak oranje (`L.geoJSON`, `fillOpacity: 0.12`)
- Haal adressen op via Locatieserver `free?q=<pc6>&fq=postcode:<pc6>`, gepagineerd tot 500
- Filter resultaten met `bounds.contains([dlat, dlon])`

---

## Adresbeheer — toggle-gedrag

Zowel bij klik als bij bulk-selectie: **al geselecteerd → verwijder, nieuw → voeg toe**.
Toast bij bulk: `"3 toegevoegd, 1 verwijderd"`.

Oranje `L.divIcon` SVG-pin als marker (20×26px, witte cirkel erin).

---

## Opruimen — alle gevallen

`clearPolyDraw()` verwijdert: polyLine, polyPreview, polyLayer, alle punt-markers, mousemove listener.

| Actie | Wat wordt opgeruimd |
|---|---|
| Wisselen van modus | `clearPolyDraw()` + PC6-highlight |
| Nieuw polygoon eerste punt | Vorige polyLayer + PC6-highlight |
| "Wis alles" | `markers.clearLayers()` + `clearPolyDraw()` + PC6-highlight |

---

## Info Modal

Knop `i` in header → modal met:
- Uitleg applicatie
- Tabel per kaartlaag: naam · aanbieder · actualiteit badge (groen = dagelijks/actueel jaar, blauw = 2023)
- CSV-velden uitleg

Sluiten: ×-knop, klik buiten modal, geen Escape nodig.

---

## CSV-export

Velden: `id, straatnaam, huisnummer, huisletter, huisnummertoevoeging, postcode, woonplaatsnaam, gebruiksdoel, oppervlakte_m2, status, lat, lon`

UTF-8 BOM (`\uFEFF`) voor Excel-compatibiliteit. Bestandsnaam: `bag-adressen-YYYY-MM-DD.csv`.

---

## Overige details

- Map-info overlay linksonder: toon huidige zoom, waarschuwing als zoom < 14
- Loading spinner overlay op kaart tijdens API-calls
- Toast rechtsonder, 2,8 sec auto-hide, animatie `fadeUp`
- Vóór elke `JSON.parse`: controleer of text begint met `{`
- `centroide_ll` parsen: `POINT(lon lat)` → regex `POINT\(([^ ]+) ([^)]+)\)`