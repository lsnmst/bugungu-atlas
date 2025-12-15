<script>
  import L from "leaflet";
  import "leaflet-minimap";
  import LayerPanel from "./components/LayerPanel.svelte";
  import SearchPoints from "./components/SearchPoints.svelte";

  import { points } from "./data/points.js";
  import { toponyms } from "./data/toponyms.js";
  import { landmarks } from "./data/landmarks.js";
  import { t } from "svelte-i18n";
  import { onMount } from "svelte";

  let map;
  let geoJsonLayer;
  let toponymsLayer;
  let landmarksLayer;

  let showModal = true;
  let modalEl;
  let mapEl;

  let activeMarker = { id: null, type: null };

  let layerControls = [];

  const isMobile =
    window.innerWidth < 768 || /Mobi|Android/i.test(navigator.userAgent);

  let activeLayers = ["points", "toponyms", "landmarks"];

  // ----------------LEGEND ----------------
  const featuresWithLayer = [
    ...points.features.map((f) => ({ feature: f, layerId: "points" })),
    ...toponyms.features.map((f) => ({ feature: f, layerId: "toponyms" })),
    ...landmarks.features.map((f) => ({ feature: f, layerId: "landmarks" })),
  ];

  let legendMap = new Map();

  const Z_NORMAL = 0;
  const Z_SELECTED = 10000;

  featuresWithLayer.forEach(({ feature, layerId }) => {
    const props = feature.properties || {};
    const type = props.type || "unknown";
    const name = props.name || type;
    const svgHtml = props.svgHtml || "";

    if (!legendMap.has(type)) {
      legendMap.set(type, { type, name, svgHtml, layerId });
    } else {
      const existing = legendMap.get(type);
      if (
        activeLayers.includes(layerId) &&
        !activeLayers.includes(existing.layerId)
      ) {
        legendMap.set(type, { type, name, svgHtml, layerId });
      }
    }
  });

  let combinedLegend = Array.from(legendMap.values());

  const legendOrder = {
    landmarks: 1,
    points: 3,
    toponyms: 2,
  };

  combinedLegend = combinedLegend.sort(
    (a, b) => legendOrder[a.layerId] - legendOrder[b.layerId],
  );

  // --------------- ACTIVE LEGEND ----------------
  $: activeLegend = combinedLegend.filter((item) =>
    activeLayers.includes(item.layerId),
  );

  // ---------------- ICON CREATOR ----------------
  function createIcon(feature, zoom = 15, layerType = "point") {
    const { svgHtml, iconWidth, iconHeight, name, hr } = feature.properties;
    const baseZoom = 15;
    const scale = Math.pow(1.2, zoom - baseZoom);

    const w = iconWidth * scale;
    const h = iconHeight * scale;

    return L.divIcon({
      className: `custom-svg-icon ${layerType}-${feature.properties.ID}`,
      html: `
      <div style="display:flex;flex-direction:column;align-items:center;">
        ${
          zoom > 14 && !isMobile
            ? `<div class='marker-label' style="color:${hr || "#000"};-webkit-text-stroke-color:${hr || "#000"};">${name}</div>`
            : ""
        }
        ${svgHtml}
      </div>
    `,
      iconSize: [w, h],
      iconAnchor: [w / 2, h],
      popupAnchor: [0, -h / 2],
    });
  }

  // ---------------- POPUP BEHAVIOR ----------------
  function onEachFeature(feature, layer, layerType) {
    const noteContent = feature.properties.note
      ? `<p style='font-size:0.6em;margin:5px;padding-bottom:30px'>${feature.properties.note}</p>`
      : "";

    const popupContent = `
      <hr style='height:4px;background:${feature.properties.hr};border:none;'>
      <p style='font-size:0.6em;margin:5px'>${feature.properties.type}</p>
      <p style='margin:5px'><b>${feature.properties.name}</b></p>
      <hr>
      ${noteContent}
    `;

    layer.bindPopup(popupContent);

    layer.on("popupopen", () => {
      document
        .querySelectorAll(".custom-svg-icon.svg-active")
        .forEach((el) => el.classList.remove("svg-active"));

      layer.setZIndexOffset(Z_SELECTED);

      const selector = `.${layerType}-${feature.properties.ID}`;
      const icon = document.querySelector(selector);
      if (icon) icon.classList.add("svg-active");

      activeMarker = { id: feature.properties.ID, type: layerType };
    });

    layer.on("popupclose", () => {
      layer.setZIndexOffset(Z_NORMAL);

      const selector = `.${layerType}-${feature.properties.ID}`;
      const icon = document.querySelector(selector);
      if (icon) icon.classList.remove("svg-active");

      activeMarker = { id: null, type: null };
    });
  }

  // ---------------- ADD LAYERS ----------------
  function addGeoJson() {
    geoJsonLayer = L.geoJSON(points, {
      pointToLayer: (feature, latlng) =>
        L.marker(latlng, {
          icon: createIcon(feature, 15, "points"),
        }),
      onEachFeature: (feature, layer) =>
        onEachFeature(feature, layer, "points"),
    }).addTo(map);
  }

  function addToponymsLayer() {
    toponymsLayer = L.geoJSON(toponyms, {
      pointToLayer: (feature, latlng) =>
        L.marker(latlng, {
          icon: createIcon(feature, 15, "points"),
        }),
      onEachFeature: (feature, layer) =>
        onEachFeature(feature, layer, "toponyms"),
    }).addTo(map);
  }

  function addLandmarksLayer() {
    landmarksLayer = L.geoJSON(landmarks, {
      pointToLayer: (feature, latlng) =>
        L.marker(latlng, {
          icon: createIcon(feature, 15, "points"),
        }),
      onEachFeature: (feature, layer) =>
        onEachFeature(feature, layer, "landmarks"),
    }).addTo(map);
  }

  // ---------------- ZOOM ICON SCALING ----------------
  function setupZoomScaling() {
    map.on("zoomend", () => {
      const zoom = map.getZoom();

      geoJsonLayer.eachLayer((layer) =>
        layer.setIcon(createIcon(layer.feature, zoom, "points")),
      );
      toponymsLayer.eachLayer((layer) =>
        layer.setIcon(createIcon(layer.feature, zoom, "toponyms")),
      );
      landmarksLayer.eachLayer((layer) =>
        layer.setIcon(createIcon(layer.feature, zoom, "landmarks")),
      );

      // restore highlight
      if (activeMarker.id != null) {
        const selector = `.${activeMarker.type}-${activeMarker.id}`;
        const icon = document.querySelector(selector);
        if (icon) icon.classList.add("svg-active");
      }
    });
  }

  // ---------------- HANDLE SEARCH SELECT ----------------
  function handleSelectPoint(point) {
    if (!point || !point.coordinates) return;

    const [lng, lat] = point.coordinates;
    const target = L.latLng(lat, lng);
    const targetZoom = Math.max(14, Math.min(map.getMaxZoom(), 15));
    const targetID = Number(point.id);

    let targetLayer = null;

    function findLayer(group) {
      if (!group) return null;
      let found = null;
      group.eachLayer((l) => {
        if (l.feature?.properties?.ID == targetID) found = l;
      });
      return found;
    }

    if (point.layerType === "points") {
      targetLayer =
        findLayer(geoJsonLayer) ||
        findLayer(toponymsLayer) ||
        findLayer(landmarksLayer);
    } else if (point.layerType === "toponyms") {
      targetLayer =
        findLayer(toponymsLayer) ||
        findLayer(geoJsonLayer) ||
        findLayer(landmarksLayer);
    } else if (point.layerType === "landmarks") {
      targetLayer =
        findLayer(landmarksLayer) ||
        findLayer(geoJsonLayer) ||
        findLayer(toponymsLayer);
    }

    if (!targetLayer) return;

    targetLayer.bindPopup(targetLayer.getPopup().getContent(), {
      autoPan: false,
    });

    map.flyTo(target, targetZoom, { duration: 1 });

    map.once("moveend", () => {
      const offsetLat = lat + 0.0008;

      map.panTo([offsetLat, lng], { animate: true });

      map.once("moveend", () => {
        targetLayer.setZIndexOffset(Z_SELECTED);
        targetLayer.openPopup();
      });
    });
  }

  // ---------------- CLICK OUTSIDE MODAL ----------------
  function handleClickOutside(event) {
    if (showModal && modalEl && !modalEl.contains(event.target))
      showModal = false;
  }

  // ---------------- MOUNT MAP ----------------
  onMount(() => {
    document.addEventListener("click", handleClickOutside);

    const initialZoom = isMobile ? 12 : 13;

    map = L.map(mapEl, {
      center: [2.1591666907005296, 31.396436976538762],
      zoom: initialZoom,
      minZoom: 10,
      maxZoom: 15,
      maxBounds: [
        [1.58, 31.15],
        [2.41, 31.8],
      ],
      maxBoundsViscosity: 0.8,
      zoomControl: false,
    });

    L.tileLayer(
      "https://www.alessandromusetta.com/geo/tiles/bugungu/{z}/{x}/{y}.png",
      { minZoom: 10, maxZoom: 15 },
    ).addTo(map);

    const mini = new L.Control.MiniMap(
      L.tileLayer(
        "https://www.alessandromusetta.com/geo/tiles/minimapbugungu/{z}/{x}/{y}.png",
        { minZoom: 8, maxZoom: 11 },
      ),
      { toggleDisplay: true, position: "topright", width: 150, height: 150 },
    ).addTo(map);

    L.control.scale({ maxWidth: 150 }).addTo(map);

    addGeoJson();
    addToponymsLayer();
    addLandmarksLayer();
    setupZoomScaling();

    layerControls = [
      {
        id: "landmarks",
        name: "Ecocultural and spiritual tourism",
        description: "Description 'Ecocultural and spiritual tourism' layer",
        visible: true,
        layer: landmarksLayer,
        toggle: (v) => {
          if (v) {
            map.addLayer(landmarksLayer);
            if (!activeLayers.includes("landmarks"))
              activeLayers = [...activeLayers, "landmarks"];
          } else {
            map.removeLayer(landmarksLayer);
            activeLayers = activeLayers.filter((id) => id !== "landmarks");
          }
        },
      },
      {
        id: "toponyms",
        name: "Fixing toponyms, and preserve vanishing toponyms",
        description:
          "Giving names to places means securing ancestral lineage, reviving memory and passing on Bugungu culture.",
        visible: true,
        layer: toponymsLayer,
        toggle: (v) => {
          if (v) {
            map.addLayer(toponymsLayer);
            if (!activeLayers.includes("toponyms"))
              activeLayers = [...activeLayers, "toponyms"];
          } else {
            map.removeLayer(toponymsLayer);
            activeLayers = activeLayers.filter((id) => id !== "toponyms");
          }
        },
      },
      {
        id: "points",
        name: "Bugungu Heritage and Information Centre",
        description: "",
        visible: true,
        layer: geoJsonLayer,
        toggle: (v) => {
          if (v) {
            map.addLayer(geoJsonLayer);
            if (!activeLayers.includes("points"))
              activeLayers = [...activeLayers, "points"];
          } else {
            map.removeLayer(geoJsonLayer);
            activeLayers = activeLayers.filter((id) => id !== "points");
          }
        },
      },
    ];

    return () => document.removeEventListener("click", handleClickOutside);
  });
</script>

<!-- TOP/BOTTOM UI -->
<div class="bottom-bar">
  <div class="bottom-bar-content">
    <img src="./BHIC.png" alt="Logo" class="bottom-bar-logo" />
    <div class="bottom-bar-title">
      <span>{@html $t("app.title")}</span><br />
      <span class="title_" style="font-size:0.9rem;"
        >{@html $t("app.title_")}</span
      >
    </div>
  </div>
</div>

<div class="left-bar"></div>
<div class="top-bar"></div>
<div class="right-bar"></div>

<!-- MAP -->
<div bind:this={mapEl} class="map-container">
  {#if showModal}
    <div class="modal-backdrop">
      <div bind:this={modalEl} class="modal-bottom-right">
        <button class="modal-close" on:click={() => (showModal = false)}
          >×</button
        >
        <div class="modal-content">
          <h3>{$t("app.modalTitle")}</h3>
          <p>{@html $t("app.modalBody")}</p>
        </div>
      </div>
    </div>
  {/if}
</div>

<!-- SEARCH -->
<SearchPoints onSelect={handleSelectPoint} {activeLayers} />

<!-- LAYER PANEL -->
<LayerPanel
  layers={layerControls}
  legend={activeLegend}
  on:toggle={(e) => toggleLayer(e.detail.id, e.detail.visible)}
/>

<style>
  .map-container {
    height: 100vh;
    width: 100vw;
    position: absolute;
    inset: 0;
    background: white;
  }

  .custom-svg-icon {
    transition: 0.2s;
    pointer-events: auto;
  }
  .svg-active {
    transform: scale(1.3);
    filter: drop-shadow(0 0 5px rgba(0, 0, 0, 0.4));
  }

  .marker-label {
    font-size: 0.7em;
    background: white;
    padding: 2px 5px;
    border-radius: 4px;
    margin-bottom: 2px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
  }
</style>
