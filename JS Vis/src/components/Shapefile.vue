<template>
  <!-- html code goes here -->
  <div id="map"></div>
</template>

<script>
import * as d3 from "d3";
import * as topo from "topojson-client";

export default {
  name: "Shapefile",
  methods: {
    async init() {
      const shapefile_dat = await d3
        .json("/data/Simplified_visdata.json")
        // .json(".data/Simplified_visdata.json")
        .then(function (data, error) {
          console.log(data, error);
          console.log("error:", error);
        });

      d3.selectAll("#map > svg").remove();

      const svg = d3
        .select("#map")
        .append("svg")
        .attr("code", "MSOA")
        .attr("viewBox", [0, 0, 800, 400])
        .attr("stroke-linejoin", "round")
        .attr("stroke-linecap", "round");

      svg.call(
        d3
          .zoom()
          .extent([
            [0, 0],
            [800, 400],
          ])
          .scaleExtent([1, 6])
          .on("zoom", zoomed)
      );

      function zoomed({ transform }) {
        svg.attr("transform", transform);
      }

      const path = d3.geoPath();

      const list = [];

      list.push(
        ...topo
          .feature(shapefile_dat, shapefile_dat.features)
          .filter((e) => e.geometry && e.geometry.type)
      );

      svg
        .append("g")
        .attr("class", "MSOA")
        .attr("stroke", "#000")
        .selectAll("path")
        .data(list)
        .join("path")
        .attr("vector-effect", "non-scaling-stroke")
        .attr("d", path)
        .attr("fill", "none")
        .attr("code", (d) => d.properties.code);
    },
  },
  data() {
    return {
      MSOA: { visibility: true, color: "dark" },
    };
  },
  async mounted() {
    this.init();
  },
};
</script>
<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
