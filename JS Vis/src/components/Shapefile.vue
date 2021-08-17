<template>
  <div class="container-fluid">
    <div class="row row-eq-height">
      <div class="col-3">
        <br /><br />
        <strong class="headers">Selection info</strong> <br /><br />
        <div id="info-boxes"></div>
      </div>
      <div class="col-6">
        <br /><br />
        <strong class="headers">Map </strong><br /><br />
        <div id="map" style="z-index: 10"></div>
        <div id="map-legend" style="z-index: 1"></div>
        <br />
        <div id="slider" class="slider">
          <vue-range-slider
            v-model="range"
            class="centered"
            min="0.0"
            max="1.8"
            step="0.1"
            dotSize="20"
            width="600px"
          ></vue-range-slider>
        </div>
      </div>
      <div class="col-2">
        <br /><br />
        <strong class="headers" style="left: 100px">Options </strong
        ><br /><br />
        <div id="buttons" class="btn-group-vertical">
          <b-button
            id="button1"
            class="mb-3 mybtn"
            pill
            v-on:click="init()"
            variant="light"
            size="lg"
          >
            <span style="font-size: smaller">Reset</span>
          </b-button>
          <b-button
            id="button2"
            class="mb-3 mybtn"
            pill
            variant="light"
            size="lg"
          >
            <span style="font-size: smaller">Center</span>
          </b-button>
          <b-button-group>
            <b-button
              id="button3"
              class="mb-3 mybtn2"
              square
              variant="light"
              size="lg"
              v-on:click="chart_toggle()"
              :class="[show_bar ? 'active' : '']"
            >
              <span style="font-size: smaller">Bar</span>
            </b-button>
            <b-button
              id="button3"
              class="mb-3 mybtn2"
              square
              variant="light"
              size="lg"
              v-on:click="chart_toggle()"
              :class="[!show_bar ? 'active' : '']"
            >
              <span style="font-size: smaller">Radial</span>
            </b-button>
          </b-button-group>
        </div>
        <br /><br />
        <strong class="headers">Chart comparison</strong> <br /><br />
        <div id="chart"></div>
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
      d3.selectAll("#chart > svg").remove();
      glob.select_count = 0;
      glob.selection_data = [];

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
        .attr("height", 200);

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

      const max_rate = 0.017455;

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

      function remove_item(array, code) {
        var iter = 0;
        while (iter < array.length) {
          if (array[iter][0].code === code) {
            array.splice(iter, 1);
          } else {
            ++iter;
          }
        }
        return array;
      }

      function clickaction() {
        const areaname = d3.select(this).attr("areaname");
        const code = d3.select(this).attr("code");
        const cases = d3.select(this).attr("cases");
        const case_rate = d3.select(this).attr("case_rate");
        const population = d3.select(this).attr("population");
        const selection_info = [
          {
            areaname: areaname,
            code: code,
            cases: cases,
            case_rate: case_rate,
            population: population,
          },
        ];
        if (d3.select(this).attr("class").includes("click-indicator")) {
          d3.select(this).attr("class", "highlight");
          remove_item(glob.selection_data, selection_info[0].code);
          --glob.select_count;
        } else {
          d3.select(this).attr("class", "click-indicator");
          glob.selection_data.push(selection_info);
          ++glob.select_count;
        }
        // console.log(glob.selection_data);
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
        .html(function () {
          if (glob.select_count >= 1) {
            return glob.selection_data[0][0].areaname;
          } else {
            return "";
          }
        });
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
        .html(function () {
          if (glob.select_count >= 1) {
            return (
              "<tspan " +
              "x=" +
              "35" +
              " dy=" +
              "25" +
              "> Population: " +
              glob.selection_data[0][0].population +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "30" +
              " dy=" +
              "25" +
              "> Cases: " +
              glob.selection_data[0][0].cases +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "30" +
              " dy=" +
              "25" +
              "> Case pct: " +
              glob.selection_data[0][0].case_rate +
              "%" +
              "</tspan>"
            );
          } else {
            return "";
          }
        });

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
        .html(function () {
          if (glob.select_count >= 2) {
            return glob.selection_data[1][0].areaname;
          } else {
            return "";
          }
        });
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
        .html(function () {
          if (glob.select_count >= 2) {
            return (
              "<tspan " +
              "x=" +
              "35" +
              " dy=" +
              "25" +
              "> Population: " +
              glob.selection_data[1][0].population +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "30" +
              " dy=" +
              "25" +
              "> Cases: " +
              glob.selection_data[1][0].cases +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "30" +
              " dy=" +
              "25" +
              "> Case pct: " +
              glob.selection_data[1][0].case_rate +
              "%" +
              "</tspan>"
            );
          } else {
            return "";
          }
        });

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
        .html(function () {
          if (glob.select_count >= 3) {
            return glob.selection_data[2][0].areaname;
          } else {
            return "";
          }
        });
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
        .html(function () {
          if (glob.select_count >= 3) {
            return (
              "<tspan " +
              "x=" +
              "35" +
              " dy=" +
              "25" +
              "> Population: " +
              glob.selection_data[2][0].population +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "30" +
              " dy=" +
              "25" +
              "> Cases: " +
              glob.selection_data[2][0].cases +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "30" +
              " dy=" +
              "25" +
              "> Case pct: " +
              glob.selection_data[2][0].case_rate +
              "%" +
              "</tspan>"
            );
          } else {
            return "";
          }
        });
    },
    draw_bar() {
      d3.selectAll("#chart > svg").remove();

      const glob = this;

      const bar = d3
        .select("#chart")
        .append("svg")
        .attr("x", 0)
        .attr("y", 20)
        .attr("width", 330)
        .attr("height", 450);

      bar.append("rect").attr("x", 5).attr("y", 5).attr("class", "chart_bg");

      var x_scale = d3.scaleBand().range([25, 285]).padding(0.6);
      var y_scale = d3.scaleLinear().range([310, 30]);
      var bar_g = bar
        .append("g")
        .attr("transform", "translate(" + 0 + "," + -30 + ")");

      x_scale.domain(
        glob.selection_data.map(function (d) {
          return d[0].areaname;
        })
      );
      y_scale.domain([
        0,
        d3.max(
          glob.selection_data.map(function (d) {
            return d[0].population;
          })
        ),
      ]);

      bar_g
        .append("g")
        .attr("transform", "translate(" + 25 + "," + 330 + ")")
        .call(d3.axisBottom(x_scale))
        .selectAll("text")
        .style("text-anchor", "end")
        .attr("dx", "-1.8em")
        .attr("dy", "-0.8em")
        .attr("transform", "rotate(-90)");

      bar_g
        .append("g")
        .attr("transform", "translate(" + 50 + "," + 20 + ")")
        .call(d3.axisLeft(y_scale).ticks(10))
        .append("text")
        .attr("y", 6)
        .attr("dy", "0.71em")
        .attr("text-anchor", "end")
        .text("Num. of Cases");

      bar_g
        .selectAll(".bar")
        .data(glob.selection_data)
        .enter()
        .append("rect")
        .attr("class", "bars")
        .attr("transform", "translate(" + 25 + "," + 20 + ")")
        .attr("x", function (d) {
          return x_scale(d[0].areaname);
        })
        .attr("width", x_scale.bandwidth())
        .attr("y", function () {
          return y_scale(0);
        })
        .attr("height", function () {
          return 310 - y_scale(0);
        })
        .transition()
        .duration(800)
        .attr("y", function (d) {
          return y_scale(d[0].population);
        })
        .attr("height", function (d) {
          return 310 - y_scale(d[0].population);
        });
    },
    draw_radial() {
      d3.selectAll("#chart > svg").remove();

      // const glob = this;
    },
    chart_toggle() {
      const glob = this;
      glob.show_bar = !glob.show_bar;
      if (glob.show_bar === true) {
        this.draw_bar();
      } else {
        this.draw_radial();
      }
    },
  },
  data() {
    return {
      range: [0, 1.8],
      select_count: 0,
      selection_data: [],
      show_bar: true,
    };
  },
  watch: {
    range() {
      this.init();
    },
    select_count() {
      this.redraw_infoboxes();
      if (this.show_bar === true) {
        this.draw_bar();
      } else {
        this.draw_radial();
      }
    },
  },
  async mounted() {
    await this.init();
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
.mybtn {
  border-color: #000 !important;
  border-width: 2.5px !important;
  width: 150px !important;
  left: 30px;
}
.mybtn2 {
  border-color: #000 !important;
  border-width: 2.5px !important;
  width: 100px !important;
}
.active {
  background: rgba(33, 184, 207, 0.692) !important;
}
.chart_bg {
  position: absolute;
  height: 425px;
  width: 320px;
  fill: rgb(255, 255, 255);
  stroke: #000;
  stroke-width: 2.5;
}
.bars {
  fill: rgb(101, 131, 167);
}
</style>
