# GIWU Dataset — Project Context

## Overview
Interactive web map of the GIWU dataset hosted on GitHub Pages, built from QGIS exports via qgis2web.

---

## 1. GitHub Pages — Interactive GIWU Web Map

### Repository
- **Repo:** `francescoleopardi02-ui/GIWU_dataset`
- **URL:** `https://francescoleopardi02-ui.github.io/GIWU_dataset/`
- Files uploaded via **GitHub Desktop** (web upload has 100-file limit)

### Key modifications to index.html (exported from qgis2web)

#### 1.1 Basemap
qgis2web cannot export Bing Virtual Earth (requires API key). The basemap is added manually after `var hash = new L.Hash(map);`:

```javascript
L.tileLayer('https://server.arcgisonline.com/ArcGIS/rest/services/Canvas/World_Light_Gray_Base/MapServer/tile/{z}/{y}/{x}', {attribution: '© Esri'}).addTo(map);
```

Alternative basemaps used previously:
- ESRI World Imagery: `.../World_Imagery/MapServer/tile/{z}/{y}/{x}`

#### 1.2 Remove Bing Satellite layer
qgis2web exports a Bing layer that causes `{q}` variable errors in Leaflet. Must remove entirely:

```javascript
// REMOVE THIS ENTIRE BLOCK:
map.createPane('pane_BingSatellite_0');
map.getPane('pane_BingSatellite_0').style.zIndex = 400;
var layer_BingSatellite_0 = L.tileLayer('https://ecn.t3.tiles.virtualearth.net/tiles/a{q}.jpeg?g=0&dir=dir_n', {
    pane: 'pane_BingSatellite_0',
    opacity: 1.0,
    attribution: '',
    minZoom: 1,
    maxZoom: 28,
    minNativeZoom: 1,
    maxNativeZoom: 18
});
layer_BingSatellite_0;
map.addLayer(layer_BingSatellite_0);
```

Also had to fix a double-quote syntax error in the Bing URL (`'...dir_n''` → `'...dir_n'`).

**Note:** if the basemap is left unchecked in the qgis2web dialog, no Bing block is exported at all and this step can be skipped. Check with `grep -c BingSatellite index.html` — if it returns 0, there is nothing to remove.

#### 1.3 Remove old Tree Control references
Remove these two lines from `<head>`:
```html
<link rel="stylesheet" href="css/L.Control.Layers.Tree.css">
<script src="js/L.Control.Layers.Tree.min.js"></script>
```

#### 1.4 Layer Control
308 layers organized into 11 groups. Code inserted before `setBounds();`:

```javascript
var overlaysTree = [
    {
        label: "Spain",
        selectAllCheckbox: true,
        children: [
        {label: "URGELL SISTEMA DEF DISSOLTO", layer: layer_URGELL_SISTEMA_DEF_DISSOLTO_1},
        // ... more layers
        ]
    },
    // Portugal, Italy, Greece, Belgium, Germany, France, CONUS, South Africa, Australia, India
];

var overlays = {};
overlaysTree.forEach(function(group) {
    if (group.label === "Greece") return;  // Greece handled separately
    var groupLayer = L.layerGroup();
    group.children.forEach(function(child) {
        groupLayer.addLayer(child.layer);
    });
    overlays["<b>" + group.label + "</b>"] = groupLayer;
    groupLayer.addTo(map);
});
overlays["<b>Greece</b>"] = layer_Sectors_Fields_S09_S10_177;   // suffix changes on every re-export
L.control.layers(null, overlays, {collapsed: false}).addTo(map);
```

The `_N` suffixes in this snippet are illustrative — they shift on every re-export (see the traps in the update workflow below).

**Greece** (`Sectors_Fields_S09_S10`) is added as a direct layer rather than through the forEach LayerGroup. The previous note here claimed a LayerGroup "silently fails for single-layer groups"; that is not accurate — a single-layer `L.layerGroup` renders its label, is added to the map, and returns valid bounds when tested against Leaflet 1.9.4. The real cause of the original breakage was most likely the zoom-button bugs described in §1.8. The special case is kept because it works and India (also a single-layer group) goes through the normal path without trouble; if you ever need to unify them, test the Greece entry's checkbox and magnifier in a real browser first.

#### 1.5 Layer categorization rules
```
Spain: URGELL, ALGERRI, NORTH_CATALAN, PINYANA, SOUTH_CATALAN
Portugal: starts with PT_
Italy: AltoTevere, Area_1, Area_2, Area_test, Astrone, Brunel, Budrio, Clitunno, Faenza, Fossalto, Marroggia, Nocciolo, Passignano, Pietro, San_Michele, Sersimone, Sferracavallo, Soia_Zera, Topino, zonaA, zone_pivot, cfr, cirio, colza, distretti, metano, prova_irrigazione, rovere
Greece: Sectors_Fields
Belgium: Aalst, Asse, Bocholt, Boortmeerbeek, Bornem, Bree, Duffel, Evergem, Houthalen, Ieper, Kampenhout, Kinrooi, Kruisem, Lier, LoReninge, Londerzeel, Maaseik, Mechelen, Meise, Opwijk, Oudsbergen, Peer, Puurs, Rumst, SintAmands, SintMartens, Staden, Tremelo, Zedelgem
Germany: brandenburg, niedersachsen
France: Lot_irrigation, Tarn_plots
CONUS: State_Name
South Africa: TID_, Hexvalley
Australia: Coleambally, Murray_Irrigation, Murrumbidgee
India: Sina
```

#### 1.6 Initial view
qgis2web exports `fitBounds` centered on the last viewed area. Replace with global view:

```javascript
// Replace:
}).fitBounds([[39.71..., 22.73...], [39.72..., 22.74...]]);
// With:
}).setView([30, 0], 3);
```

To auto-zoom to all layers, add inside `setBounds()`:
```javascript
function setBounds() {
    if (bounds_group.getLayers().length > 0) { map.fitBounds(bounds_group.getBounds()); }
}
```

#### 1.7 CSS for Layer Control
```css
.leaflet-control-layers {
    max-height: 80vh;
    overflow-y: auto;
    font-size: 13px;
    padding: 8px 12px;
    background: rgba(255,255,255,0.95);
    border-radius: 6px;
    box-shadow: 0 2px 6px rgba(0,0,0,0.3);
}
.leaflet-control-layers-overlays label {
    display: block;
    margin: 3px 0;
}
```

#### 1.8 Zoom-to-layer buttons
Magnifying glass icon next to each group name, zooms to that group's extent:

```javascript
var overlaysList = document.querySelector('.leaflet-control-layers-overlays');

function boundsForOverlay(name) {
    for (var key in overlays) {
        if (key.replace(/<\/?b>/g, '') !== name) continue;
        var layer = overlays[key];
        if (layer.getBounds) return layer.getBounds();
        if (layer.getLayers && layer.getLayers().length > 0) {
            return L.featureGroup(layer.getLayers()).getBounds();
        }
        return null;
    }
    return null;
}

// the name lives in its own span, so it stays readable after the button is appended
function overlayNameOf(label) {
    var holder = label.querySelector('span');
    var nameSpan = holder && holder.querySelector('span');
    return nameSpan ? nameSpan.textContent.trim() : '';
}

function addZoomButtons() {
    overlaysList.querySelectorAll('label').forEach(function(label) {
        var holder = label.querySelector('span');
        if (!holder || holder.querySelector('.zoom-to-layer')) return;
        var btn = document.createElement('span');
        btn.className = 'zoom-to-layer';
        btn.innerHTML = ' &#128269;';
        btn.style.cursor = 'pointer';
        btn.style.fontSize = '14px';
        btn.title = 'Zoom to layer';
        holder.appendChild(btn);
    });
}

// delegated, so it survives the control rebuilding its list on every layer change
overlaysList.addEventListener('click', function(e) {
    var btn = e.target.closest('.zoom-to-layer');
    if (!btn) return;
    e.preventDefault();
    e.stopPropagation();
    var bounds = boundsForOverlay(overlayNameOf(btn.closest('label')));
    if (bounds && bounds.isValid()) { map.fitBounds(bounds); }
});

new MutationObserver(addZoomButtons).observe(overlaysList, {childList: true, subtree: true});
addZoomButtons();
```

**Do not "simplify" this back to a `setTimeout` that attaches one `onclick` per icon.** That earlier version never worked, for two reasons — both verified against real Leaflet 1.9.4:

1. **Leaflet's markup is `label > span(holder) > [input, span(name)]`.** `label.querySelector('span')` returns the *holder*, not the name. Appending the icon to the holder then reading `holder.textContent` at click time yields `"Spain 🔍"`, which never matches the `overlays` key `"Spain"`, so the loop falls through silently. Hence reading the name from the inner name span, which never contains the icon.
2. **`L.Control.Layers._update()` rebuilds the entire overlays list from scratch** whenever a layer is added to or removed from the map outside the control (`map.addLayer` / `map.removeLayer`). Icons attached once to the original `<label>` elements are destroyed along with their handlers. Toggling a checkbox *inside* the control does not rebuild (`_handlingClick` suppresses it), which is why the breakage looked intermittent. Hence the delegated listener plus the `MutationObserver`.

`bounds.isValid()` is checked because `fitBounds` throws `"Bounds are not valid"` on an empty group.

**Testing caveat — cache will lie to you.** GitHub Pages serves HTML with `Cache-Control: max-age=600`, and `npx http-server` defaults to `max-age=3600`. A stale page will show the old behaviour even after a correct deploy. Use `npx http-server -p 8765 -c-1` locally (the `-c-1` disables caching) and hard-refresh with Ctrl+F5. Confirm which version you are actually running:

```javascript
// in the browser console
[...document.scripts].find(s => !s.src).textContent.includes('MutationObserver')
```

### Update workflow (when adding new layers)
1. Export from qgis2web — **Leaflet**, not OpenLayers (every manual modification here uses the Leaflet API), basemap unchecked. Verify *all* existing layers are still ticked in the dialog, not just the new ones.
2. Delete the contents of `css/`, `data/`, `js/`, `legend/`, `markers/`, `webfonts/` in the repo folder — but keep `CLAUDE.md`.
3. Copy all new export content into repo folder.
4. Modify `index.html`: add basemap, remove Bing (if present), remove Tree refs, add CSS, rebuild layer control, fix initial view, add zoom buttons.
5. Verify locally before pushing: `npx http-server -p 8765 -c-1`, then open `http://localhost:8765/index.html`.
6. GitHub Desktop → Commit → Push (~62 MB, takes a few minutes).
7. Wait 2-3 min, test in incognito (Ctrl+Shift+N).

#### Two traps in this workflow

**qgis2web renumbers every layer on each export.** The `_N` suffix is positional, so inserting one new layer shifts the numbering of everything after it: `distretti84_151.js` becomes `distretti84_150.js`, and so on. Consequences:

- Step 2 is not optional. Skipping it leaves the old files behind under their old names next to the new ones — we once ended up with 486 files instead of 308, all orphaned duplicates.
- Every `layer_*` reference in the hand-written layer control breaks on each re-export. Do not hand-edit the numbers. Instead, match old to new by *base name* (strip the `layer_` prefix and the trailing `_N`), carry the labels and grouping over from the previous committed `index.html` via `git show HEAD:index.html`, and regenerate the whole `overlaysTree`.

**Verify the layer set after every export**, before touching `index.html` — a layer accidentally unticked in QGIS disappears silently:

```bash
git ls-tree -r HEAD --name-only -- data | sed -E 's#data/(.*)_[0-9]+\.js#\1#' | sort > /tmp/old.txt
ls data/*.js | sed -E 's#data/(.*)_[0-9]+\.js#\1#' | sort > /tmp/new.txt
diff /tmp/old.txt /tmp/new.txt   # expect only the intended additions
```

Then sanity-check the counts — these three must agree:

```bash
ls data/*.js | wc -l              # data files
grep -c '<script src="data/' index.html
grep -c '{label:' index.html      # entries in the layer control
```

#### Notes
- GitHub Pages serves from `main`; there is no `gh-pages` branch and no build step, so a push to `main` is the deploy.
- `data/distretti84_*.js` is ~47 MB. Under GitHub's 100 MB per-file limit, but it dominates push time.
- Checking JS syntax without a browser:
  ```bash
  node -e "const h=require('fs').readFileSync('index.html','utf8');[...h.matchAll(/<script>([\s\S]*?)<\/script>/g)].forEach(m=>{try{new Function(m[1])}catch(e){console.log(e.message)}})"
  ```
