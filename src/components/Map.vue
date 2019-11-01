<template>
  <div id="container">
    <div id="map" class="map"></div>
    <div id="below">
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

      <div id="mouse-position"></div>
      <form>
        <label>Projection</label>
        <select id="projection">
          <option value="EPSG:4326">EPSG:4326</option>
          <option value="EPSG:3857">EPSG:3857</option>
        </select>
        <label>Precision</label>
        <input id="precision" type="number" min="0" max="12" value="4" />
      </form>

      <div>
        <label>
          Latitude
          <input type="text" v-model="lat" />
        </label>
        <label>
          Longitude
          <input type="text" v-model="lng" />
        </label>
        <label>
          Zoom
          <input type="text" v-model="zoom" />
        </label>
        <button v-on:click="go()">Перейти</button>
      </div>
    </div>
  </div>
</template>

<style lang="scss">
@import "~ol/ol.css";
#map {
  width: 100%;
  height: calc(100% - 100px);
}

#below {
  width: 100%;
  height: 100px;
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
import { defaults as defaultControls, Control, ScaleLine } from "ol/control";
import {
  defaults as defaultInteractions,
  Modify,
  Select
} from "ol/interaction";
import * as ol from "ol";
import * as olProj from "ol/proj";
import MousePosition from "ol/control/MousePosition";
import { createStringXY } from "ol/coordinate";
import { eventBus } from "../main.js";
import WFS from "ol/format/WFS";

import VectorLayer from "ol/layer/Vector";
import VectorSource from "ol/source/Vector";
import GeoJSON from "ol/format/GeoJSON";
import GML from "ol/format/GML";
import { bbox as bboxStrategy } from "ol/loadingstrategy";
import { Stroke, Style } from "ol/style";

import $ from "jquery";

export default {
  data() {
    return {
      map: null,
      view: null,
      lng: 55.96779,
      lat: 54.74306,
      zoom: 12,
      str: null,
      namespace: "gis_example",
      vectorTileName: "landuse",
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
    go() {
      const place = olProj.transform(
        [this.lng, this.lat],
        "EPSG:4326",
        "EPSG:3857"
      );
      this.view.animate({
        center: place,
        zoom: this.zoom,
        duration: 1000
      });
    },

    removeLayers() {
      var layersToRemove = [];
      this.map.getLayers().forEach(layer => {
        if (layer.Name !== "Main" && layer.Name !== "Vector") {
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
          'All maps Â© <a href="https://www.opencyclemap.org/">OpenCycleMap</a>',
          ATTRIBUTION
        ],
        url:
          "https://{a-c}.tile.thunderforest.com/cycle/{z}/{x}/{y}.png" +
          "?apikey=dc8a9ee9b94a4856872e2fb768b61ef6"
      })
    });

    osm.Name = "Main";

    var view = new ol.View({
      center: [0, 0],
      zoom: 2
    });

    var mousePositionControl = new MousePosition({
      coordinateFormat: createStringXY(4),
      projection: "EPSG:4326",
      // comment the following two lines to have the mouse position
      // be placed within the map.
      className: "custom-mouse-position",
      target: document.getElementById("mouse-position"),
      undefinedHTML: "&nbsp;"
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

    var select = new Select({
      wrapX: false
    });

    var modify = new Modify({
      features: select.getFeatures()
    });

    var vectorSource = new VectorSource({
      format: new GeoJSON(),
      url: extent => {
        return (
          "http://localhost:8080/geoserver/wfs?service=WFS&" +
          "request=GetFeature&typename=" +
          this.vectorTileName +
          "&" +
          "outputFormat=application/json&srsname=EPSG:3857&" +
          "bbox=" +
          extent.join(",") +
          ",EPSG:3857"
        );
      },
      strategy: bboxStrategy
    });

    var vector = new VectorLayer({
      source: vectorSource,
      style: new Style({
        stroke: new Stroke({
          color: "rgba(0, 0, 255, 1.0)",
          width: 2
        })
      })
    });

    vector.Name = "Vector";

    var map = new Map({
      interactions: defaultInteractions({
        doubleClickZoom: true,
        dragAndDrop: true,
        dragPan: true,
        keyboardPan: true,
        keyboardZoom: true,
        mouseWheelZoom: true,
        pointer: true,
        select: true
      }).extend([select, modify]),
      controls: defaultControls().extend([
        scaleControl(),
        new RotateNorthControl(),
        mousePositionControl
      ]),
      layers: [osm, vector],
      target: "map",
      view
    });

    modify.on("modifyend", evt => {
      console.log(evt);
      var target = evt.features.getArray();
      console.log(target);

      var wfst = new WFS();
      var gml = new GML({
        featureNS: "http://localhost:8080/geoserver/gis_example",
        featureType: this.vectorTileName,
        srsName: "EPSG:3857"
      });
      var node = wfst.writeTransaction(null, target, null, gml);
      var str = new XMLSerializer().serializeToString(node);
      str = str
        .replace(
          "feature:" + this.vectorTileName,
          this.namespace + ":" + this.vectorTileName
        )
        .replace("geometry", "geom");
      console.log(str);

      this.str = str;

      $.ajax({
        url: "http://localhost:8080/geoserver/gis_example/wfs",
        data: str,
        service: "WFS",
        type: "POST",
        dataType: "xml",
        processData: false,
        contentType: "text/xml",
        success: function(data) {
          var result = wfst.readTransactionResponse(data);
          console.log(result);

          const tile = new TileLayer({
            source: new TileWMS({
              url: "http://localhost:8080/geoserver/gis_example/wms",
              params: {
                LAYERS: "gis_example:" + this.vectorTileName,
                TILED: true
              },
              serverType: "geoserver",
              projection: "EPSG:4326",
              transition: 0
            })
          });
          tile.Name = this.vectorTileName;
          const index = this.targetLayers.findIndex(
            e => e.Name === this.vectorTileName
          );
          this.targetLayers[index] = tile;
          this.targetLayers = this.targetLayers.slice();

          eventBus.$emit("layer-updated", this.vectorTileName);
        },
        error: function(e) {
          var errorMsg = e ? e.status + " " + e.statusText : "";
          console.log(
            "Error saving this feature to GeoServer.<br><br>" + errorMsg
          );
        },
        context: this
      });
    });

    this.map = map;
    this.view = view;

    var projectionSelect = document.getElementById("projection");
    projectionSelect.addEventListener("change", function(event) {
      mousePositionControl.setProjection(event.target.value);
    });

    var precisionInput = document.getElementById("precision");
    precisionInput.addEventListener("change", function(event) {
      var format = createStringXY(event.target.valueAsNumber);
      mousePositionControl.setCoordinateFormat(format);
    });

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

    this.go();
  }
};
</script>