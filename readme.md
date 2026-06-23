DEMO: https://uijtdehaag.github.io/BAG-Adres-Selectie/

Vibe Coding met Claude:

---

# BAG Adresselectie — Reproductieprompt

Bouw een **single-file HTML/JS/CSS app** (geen server, geen API-key) voor het selecteren en exporteren van Nederlandse BAG-adressen via een kaart.

---

## Technische basis

- **Leaflet 1.9.4** via unpkg CDN
- Fonts: **Inter** + **JetBrains Mono** via Google Fonts
- Volledig client-side

---

## Layout & Design

Flexbox: sidebar (340px) links + kaart rechts (`flex:1`).

**Kleuren:**
```
--blue: #1B4F8A  --orange: #E8610A  --bg: #F2F4F7
--surface: #fff  --border: #D8DCE6  --muted: #6B7280
```

**Sidebar secties (top→bottom):**
1. Blauwe header — titel "BAG Adresselectie", subtitel "Vibe Code", ronde `i`-knop rechts
2. Zoekbalk met autocomplete-dropdown
3. Toolbar — twee rijen:
   - **Rij 1 Selectie** (blauw, class `sel-btn`): Klik · Polygoon · PC6 — mode-label als eigen regel eronder in blauw cursief
   - **Rij 2 Lagen** (teal `#0E7490`, class `layer-btn`): PC6-labels · Luchtfoto
4. Scrollbare adreslijst met chips (straatnaam mono, postcode, gebruiksdoel-badge blauw)
5. Footer: "Wis alles" + oranje "Exporteer CSV"

**Desktop:** smalle blauwe pijl-tab (`#sidebar-toggle`, `position:absolute`, `left: var(--sidebar-w)`) klapt sidebar in/uit. Na toggle: `setTimeout(() => map.invalidateSize(), 300)`.

**Mobiel (≤700px):**
- Hamburger `#hamburger` — `position: fixed`, `z-index: 1000`, linksboven, **buiten `#app`** direct onder `<body>`. Animeert naar ×.
- Sidebar: `position: fixed`, off-canvas drawer (`transform: translateX(-110%)`), `.open` klapt in
- Backdrop `#sidebar-backdrop`: `position: fixed`, `z-index: 998`, sluit sidebar bij klik
- Kaart: `position: fixed; inset: 0; z-index: 1`
- Desktop pijl-tab: `display: none !important` op mobiel

---

## Kaart initialisatie

```javascript
// Zoomknoppen RECHTSBOVEN (niet standaard linksboven — conflicteert met hamburger)
const map = L.map('map', { zoomControl: false }).setView([51.5719, 4.7683], 18);
L.control.zoom({ position: 'topright' }).addTo(map);

const cartoLayer    = L.tileLayer('https://{s}.basemaps.cartocdn.com/light_all/{z}/{x}/{y}{r}.png', {...});
const luchtfotoLayer = L.tileLayer('https://service.pdok.nl/hwh/luchtfotorgb/wmts/v1_0/Actueel_orthoHR/EPSG:3857/{z}/{x}/{y}.jpeg');
const bagLayer      = L.tileLayer.wms('https://service.pdok.nl/lv/bag/wms/v2_0', { layers:'pand', transparent:true, opacity:0.55, minZoom:14 });
```

CBS Postcode6 WFS laden bij `moveend`/`zoomend` (300ms debounce, minZoom 14):
```
GET https://service.pdok.nl/cbs/postcode6/2023/wfs/v1_0
  ?...&bbox=<minLat,minLng,maxLat,maxLng,EPSG:4326>   ← lat eerst!
```
Labels: `L.divIcon` class `pc6-label` — transparant, tekst `#1B4F8A` bold, `text-shadow: 0 0 3px #fff` (3×).

---

## PDOK API-endpoints

| Doel | Endpoint |
|---|---|
| Autocomplete | `locatieserver/v3_1/suggest?q=<term>&fq=type:adres&rows=8` |
| Adresdetail | `locatieserver/v3_1/lookup?id=<id>&fl=...` |
| Klik op kaart | `locatieserver/v3_1/reverse?lat=<lat>&lon=<lon>&type=adres&rows=1` |
| Gebied ophalen | `locatieserver/v3_1/reverse?lat=<c.lat>&lon=<c.lng>&rows=50&distance=<m>&start=<offset>` |

⚠️ Gebruik altijd `reverse` voor ruimtelijke queries. **Nooit** `free?q=*&fq=bboxField:[...]`.

Pagineer in blokken van 50 tot max 500 (`start=0,50,...`), stop als `page.length < 50`.

---

## Selectiemodi

### `click`
`map.on('click')` → `reverse` → adres toevoegen. Min zoom 14.

### `poly` — Vrije polygoon
- Enkele klik: punt toevoegen als `L.circleMarker` (r=4, blauw/wit), rubber-band preview via `map.on('mousemove')`
- Dubbelklik: `L.DomEvent.stop(e)` (voorkom zoom), polygoon sluiten
- Filter: bbox-grofcheck + **ray-casting punt-in-polygoon**
- Eerste nieuw punt wist vorige voltooide polygoon automatisch

```javascript
function pointInPolygon(point, vs) {
  const [py, px] = point;
  let inside = false;
  for (let i = 0, j = vs.length-1; i < vs.length; j = i++) {
    const [iy,ix] = vs[i], [jy,jx] = vs[j];
    if (((iy>py) !== (jy>py)) && px < ((jx-ix)*(py-iy))/(jy-iy)+ix)
      inside = !inside;
  }
  return inside;
}
```

### `pc6sel` — Postcodegebied
1. OGC INTERSECTS op CBS Postcode6 WFS → haal `postcode6` veldwaarde op
2. Highlight vlak oranje (`L.geoJSON`, `fillOpacity: 0.12`)
3. `locatieserver/free?q=<pc6>&fq=postcode:<pc6>` gepagineerd → filter met `bounds.contains()`

---

## Toggle-gedrag (alle modi)

Al geselecteerd + opnieuw aangeklikt/geselecteerd → **verwijderen**. Bij bulk: toast `"3 toegevoegd, 1 verwijderd"`.

## Opruimen — `clearPolyDraw()`

Verwijdert: polyLine, polyPreview, polyLayer, alle punt-markers, mousemove listener.

| Trigger | Opruiming |
|---|---|
| Wisselen van modus | `clearPolyDraw()` + PC6-highlight weg |
| Nieuw eerste punt | Vorige polyLayer + PC6-highlight weg |
| "Wis alles" | `markers.clearLayers()` + `clearPolyDraw()` + PC6-highlight weg |

---

## Gebruiksdoel (async per adres)

```
GET https://service.pdok.nl/lv/bag/wfs/v2_0
  ?typeName=bag:pand&outputFormat=application/json
  &propertyName=gebruiksdoel,oppervlakte_min,oppervlakte_max,status
  &count=1&FILTER=<OGC INTERSECTS op lat,lon>
```

OGC filter (coördinaten als `lat,lon` — let op volgorde!):
```xml
<Filter xmlns="http://www.opengis.net/ogc" xmlns:gml="http://www.opengis.net/gml">
  <Intersects>
    <PropertyName>geom</PropertyName>
    <gml:Point srsName="urn:ogc:def:crs:EPSG::4326">
      <gml:coordinates>52.0923,5.2148</gml:coordinates>
    </gml:Point>
  </Intersects>
</Filter>
```

⚠️ **Nooit CQL_FILTER op postcode/huisnummer — werkt niet betrouwbaar.**  
⚠️ Controleer altijd of response begint met `{` vóór `JSON.parse`.  
Fallback: bbox `lon±0.0001,lat±0.0001,EPSG:4326` op dezelfde laag.  
**Snapshot `lat`/`lon` per async-call** — parallelle calls overschrijven anders elkaars resultaat.

---

## CSV-export

Velden: `id, straatnaam, huisnummer, huisletter, huisnummertoevoeging, postcode, woonplaatsnaam, gebruiksdoel, oppervlakte_m2, status, lat, lon`  
UTF-8 BOM (`\uFEFF`). Bestandsnaam: `bag-adressen-YYYY-MM-DD.csv`.

---

## Info modal

`i`-knop in header → modal met kaartlagen-tabel (naam · aanbieder · actualiteit-badge). Sluiten via ×-knop of klik buiten modal.

---

## Overige details

- `centroide_ll` parsen: `POINT(lon lat)` → `/POINT\(([^ ]+) ([^)]+)\)/`
- Toast: rechtsonder, 2,8s auto-hide, `fadeUp` animatie
- Loading spinner overlay op kaart tijdens API-calls
- Map-info overlay linksonder: huidige zoom + waarschuwing < 14
- Marker: oranje SVG-pin `L.divIcon` (20×26px, witte cirkel erin)
