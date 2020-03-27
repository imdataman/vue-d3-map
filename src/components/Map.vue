<template>
  <div id="map">
    <svg class="canvas" :viewBox="`0, 0, ${width}, ${height}`">
      <g class="pathGroup"></g>
      <text id="tooltip" x="50" y="50">{{ displayedCountry }}</text>
    </svg>
  </div>
</template>

<script>
import json from "@/assets/world.json";
import * as geo from "d3-geo";
import * as selection from "d3-selection";
import * as zoom from "d3-zoom";
import * as topojson from "topojson-client";

export default {
  name: "Map",
  data() {
    return {
      myJson: topojson.feature(
        json,
        json.objects["ne_50m_admin_0_countries_lakes"]
      ),
      displayedCountry: "",
      width: 950,
      height: 550
    };
  },
  computed: {
    projection() {
      return geo
        .geoEqualEarth()
        .fitSize([this.width, this.height], this.myJson);
    },
    path() {
      return geo.geoPath().projection(this.projection);
    },
    g() {
      return selection.select(".pathGroup");
    },
    svg() {
      return selection.select(".canvas");
    },
    zoom() {
      return zoom
        .zoom()
        .scaleExtent([1, 8])
        .on("zoom", this.zoomed);
    }
  },
  methods: {
    zoomed() {
      const { transform } = selection.event;
      this.g.attr("transform", transform);
      this.g.attr("stroke-width", 1 / transform.k);
    },
    clicked(d) {
      const [[x0, y0], [x1, y1]] = this.path.bounds(d);
      selection.event.stopPropagation();
      this.svg
        .transition()
        .duration(750)
        .call(
          this.zoom.transform,
          zoom.zoomIdentity
            .translate(this.width / 2, this.height / 2)
            .scale(
              Math.min(
                8,
                0.9 / Math.max((x1 - x0) / this.width, (y1 - y0) / this.height)
              )
            )
            .translate(-(x0 + x1) / 2, -(y0 + y1) / 2),
          selection.mouse(this.svg.node())
        );
    },
    reset() {
      this.svg
        .transition()
        .duration(750)
        .call(
          this.zoom.transform,
          zoom.zoomIdentity,
          zoom
            .zoomTransform(this.svg.node())
            .invert([this.width / 2, this.height / 2])
        );
    },
    mouseentered(d, i, node) {
      this.displayedCountry = d.properties.NAME;
      selection
        .select(node[i])
        .classed("active", true)
        .raise();
    },
    mouseleft(d, i, node) {
      this.displayedCountry = "";
      selection.select(node[i]).classed("active", false);
    }
  },
  mounted() {
    this.g
      .selectAll(".country")
      .data(this.myJson.features)
      .join("path")
      .attr("class", "country")
      .attr("d", this.path)
      .on("mouseenter", this.mouseentered)
      .on("mouseleave", this.mouseleft)
      .on("click", this.clicked);
    this.svg.on("click", this.reset);
    this.svg.call(this.zoom);
  }
};
</script>

<style lang="scss">
.country {
  fill: WhiteSmoke;
  stroke: Gainsboro;
}

#tooltip {
  font-size: 1.5rem;
}

.active {
  stroke: black;
}
</style>
