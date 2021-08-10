<template>
  <div class="container-fluid">
    <div class="row row-eq-height">
      <div class="col-3">
        <br /><br />
        <strong class="headers">LHS Panel</strong> <br /><br />
        <div id="info-boxes"></div>
      </div>
      <div class="col-6">
        <br /><br />
        <strong class="headers">Map </strong><br /><br />
        <div id="map" style="z-index: 10"></div>
        <div id="map-legend" style="z-index: 1"></div>
        <div id="slider" style="z-index: 1">
          <vue-range-slider
            v-model="range"
            class="centered"
            min="0.0"
            max="1.8"
            step="0.1"
            dotSize="15"
            width="600px"
          ></vue-range-slider>
        </div>
      </div>
      <div class="col-2">
        <br /><br />
        <strong class="headers">Options </strong><br /><br />
        <div id="buttons" class="btn-group-vertical">
          <b-button
            id="button1"
            class="mb-3"
            pill
            v-on:click="init()"
            variant="outline-dark"
            size="lg"
          >
            <span style="font-size: smaller">Reset</span>
          </b-button>
          <b-button
            id="button2"
            class="mb-3"
            pill
            variant="outline-dark"
            size="lg"
          >
            <span style="font-size: smaller">Button2</span>
          </b-button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import * as d3 from "d3";
import * as topo from "topojson-client";
import "vue-range-component/dist/vue-range-slider.css";
import VueRangeSlider from "vue-range-component";

export default {
  name: "Shapefile",
  components: { VueRangeSlider },
  methods: {
    async init() {
      const glob = this;
      const shapefile_dat = await d3.json("/data/Simplified_visdata.json");

      glob.shape_data = shapefile_dat;

      //Reset to default
      d3.selectAll("#map > svg").remove();
      d3.selectAll("#map-legend > svg").remove();
      glob.areanames_clicked = [];
      glob.areacodes_clicked = [];
      glob.areacases_clicked = [];
      glob.areapop_clicked = [];
      glob.areacaserate_clicked = [];
      glob.areaNA_clicked = [];
      glob.areaNApct_clicked = [];
      glob.select_count = 0;

      glob.width = 800;
      glob.height = 600;

      const svg = d3
        .select("#map")
        .append("svg")
        .attr("viewBox", [screen.width * (2 / 5), 0, glob.width, glob.height])
        .attr("stroke-linejoin", "round")
        .attr("stroke-linecap", "round");

      svg.call(
        d3
          .zoom()
          .extent([
            [0, 0],
            [glob.width, glob.height],
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

      var legend = d3
        .select("#map-legend")
        .append("svg")
        .attr("x", screen.width * (2 / 5))
        .attr("y", 700)
        .attr("height", 100)
        .attr("width", 600);

      var defs = legend.append("defs");

      //Slider on case rate
      d3.select("#slider")
        .attr("x", screen.width * (2 / 5))
        .attr("width", 600)
        .attr("height", 100);

      const path = d3.geoPath();

      glob.list = [];

      glob.list.push(
        ...topo
          .feature(glob.shape_data, glob.shape_data.objects.Visualisation_data)
          .features.filter(function (d) {
            if (
              glob.range[0] <= d.properties.Case_rate * 100 &&
              d.properties.Case_rate * 100 <= glob.range[1]
            ) {
              return d;
            }
          })
      );

      // Color scheme setting
      const colorscale = d3.scaleSequential(d3.interpolateOrRd);

      const max_rate = d3.max(
        glob.list.map((row) => {
          return row.properties.Case_rate;
        })
      );

      const NA_cases = Number(
        d3.mean(
          glob.list.map((row) => {
            return row.properties.Cases;
          })
        )
      ).toFixed(0);

      const NA_case_rate = Number(
        d3.mean(
          glob.list.map((row) => {
            return row.properties.Case_rate;
          })
        ) * 100
      ).toFixed(2);

      function mouseout() {
        tooltip.style("visibility", "hidden");
        if (d3.select(this).attr("class").includes("highlight")) {
          d3.select(this).attr("class", "none");
        } else {
          d3.select(this).attr("class", "click-indicator");
        }
      }

      function mousemove(e) {
        return tooltip
          .style("top", e.pageY - 20 + "px")
          .style("left", e.pageX - 325 + "px");
      }

      function mouseover() {
        tooltip
          .style("visibility", "visible")
          .html(
            d3.select(this).attr("areaname") +
              "<br/>Population: " +
              d3.select(this).attr("population") +
              "<br/>Cases: " +
              d3.select(this).attr("cases") +
              d3.select(this).attr("NA_ind") +
              "<br/> Case pct: " +
              d3.select(this).attr("case_rate") +
              "%" +
              d3.select(this).attr("NA_ind_pct")
          );
        if (d3.select(this).attr("class").includes("none")) {
          d3.select(this).attr("class", "highlight");
        }
      }

      function remove_item(array, item) {
        var iter = 0;
        while (iter < array.length) {
          if (array[iter] === item) {
            array.splice(iter, 1);
          } else {
            ++iter;
          }
        }
        return array;
      }

      function clickaction() {
        if (d3.select(this).attr("class").includes("click-indicator")) {
          d3.select(this).attr("class", "highlight");
          --glob.select_count;
          remove_item(glob.areanames_clicked, d3.select(this).attr("areaname"));
          remove_item(glob.areacodes_clicked, d3.select(this).attr("code"));
          remove_item(glob.areacases_clicked, d3.select(this).attr("cases"));
          remove_item(
            glob.areacaserate_clicked,
            d3.select(this).attr("case_rate")
          );
          remove_item(glob.areapop_clicked, d3.select(this).attr("population"));
          remove_item(
            glob.areaNApct_clicked,
            d3.select(this).attr("NA_ind_pct")
          );
          remove_item(glob.areaNA_clicked, d3.select(this).attr("NA_ind"));
        } else {
          d3.select(this).attr("class", "click-indicator");
          ++glob.select_count;
          glob.areanames_clicked.push(d3.select(this).attr("areaname"));
          glob.areacodes_clicked.push(d3.select(this).attr("code"));
          glob.areacases_clicked.push(d3.select(this).attr("cases"));
          glob.areacaserate_clicked.push(d3.select(this).attr("case_rate"));
          glob.areapop_clicked.push(d3.select(this).attr("population"));
          glob.areaNApct_clicked.push(d3.select(this).attr("NA_ind_pct"));
          glob.areaNA_clicked.push(d3.select(this).attr("NA_ind"));
        }
        console.log(glob.areacases_clicked);
      }

      // Map data and tools
      svg
        .append("g")
        .selectAll("path")
        .data(glob.list)
        .join("path")
        .attr("vector-effect", "non-scaling-stroke")
        .attr("d", path)
        .attr("class", "none")
        .attr("fill", (d) => {
          return colorscale(d.properties.Case_rate / max_rate);
        })
        .attr("code", (d) => d.properties.code)
        .attr("areaname", (d) => d.properties.areaName)
        .attr("population", (d) => d.properties.pop_count)
        .attr("cases", (d) => Number(d.properties.Cases))
        .attr("case_rate", (d) =>
          Number(d.properties.Case_rate * 100).toFixed(2)
        )
        .attr("click-indicator", false)
        .on("mouseover", mouseover)
        .on("mousemove", mousemove)
        .on("mouseout", mouseout)
        .on("click", clickaction)
        .attr("NA_ind", (d) => {
          if (d.properties.Cases > NA_cases) {
            return (
              "<svg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='Black' class='bi bi-caret-up-fill' viewBox='0 0 16 16'>" +
              "<path d='m7.247 4.86-4.796 5.481c-.566.647-.106 1.659.753 1.659h9.592a1 1 0 0 0 .753-1.659l-4.796-5.48a1 1 0 0 0-1.506 0z'/>" +
              "</svg>"
            );
          } else if (d.properties.Cases < NA_cases) {
            return (
              "<svg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='Black' class='bi bi-caret-down-fill' viewBox='0 0 16 16'>" +
              "<path d='M7.247 11.14 2.451 5.658C1.885 5.013 2.345 4 3.204 4h9.592a1 1 0 0 1 .753 1.659l-4.796 5.48a1 1 0 0 1-1.506 0z'/>" +
              "</svg>"
            );
          } else {
            return (
              "<svg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='Black' class='bi bi-dash' viewBox='0 0 16 16'>" +
              "<path d='M4 8a.5.5 0 0 1 .5-.5h7a.5.5 0 0 1 0 1h-7A.5.5 0 0 1 4 8z'/>" +
              "</svg>"
            );
          }
        })
        .attr("NA_ind_pct", (d) => {
          if (Number(d.properties.Case_rate * 100).toFixed(2) > NA_case_rate) {
            return (
              "<svg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='Black' class='bi bi-caret-up-fill' viewBox='0 0 16 16'>" +
              "<path d='m7.247 4.86-4.796 5.481c-.566.647-.106 1.659.753 1.659h9.592a1 1 0 0 0 .753-1.659l-4.796-5.48a1 1 0 0 0-1.506 0z'/>" +
              "</svg>"
            );
          } else if (
            Number(d.properties.Case_rate * 100).toFixed(2) < NA_case_rate
          ) {
            return (
              "<svg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='Black' class='bi bi-caret-down-fill' viewBox='0 0 16 16'>" +
              "<path d='M7.247 11.14 2.451 5.658C1.885 5.013 2.345 4 3.204 4h9.592a1 1 0 0 1 .753 1.659l-4.796 5.48a1 1 0 0 1-1.506 0z'/>" +
              "</svg>"
            );
          } else {
            return (
              "<svg xmlns='http://www.w3.org/2000/svg' width='16' height='16' fill='Black' class='bi bi-dash' viewBox='0 0 16 16'>" +
              "<path d='M4 8a.5.5 0 0 1 .5-.5h7a.5.5 0 0 1 0 1h-7A.5.5 0 0 1 4 8z'/>" +
              "</svg>"
            );
          }
        });

      // Add Legend
      var colorscale_linear = d3
        .scaleLinear()
        .domain([0.0, 0.35, 0.7, 1.05, 1.4, 1.75]);

      var linearGradient = defs
        .append("linearGradient")
        .attr("id", "linear-gradient");

      linearGradient
        .attr("x1", "0%")
        .attr("y1", "0%")
        .attr("x2", "100%")
        .attr("y2", "0%");

      linearGradient
        .selectAll("stop")
        .data([
          {
            offset: "0%",
            color: d3.interpolateOrRd(0),
          },
          {
            offset: "20%",
            color: d3.interpolateOrRd(0.2),
          },
          {
            offset: "40%",
            color: d3.interpolateOrRd(0.4),
          },
          {
            offset: "60%",
            color: d3.interpolateOrRd(0.6),
          },
          {
            offset: "80%",
            color: d3.interpolateOrRd(0.8),
          },
          {
            offset: "100%",
            color: d3.interpolateOrRd(1),
          },
        ])
        .enter()
        .append("stop")
        .attr("offset", function (d) {
          return d.offset;
        })
        .attr("stop-color", function (d) {
          return d.color;
        });

      var xLeg = d3.scaleLinear().domain([0, 1.75]).range([10, 590]);

      var axisLeg = d3.axisBottom(xLeg).tickValues(colorscale_linear.domain());

      legend
        .append("rect")
        .attr("x", 0)
        .attr("y", 50)
        .attr("width", 600)
        .attr("height", 20)
        .style("fill", "url(#linear-gradient)");

      legend
        .append("text")
        .attr("x", 300)
        .attr("y", 30)
        .style("text-anchor", "middle")
        .style("font", "Avenir")
        .style("font-size", "12pt")
        .text("Case rate");

      legend
        .attr("class", "axis")
        .append("g")
        .attr("transform", "translate(0,70)")
        .call(axisLeg);
    },

    redraw_infoboxes() {
      d3.selectAll("#info-boxes > svg").remove();

      const glob = this;

      const infoboxes = d3
        .select("#info-boxes")
        .append("svg")
        .attr("x", 10)
        .attr("y", 75)
        .attr("width", 300)
        .attr("height", 600);

      infoboxes
        .append("rect")
        .attr("x", 10)
        .attr("y", 20)
        .attr("class", "infobox_rect")
        .style("visibility", function () {
          if (glob.select_count < 1) {
            return "hidden";
          }
        });
      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 50)
        .attr("class", "infobox_title")
        .style("visibility", function () {
          if (glob.select_count < 1) {
            return "hidden";
          }
        })
        .html(glob.areanames_clicked[0]);
      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 70)
        .attr("class", "infobox_text")
        .style("visibility", function () {
          if (glob.select_count < 1) {
            return "hidden";
          }
        })
        .html(
          "<tspan " +
            "x=" +
            "35" +
            " dy=" +
            "25" +
            "> Population: " +
            glob.areapop_clicked[0] +
            "</tspan>" +
            "<tspan " +
            "x=" +
            "30" +
            " dy=" +
            "25" +
            "> Cases: " +
            glob.areacases_clicked[0] +
            glob.areaNA_clicked[0] +
            "</tspan>" +
            "<tspan " +
            "x=" +
            "30" +
            " dy=" +
            "25" +
            "> Case pct: " +
            glob.areacaserate_clicked[0] +
            "%" +
            glob.areaNApct_clicked[0] +
            "</tspan>"
        );

      infoboxes
        .append("rect")
        .attr("x", 10)
        .attr("y", 190)
        .attr("class", "infobox_rect")
        .style("visibility", function () {
          if (glob.select_count < 2) {
            return "hidden";
          }
        });
      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 220)
        .attr("class", "infobox_title")
        .style("visibility", function () {
          if (glob.select_count < 2) {
            return "hidden";
          }
        })
        .html(glob.areanames_clicked[1]);
      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 240)
        .attr("class", "infobox_text")
        .style("visibility", function () {
          if (glob.select_count < 2) {
            return "hidden";
          }
        })
        .html(
          "<tspan " +
            "x=" +
            "35" +
            " dy=" +
            "25" +
            "> Population: " +
            glob.areapop_clicked[1] +
            "</tspan>" +
            "<tspan " +
            "x=" +
            "30" +
            " dy=" +
            "25" +
            "> Cases: " +
            glob.areacases_clicked[1] +
            glob.areaNA_clicked[1] +
            "</tspan>" +
            "<tspan " +
            "x=" +
            "30" +
            " dy=" +
            "25" +
            "> Case pct: " +
            glob.areacaserate_clicked[1] +
            "%" +
            glob.areaNApct_clicked[1] +
            "</tspan>"
        );

      infoboxes
        .append("rect")
        .attr("x", 10)
        .attr("y", 360)
        .attr("class", "infobox_rect")
        .style("visibility", function () {
          if (glob.select_count < 3) {
            return "hidden";
          }
        });
      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 390)
        .attr("class", "infobox_title")
        .style("visibility", function () {
          if (glob.select_count < 3) {
            return "hidden";
          }
        })
        .html(glob.areanames_clicked[2]);
      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 410)
        .attr("class", "infobox_text")
        .style("visibility", function () {
          if (glob.select_count < 3) {
            return "hidden";
          }
        })
        .html(
          "<tspan " +
            "x=" +
            "35" +
            " dy=" +
            "25" +
            "> Population: " +
            glob.areapop_clicked[2] +
            "</tspan>" +
            "<tspan " +
            "x=" +
            "30" +
            " dy=" +
            "25" +
            "> Cases: " +
            glob.areacases_clicked[2] +
            glob.areaNA_clicked[2] +
            "</tspan>" +
            "<tspan " +
            "x=" +
            "30" +
            " dy=" +
            "25" +
            "> Case pct: " +
            glob.areacaserate_clicked[2] +
            "%" +
            glob.areaNApct_clicked[2] +
            "</tspan>"
        );
    },
  },
  data() {
    return {
      range: [0, 1.8],
      areanames_clicked: [],
      areacodes_clicked: [],
      areacases_clicked: [],
      areapop_clicked: [],
      areacaserate_clicked: [],
      areaNA_clicked: [],
      areaNApct_clicked: [],
      select_count: 0,
    };
  },
  watch: {
    range() {
      this.init();
    },
    areanames_clicked() {
      this.redraw_infoboxes();
    },
  },
  async mounted() {
    await this.init();
    this.redraw_infoboxes();
  },
};
</script>
<style>
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
.headers {
  font-size: 16pt;
  fill: #000;
}
.centered {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
}
.tooltip {
  padding: 7px;
  background: white;
  border: 2px;
  border-style: solid;
  border-radius: 2px;
  opacity: 10 !important;
}
.container-fluid {
  min-height: 100%;
}
.axis path,
.axis line {
  fill: none;
  stroke: none;
  shape-rendering: crispEdges;
}
.axis text {
  font-family: Avenir;
  fill: #000;
  font-size: 9pt;
}
.click-indicator {
  stroke-width: 2;
  stroke: rgb(19, 122, 33);
}
.highlight {
  stroke-width: 2;
  stroke: #000;
}
.slider .range-slider-fill {
  background-color: #ff9c00;
}
.none {
  stroke-width: 0.06;
  stroke: #000;
}
.infobox_rect {
  position: absolute;
  height: 150px;
  width: 280px;
  fill: rgb(255, 255, 255);
  stroke: #000;
  stroke-width: 2;
}
.infobox_title {
  fill: rgb(0, 0, 0);
  font-size: 16pt;
}
.infobox_text {
  fill: #000;
  font-size: 14pt;
}
.hide {
  visibility: hidden;
}
</style>
