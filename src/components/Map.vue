<template>
  <div id="container">
    <div id="map" class="map"></div>
    <select id="units">
      <option value="degrees">degrees</option>
      <option value="imperial">imperial inch</option>
      <option value="us">us inch</option>
      <option value="nautical">nautical mile</option>
      <option value="metric" selected>metric</option>
    </select>

    <select id="type">
      <option value="scaleline">ScaleLine</option>
      <option value="scalebar">ScaleBar</option>
    </select>

    <select id="steps" style="display:none">
      <option value="2">2 steps</option>
      <option value="4" selected>4 steps</option>
      <option value="6">6 steps</option>
      <option value="8">8 steps</option>
    </select>

    <div id="showScaleTextDiv" style="display:none">
      <input type="checkbox" id="showScaleText" checked />Show scale text
    </div>
  </div>
</template>

<style lang="scss">
@import "~ol/ol.css";
#map {
  width: 100%;
  height: 95%;
}
#container {
  width: 100%;
  height: 100%;
}

.rotate-north {
  top: 65px;
  left: 0.5em;
}
.ol-touch .rotate-north {
  top: 80px;
}
</style>

<script>
import Map from "ol/Map";
import { Tile as TileLayer } from "ol/layer";
import TileWMS from "ol/source/TileWMS";
import OSM, { ATTRIBUTION } from "ol/source/OSM";
import sync from "ol-hashed";
import { defaults as defaultControls, Control, ScaleLine } from "ol/control";
import { defaults } from "ol/interaction";
import * as ol from "ol";
import * as olProj from "ol/proj";

import { eventBus } from "../main.js";

export default {
  data() {
    return {
      map: null,
      targetLayers: []
    };
  },

  created() {
    eventBus.$on("layers-submitted", layers => {
      console.log("layers-submitted", layers);
      this.downloadLayers(layers);
    });
  },

  methods: {
    removeLayers() {
      var layersToRemove = [];
      this.map.getLayers().forEach(function(layer) {
        if (layer.Name !== "Main") {
          layersToRemove.push(layer);
        }
      });
      console.log("layersToRemove", layersToRemove);
      var len = layersToRemove.length;
      for (var i = 0; i < len; i++) {
        this.map.removeLayer(layersToRemove[i]);
      }
    },

    downloadLayers(layers) {
      if (this.targetLayers.length === 0) {
        for (var i = 0; i < layers.length; i++) {
          const tile = new TileLayer({
            source: new TileWMS({
              url: "http://localhost:8080/geoserver/gis_example/wms",
              params: { LAYERS: "gis_example:" + layers[i].Name, TILED: true },
              serverType: "geoserver",
              projection: "EPSG:4326",
              transition: 0
            })
          });
          tile.Name = layers[i].Name;
          this.targetLayers.push(tile);
        }
        return;
      }
      this.removeLayers();
      for (i = 0; i < layers.length; i++) {
        const tl = this.targetLayers.find(e => e.Name === layers[i].Name);
        if (tl) {
          this.map.addLayer(tl);
        }
      }

      console.log(this.map);
    }
  },

  mounted: function() {
    var unitsSelect = document.getElementById("units");
    var typeSelect = document.getElementById("type");
    var stepsSelect = document.getElementById("steps");
    var scaleTextCheckbox = document.getElementById("showScaleText");
    var showScaleTextDiv = document.getElementById("showScaleTextDiv");

    var scaleType = "scaleline";
    var scaleBarSteps = 4;
    var scaleBarText = true;
    var control;

    function scaleControl() {
      if (scaleType === "scaleline") {
        control = new ScaleLine({
          units: unitsSelect.value
        });
        return control;
      }
      control = new ScaleLine({
        units: unitsSelect.value,
        bar: true,
        steps: scaleBarSteps,
        text: scaleBarText,
        minWidth: 140
      });
      return control;
    }

    var osm = new TileLayer({
      source: new OSM({
        attributions: [
          'All maps © <a href="https://www.opencyclemap.org/">OpenCycleMap</a>',
          ATTRIBUTION
        ],
        url:
          "https://{a-c}.tile.thunderforest.com/cycle/{z}/{x}/{y}.png" +
          "?apikey=dc8a9ee9b94a4856872e2fb768b61ef6"
      })
    });

    osm.Name = "Main";

    var london = olProj.transform(
      [20.509207, 54.715424],
      "EPSG:4326",
      "EPSG:3857"
    );

    var view = new ol.View({
      center: london,
      zoom: 8
    });

    var RotateNorthControl = /*@__PURE__*/ (function(Control) {
      function RotateNorthControl(opt_options) {
        var options = opt_options || {};

        var button = document.createElement("button");
        button.innerHTML = "N";

        var element = document.createElement("div");
        element.className = "rotate-north ol-unselectable ol-control";
        element.appendChild(button);

        Control.call(this, {
          element: element,
          target: options.target
        });

        button.addEventListener(
          "click",
          this.handleRotateNorth.bind(this),
          false
        );
      }

      if (Control) RotateNorthControl.__proto__ = Control;
      RotateNorthControl.prototype = Object.create(
        Control && Control.prototype
      );
      RotateNorthControl.prototype.constructor = RotateNorthControl;

      RotateNorthControl.prototype.handleRotateNorth = function handleRotateNorth() {
        this.getMap()
          .getView()
          .setRotation(0);
      };

      return RotateNorthControl;
    })(Control);

    var map = new Map({
      interactions: defaults({
        doubleClickZoom: true,
        dragAndDrop: true,
        dragPan: true,
        keyboardPan: true,
        keyboardZoom: true,
        mouseWheelZoom: true,
        pointer: true,
        select: true
      }),
      controls: defaultControls().extend([
        scaleControl(),
        new RotateNorthControl()
      ]),
      layers: [osm],
      target: "map",
      view
    });

    sync(map);

    this.map = map;

    function onChange() {
      control.setUnits(unitsSelect.value);
    }
    function onChangeType() {
      scaleType = typeSelect.value;
      if (typeSelect.value === "scalebar") {
        stepsSelect.style.display = "inline";
        showScaleTextDiv.style.display = "inline";
        map.removeControl(control);
        map.addControl(scaleControl());
      } else {
        stepsSelect.style.display = "none";
        showScaleTextDiv.style.display = "none";
        map.removeControl(control);
        map.addControl(scaleControl());
      }
    }
    function onChangeSteps() {
      scaleBarSteps = parseInt(stepsSelect.value, 10);
      map.removeControl(control);
      map.addControl(scaleControl());
    }
    function onChangeScaleText() {
      scaleBarText = scaleTextCheckbox.checked;
      map.removeControl(control);
      map.addControl(scaleControl());
    }
    unitsSelect.addEventListener("change", onChange);
    typeSelect.addEventListener("change", onChangeType);
    stepsSelect.addEventListener("change", onChangeSteps);
    scaleTextCheckbox.addEventListener("change", onChangeScaleText);
  }
};
</script>