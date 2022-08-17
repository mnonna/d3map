<template>
  <section class="map">
    <svg ref="mapInner" class="map__inner">
    </svg>
  </section>
</template>

<script j>
/* eslint-disable */
import { feature } from 'topojson';
import * as d3 from "d3";
import { quadtree } from "d3-quadtree";

export default {
  name: "WorldMap",
  data() {
    return {
      mapEl: null,
      svg: null,
      grid: null,
      group: null,
      path: null,
      mapData: null,
      pathGenerator: null,
      projection: null,
      tree: null,
      pointsRaw: [],
      pointsCoords: [],
      pointsGeoData: null,
      zoom: null,
      cluster: {
        points: [],
        range: 45
      }
    }
  },
  mounted() {
    this.init();
    this.bindEvents();
  },
  methods: {
    bindEvents() {
      window.addEventListener('resize', this.onResize);
    },

    init() {
      this.mapEl = this.$refs.mapInner;
      const svgRect = this.mapEl.getBoundingClientRect();
      const { width, height } = svgRect;

      this.projection = d3.geoMercator();
      this.pathGenerator = d3.geoPath().projection(this.projection).pointRadius(1);
      
      this.calcSize();

      this.svg.attr("preserveAspectRatio", "xMinYMin meet")
        .attr("viewBox", "0 0 " + width + " " + height );

      this.mapData = d3.json('world.json').then((data) => {
        const geo = data;

        this.group = this.svg.append('g');
        
        this.group.selectAll('path').data(data.features)
        .enter().append('path')
          .attr('data-country', function(d) {
            return d.properties.name;
          })
          .attr("fill", "#0349fc")
          .attr("stroke", "#FFFFFF")
          .attr('class', 'country')
          .attr('d', this.pathGenerator);
      });

      this.zoom = d3.zoom()
      .scaleExtent([1, 16])
      .on('zoom', function(event) {
          this.group.selectAll('path')
           .attr('transform', event.transform);
          this.zoomPoints(event);
      }.bind(this));

      this.svg.call(this.zoom);
    },

    calcSize() {
      const svgRect = this.mapEl.getBoundingClientRect();
      const { width, height } = svgRect;

      this.svg = d3.select(this.mapEl);
      this.svg.attr('width', width).attr('height', height);
      this.projection.scale(width / (2.2 * Math.PI))
          .center([0, 30])
          .translate([(width / 2), (height / 2)]);
    },

    onResize() {
      this.calcSize();
      this.pathProjection();
    },

    pathProjection() {
      this.group.selectAll('path')
        .enter().append('path')
          .attr("fill", "#69b3a2")
          .attr('class', 'country')
          .attr('d', this.pathGenerator);
    },

    pointsProjection() {
      this.group.selectAll("circle")
      .enter().append("circle")
      .attr("class", "map__point")
      .attr("r", 5)
      .attr("cx", function (d) { 
        const coords = [d.long, d.lat];

        return this.projection(coords)[0]; 
      }.bind(this))
      .attr("cy", function (d) {
        const coords = [d.long, d.lat];

        return this.projection(coords)[1];
      }.bind(this))
    },

    searchGrid() {
      const svgRect = this.mapEl.getBoundingClientRect();
      const { width, height } = svgRect;

      for (let x = 0; x <= width; x += this.cluster.range) {
        for (let y = 0; y <= height; y+= this.cluster.range) {
          const searched = this.searchTree(this.tree, x, y, x + this.cluster.range, y + this.cluster.range);

          const centerPoint = searched.reduce(function(prev, current) {
            return [prev[0] + current[0], prev[1] + current[1]];
          }, [0, 0]);

          centerPoint[0] = centerPoint[0] / searched.length;
          centerPoint[1] = centerPoint[1] / searched.length;
          centerPoint.push(searched);

          if (centerPoint[0] && centerPoint[1]) {
            this.cluster.points.push(centerPoint);
          }
        }
      }
    },

    createQuadtree() {
      this.tree = d3.quadtree().addAll(this.pointsCoords);
    },

    searchTree(quadtree, x0, y0, x3, y3) {
      var validData = [];
      quadtree.visit(function(node, x1, y1, x2, y2) {
        var p = node.data;
        
        if (p) {
          p.selected = (p[0] >= x0) && (p[0] < x3) && (p[1] >= y0) && (p[1] < y3);
          
          if (p.selected) {
            validData.push(p);
          }
        }
        return x1 >= x3 || y1 >= y3 || x2 < x0 || y2 < y0;
      });
      return validData;
    },

    zoomPoints(event) {
      this.group.selectAll("circle")
        .attr('transform', event.transform);
    }
  },
}
</script>

<style lang="scss" scoped>
.map {
  width: 100%;
  height: 100vh;

  &__wrapper {
    width: 100%;
    height: 100%;
    overflow: hidden;
  }

  &__inner {
    width: 100%;
    height: 100%;
  }

  &__point {
    width: 10px;
    height: 10px;
  }
}
</style>