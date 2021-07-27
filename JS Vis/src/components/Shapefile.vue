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
      const shapefile_dat = await d3.json("/data/Simplified_visdata.json");

      d3.selectAll("#map > svg").remove();

      const svg = d3
        .select("#map")
        .append("svg")
        .attr("code", "MSOA")
        .attr("viewBox", [screen.width * (2 / 5), 0, 800, 900])
        .attr("stroke-linejoin", "round")
        .attr("stroke-linecap", "round");

      svg.call(
        d3
          .zoom()
          .extent([
            [0, 0],
            [1000, 1000],
          ])
          .scaleExtent([1, 6])
          .on("zoom", zoomed)
      );

      function zoomed({ transform }) {
        svg.attr("transform", transform);
      }

      const path = d3.geoPath();

      const list = [];
      console.log(shapefile_dat.objects.Visualisation_data);

      list.push(
        ...topo
          .feature(shapefile_dat, shapefile_dat.objects.Visualisation_data)
          .features.filter((e) => e.geometry && e.geometry.type)
      );

      const max_rate = d3.max(
        list.map((row) => {
          return row.properties.Case_rate;
        })
      );

      const colorscale = d3.scaleSequential(d3.interpolateOrRd);

      svg
        .append("g")
        .attr("class", "MSOA")
        .attr("stroke", "#000")
        .style("stroke-width", 0.15)
        .selectAll("path")
        .data(list)
        .join("path")
        .attr("vector-effect", "non-scaling-stroke")
        .attr("d", path)
        .attr("fill", (d) => colorscale(d.properties.Case_rate / max_rate))
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
