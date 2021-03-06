<template>

  <v-card v-draggable-win class="wgu-measurewin" v-if=show v-bind:style="{ left: left, top: top }">
    <v-toolbar class="red darken-3 white--text" dark>
      <v-toolbar-side-icon><v-icon>{{icon}}</v-icon></v-toolbar-side-icon>
      <v-toolbar-title>Measure</v-toolbar-title>
      <v-spacer></v-spacer>
      <v-toolbar-side-icon @click="show = false"><v-icon>close</v-icon></v-toolbar-side-icon>
    </v-toolbar>

    <v-card-title primary-title>
      <div class="">
        <v-btn-toggle v-model="measureType" mandatory>
           <v-btn large value="distance">
             Distance
           </v-btn>
           <v-btn large value="area">
             Area
           </v-btn>
         </v-btn-toggle>
      </div>

    </v-card-title>

    <v-card-actions>
      <div class="">
        <div class="measure-result">
          LENGTH: {{distance}}
        </div>
        <div class="measure-result">
          AREA: {{area}}
        </div>
      </div>

    </v-card-actions>
  </v-card>

</template>

<script>
  import { DraggableWin } from '../../directives/DraggableWin';
  import { Mapable } from '../../mixins/Mapable';
  import DrawInteraction from 'ol/interaction/draw'
  import LineStringGeom from 'ol/geom/linestring'
  import PolygonGeom from 'ol/geom/polygon'
  import Sphere from 'ol/sphere'
  import Observable from 'ol/observable'
  import VectorSource from 'ol/source/vector'
  import VectorLayer from 'ol/layer/vector'
  import Style from 'ol/style/style'
  import Stroke from 'ol/style/stroke'
  import Circle from 'ol/style/circle'
  import Fill from 'ol/style/fill'

  export default {
    directives: {
      DraggableWin
    },
    mixins: [Mapable],
    props: ['icon'],
    data () {
      return {
        area: ' -- ',
        distance: ' -- ',
        measureType: 'distance',
        show: false,
        left: '100px',
        top: '200px'
      }
    },
    watch: {
      show () {
        var me = this;
        if (me.show === true) {
          me.addInteraction();
        } else {
          me.removeInteraction();
        }
      },
      // listen to changed measurement type
      measureType (newVal, oldVal) {
        this.addInteraction();
      }
    },
    methods: {
      /**
       * This function is executed, after the map is bound (see mixins/Mapable)
       */
      onMapBound () {
        this.createMeasureLayer();
      },
      /**
       * Creates a vector layer for the measurement results and adds it to the
       * map.
       */
      createMeasureLayer () {
        const me = this;
        const measureConf = me.$appConfig.modules.wgumeasure || {};
        // create a vector layer to
        var source = new VectorSource();
        var vector = new VectorLayer({
          name: 'Measure Layer',
          source: source,
          style: new Style({
            fill: new Fill({
              color: measureConf.fillColor || 'rgba(255, 255, 255, 0.2)'
            }),
            stroke: new Stroke({
              color: measureConf.strokeColor || 'rgba(0, 0, 0, 0.5)',
              width: 2
            })
          })
        });

        me.map.addLayer(vector);

        // make vector source available as member
        me.source = source;
      },
      /**
       * Creates and adds the necessary draw interaction and adds it to the map.
       */
      addInteraction () {
        const me = this;
        const measureConf = me.$appConfig.modules.wgumeasure || {};
        // cleanup possible old draw interaction
        if (me.draw) {
          me.removeInteraction();
        }

        var type = (this.measureType === 'area' ? 'Polygon' : 'LineString');
        var draw = new DrawInteraction({
          source: me.source,
          type: type,
          style: new Style({
            fill: new Fill({
              color: measureConf.sketchFillColor || 'rgba(255, 255, 255, 0.2)'
            }),
            stroke: new Stroke({
              color: measureConf.sketchStrokeColor || 'rgba(0, 0, 0, 0.5)',
              lineDash: [10, 10],
              width: 2
            }),
            image: new Circle({
              radius: 5,
              stroke: new Stroke({
                color: measureConf.sketchVertexStrokeColor || 'rgba(0, 0, 0, 0.7)'
              }),
              fill: new Fill({
                color: measureConf.sketchVertexFillColor || 'rgba(255, 255, 255, 0.2)'
              })
            })
          })
        });
        me.map.addInteraction(draw);

        var listener;
        var sketch;
        draw.on('drawstart', (evt) => {
          // clear old measure ffeatures
          me.source.clear();
          // preserve sketch
          sketch = evt.feature;

          listener = me.map.on('click', (evt) => {
            var geom = sketch.getGeometry();
            var output;
            if (geom instanceof PolygonGeom) {
              output = me.formatArea(geom);
              me.area = output;
            } else if (geom instanceof LineStringGeom) {
              output = me.formatLength(geom);
              me.distance = output;
            }
          });
        }, this);

        draw.on('drawend', () => {
          // unset sketch
          sketch = null;
          Observable.unByKey(listener);
        }, this);

        // make draw interaction available as member
        me.draw = draw;
      },
      /**
       * Removes the current interaction and clears the values.
       */
      removeInteraction () {
        var me = this;
        if (me.draw) {
          me.map.removeInteraction(me.draw);
        }
        if (me.source) {
          me.source.clear();
        }
        me.distance = ' -- ';
        me.area = ' -- ';
      },
      /**
       * Calculates and formats the length of the given line.
       *
       * @param  {ol.geom.LineString} line The LineString object to calculate length for
       */
      formatLength (line) {
        var length = Sphere.getLength(line);
        var output;
        if (length > 100) {
          output = (Math.round(length / 1000 * 100) / 100) +
              ' ' + 'km';
        } else {
          output = (Math.round(length * 100) / 100) +
              ' ' + 'm';
        }
        return output;
      },
      /**
       * Calculates and formats the area of the given polygon.
       *
       * @param  {ol.geom.Polygon} polygon The Polygon object to calculate area for
       */
      formatArea (polygon) {
        var area = Sphere.getArea(polygon);
        var output;
        if (area > 10000) {
          output = (Math.round(area / 1000000 * 100) / 100) +
              ' ' + 'km²';
        } else {
          output = (Math.round(area * 100) / 100) +
              ' ' + 'm²';
        }
        return output;
      }
    }
  }
</script>

<style>

  .wgu-measurewin {
    background-color: white;
    z-index: 2;
  }

  .wgu-measurewin.card {
    position: absolute;
  }

  .measure-result {
    font-size: 14px;
    padding-left: 8px;
    padding-bottom: 8px;
  }

</style>
