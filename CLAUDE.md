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

#### 1.3 Remove old Tree Control references
Remove these two lines from `<head>`:
```html
<link rel="stylesheet" href="css/L.Control.Layers.Tree.css">
<script src="js/L.Control.Layers.Tree.min.js"></script>
```

#### 1.4 Layer Control
306 layers organized into 11 groups. Code inserted before `setBounds();`:

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
overlays["<b>Greece</b>"] = layer_Sectors_Fields_S09_S10_176;
L.control.layers(null, overlays, {collapsed: false}).addTo(map);
```

**Important:** Greece (`Sectors_Fields_S09_S10`) must be added as a direct layer, NOT through the forEach LayerGroup — the LayerGroup silently fails for single-layer groups.

#### 1.5 Layer categorization rules
```
Spain: URGELL, ALGERRI, NORTH_CATALAN, PINYANA, SOUTH_CATALAN
Portugal: starts with PT_
Italy: AltoTevere, Astrone, Brunel, Budrio, Clitunno, Faenza, Fossalto, Marroggia, Nocciolo, Passignano, Pietro, San_Michele, Sersimone, Sferracavallo, Soia_Zera, Topino, zonaA, zone_pivot, cfr, cirio, colza, distretti, metano, prova_irrigazione, rovere, Area_test
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
Added magnifying glass icon next to each layer name:

```javascript
setTimeout(function() {
    var labels = document.querySelectorAll('.leaflet-control-layers-overlays label');
    labels.forEach(function(label) {
        var span = label.querySelector('span');
        if (span) {
            var btn = document.createElement('span');
            btn.innerHTML = ' &#128269;';
            btn.style.cursor = 'pointer';
            btn.style.fontSize = '14px';
            btn.title = 'Zoom to layer';
            btn.onclick = function(e) {
                e.preventDefault();
                e.stopPropagation();
                var layerName = span.textContent.trim();
                for (var key in overlays) {
                    var cleanKey = key.replace(/<\/?b>/g, '');
                    if (cleanKey === layerName) {
                        var layer = overlays[key];
                        if (layer.getBounds) {
                            map.fitBounds(layer.getBounds());
                        } else if (layer.getLayers && layer.getLayers().length > 0) {
                            var group = L.featureGroup(layer.getLayers());
                            map.fitBounds(group.getBounds());
                        }
                        break;
                    }
                }
            };
            span.appendChild(btn);
        }
    });
}, 1000);
```

**Note:** This does NOT work when testing locally (`file://`) — only on GitHub Pages.

### Update workflow (when adding new layers)
1. Export from qgis2web (basemap unchecked)
2. Delete all content in local repo folder
3. Copy all new export content into repo folder
4. Modify `index.html`: add basemap, remove Bing, remove Tree refs, add layer control with new groups
5. GitHub Desktop → Commit → Push
6. Wait 2-3 min, test in incognito (Ctrl+Shift+N)
