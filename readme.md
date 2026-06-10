# Earth Data – Interactieve Aardbol

Een interactieve 3D-aardbol gebouwd met **Three.js WebGPU** waarbij landen aanklikbaar zijn en gekoppeld kunnen worden aan databronnen en visualisaties.

---

## Opdracht / Idee

- Aanleveren: THREE JS aardbol, responsive, aanklikbaar per land.
- Student kiest data uit dat per land moet worden gevisualiseerd (on click).
- Student richt de database in om de data op te slaan.
- Student kiest grafiek/UI-element uit om de data in te projecteren.
- Student faciliteert het inlaad-proces met AJAX en denkt na over caching-mogelijkheden.

---

## Technische uitwerking

### Aardbol (`index.html`)

De aardbol is gebouwd op het Three.js WebGPU/TSL-voorbeeld. De rendering gebruikt **Three Shading Language (TSL)** voor alle shader-logica, waardoor parameters als atmosfeerkleur en roughness live aanpasbaar zijn via uniforms.

| Laag | Technologie | Toelichting |
|---|---|---|
| Rendering | Three.js WebGPU + TSL | PBR-materiaal met dag/nacht, wolken, bump mapping en atmosfeer |
| Landgrenzen | GeoJSON via `world-atlas@2` + `topojson-client` | 177 landen, 110m resolutie |
| Interactie | Raycasting + point-in-polygon | Hover = outline, klik = fill |
| Overlay | Canvas-textuur (4096×2048) | Equirectangulaire projectie, co-roteert met de bol |

#### Raycasting en landdetectie

Bij elke `mousemove` of `mousedown` wordt een ray vanuit de camera door het muispunt geschoten. Het raakpunt op de bol wordt omgezet naar geografische coördinaten (lon/lat) via de inverse van de `SphereGeometry`-formule:

```
x = −cos(φ) · sin(θ)   →   φ = atan2(z, −x)
```

De rotatie van de aardbol wordt meegenomen door het raakpunt eerst via `worldToLocal()` naar de lokale ruimte van de bol te transformeren. Vervolgens wordt met een ray-casting point-in-polygon algoritme bepaald welk land de coördinaat bevat.

#### Configureerbare parameters (UI-panel)

| Parameter | Beschrijving |
|---|---|
| Auto-rotate | Start/stopt de rotatie |
| Dag/nacht verschil | Schakelt de nacht-textuur en PBR-belichting in of uit |
| Wolken | Schakelt de wolkenlaag in of uit |
| Dag kleur / Schemering kleur | Atmosfeerkleur aan dag- en schaduwzijde |
| Roughness laag / hoog | Remaps de ruwheid van oceanen en wolken |
| Outline kleur + dikte | Kleur en dikte van de hover-omtrek |
| Vul kleur | Vulkleur van het geselecteerde land |

---

## Data-integratie (uit te werken door student)

### 1 – Datakeuze

Kies een dataset die zinvol is per land, bijvoorbeeld:

- Bevolkingsdichtheid, BBP, CO₂-uitstoot (World Bank / Our World in Data)
- Toerismecijfers (UNWTO)
- Eigen verzamelde data (enquête, sensor, scraper)

### 2 – Database

Richt een relationele of document-gebaseerde database in om de data per land op te slaan.

```sql
-- Voorbeeld: relationeel schema
CREATE TABLE countries (
    iso_a3   CHAR(3) PRIMARY KEY,   -- ISO 3166-1 alpha-3 landcode
    name     VARCHAR(100)
);

CREATE TABLE indicators (
    iso_a3   CHAR(3) REFERENCES countries(iso_a3),
    year     INT,
    metric   VARCHAR(50),           -- bijv. 'gdp_usd', 'population'
    value    NUMERIC,
    PRIMARY KEY (iso_a3, year, metric)
);
```

Koppel de GeoJSON-features aan de database via de **ISO 3166-1 alpha-3** landcode (beschikbaar in `world-atlas@2` met de `countries-50m.json` of via Natural Earth).

### 3 – Grafiek / UI-element

Toon data bij een klik op een land in een overlay of side-panel. Mogelijke keuzes:

| Type | Library | Geschikt voor |
|---|---|---|
| Staafdiagram | Chart.js / D3.js | Vergelijking over jaren |
| Lijngrafiek | Chart.js / D3.js | Tijdreeksen |
| Gauge / KPI | D3.js | Enkelvoudige waarde t.o.v. max |
| Tabel | Eigen HTML | Gedetailleerde ruwe data |

### 4 – AJAX en caching

Data wordt asynchroon opgehaald zodat de 3D-rendering niet blokkeert.

```js
// Voorbeeld: data ophalen bij klik op een land
async function loadCountryData(isoA3) {
    const cacheKey = `country_data_${isoA3}`;

    // 1. Controleer sessionStorage-cache
    const cached = sessionStorage.getItem(cacheKey);
    if (cached) return JSON.parse(cached);

    // 2. Haal op via API
    const res  = await fetch(`/api/indicators/${isoA3}`);
    const data = await res.json();

    // 3. Sla op in cache (vervalt bij sluiten tabblad)
    sessionStorage.setItem(cacheKey, JSON.stringify(data));
    return data;
}
```

**Caching-overwegingen:**

| Methode | Scope | Vervaltijd | Geschikt voor |
|---|---|---|---|
| `sessionStorage` | tabblad | tabblad sluiten | enkelvoudige sessie |
| `localStorage` | browser | nooit (handmatig) | stabiele referentiedata |
| HTTP `Cache-Control` | netwerk | instelbaar | hergebruik bij herladen |
| Service Worker | browser | instelbaar | offline ondersteuning |

Voor veranderlijke data (live statistieken) is een korte `Cache-Control: max-age` op de API-response de meest robuuste aanpak. Voor statische referentiedata (landnamen, vaste statistieken) is `localStorage` met een timestamp-check afdoende.

---

## Lokaal draaien

Open `index.html` direct in een browser met WebGPU-ondersteuning (Chrome 113+, Edge 113+). Er is geen buildstap nodig — alle afhankelijkheden worden geladen via CDN.
