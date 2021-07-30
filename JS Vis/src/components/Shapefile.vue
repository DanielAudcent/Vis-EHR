<template>
  <!-- html code goes here -->
  <div class="container-fluid" style="col">
    <div class="left">
      <div id="map"></div>
    </div>
    <div class="right">
      <div id="buttons" class="btn-group-vertical">
        <b-button
          id="button1"
          class="mb-3"
          squared
          v-on:click="init()"
          variant="outline-light"
          size="lg"
        >
          Reset
        </b-button>
        <b-button
          id="button2"
          class="mb-3"
          squared
          variant="outline-light"
          size="lg"
        >
          button2
        </b-button>
      </div>
    </div>
    <div class="clear"></div>
  </div>
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
        .attr("viewBox", [screen.width * (2 / 5), 0, 800, 900])
        .attr("stroke-linejoin", "round")
        .attr("stroke-linecap", "round");

      svg.call(
        d3
          .zoom()
          .extent([
            [screen.width * (2 / 5), 0],
            [800, 900],
          ])
          .scaleExtent([1, 10])
          .on("zoom", zoomed)
      );

      function zoomed({ transform }) {
        svg.attr("transform", transform);
      }

      const tooltip = d3
        .select("#map")
        .append("div")
        .style("position", "absolute")
        .style("visibility", "hidden")
        .style("z-index", "10")
        .attr("class", "tooltip");

      const path = d3.geoPath();

      const list = [];

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
        .attr("code", (d) => d.properties.code)
        .attr("areaname", (d) => d.properties.areaName)
        .attr("population", (d) => d.properties.pop_count)
        .attr("cases", (d) => d.properties.Cases)
        .attr("case_rate", (d) => {
          d.properties.Case_rate * 100;
        })
        .on("mouseover", function () {
          d3.select(this).attr("stroke-width", 2);
          console.log(d3.select(this).attr("population"));
          tooltip
            .style("visibility", "visible")
            .html(
              d3.select(this).attr("areaname") +
                "<br/>Population: " +
                d3.select(this).attr("population") +
                "<br/>Cases: " +
                d3.select(this).attr("cases") +
                "<br/> Case Rate: " +
                d3.select(this).attr("case_rate") +
                "%"
            );
        })
        .on("mousemove", function (e) {
          return tooltip
            .style("top", e.pageY - 20 + "px")
            .style("left", e.pageX + 20 + "px");
        })
        .on("mouseout", function () {
          d3.select(this).attr("stroke-width", 0.15);
          tooltip.style("visibility", "hidden");
        });
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
.left {
  float: left;
  left: 0;
  width: 70%;
  height: 1000px;
  background-color: rgb(133, 149, 165);
  margin: 30px;
}
.right {
  float: right;
  right: 0;
  width: 20%;
  height: 1000px;
  background-color: rgb(133, 149, 165);
  margin: 30px;
}
.centered {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
}
.clear {
  clear: both;
}
.tooltip {
  padding: 7px;
  background: white;
  border: 2px;
  border-style: solid;
  border-radius: 2px;
  opacity: 10 !important;
}
</style>
