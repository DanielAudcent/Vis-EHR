<template>
  <div class="container-fluid">
    <div class="row row-eq-height">
      <div class="col-4">
        <br />
        <strong class="headers">Selection info</strong> <br /><br />
        <div id="info-boxes"></div>
        <div id="cr-hist"></div>
      </div>
      <div class="col-5">
        <br />
        <strong class="headers">Map </strong><br /><br />
        <div id="map"></div>
        <br />
        <div id="map-legend"></div>
        <div id="slider" class="slider">
          <vue-range-slider
            v-model="range"
            :min="0.0"
            :max="1.8"
            :step="0.1"
            :dotSize="20"
            width="550px"
          ></vue-range-slider>
        </div>
      </div>
      <div class="col-3">
        <br />
        <strong class="headers" style="left: 100px">Options </strong
        ><br /><br />
        <div id="buttons" class="btn-group-vertical">
          <b-button
            id="button1"
            class="mb-3 mybtn"
            pill
            v-on:click="
              circles_range = [100, 101];
              init();
              draw_hist();
            "
            variant="light"
            size="lg"
          >
            <span style="font-size: smaller">Reset</span>
          </b-button>
          <strong class="headers2">Chart type</strong>
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
              <span style="font-size: smaller">Radar</span>
            </b-button>
          </b-button-group>
          <strong class="headers2">Colour Scheme</strong>
          <b-form-select
            v-model="col_select"
            :options="col_options"
            class="mb-3 mybtn3"
            size="lg"
          >
          </b-form-select>
        </div>
        <br /><br />
        <strong class="headers">Chart comparison</strong><br />
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

      //Reset to default settings / view
      d3.selectAll("#map > *").remove();
      d3.selectAll("#map-legend > svg").remove();
      d3.selectAll("#chart > *").remove();
      d3.selectAll("#info-boxes > svg").remove();
      glob.select_count = 0;
      glob.selection_data = [];

      glob.width = 650;
      glob.height = 600;

      d3.select("#info-boxes")
        .append("svg")
        .attr("x", 10)
        .attr("y", 65)
        .attr("width", 380)
        .attr("height", 380);

      var legend = d3
        .select("#map-legend")
        .append("svg")
        .attr("x", screen.width * (2 / 5))
        .attr("y", 1000)
        .attr("height", 100)
        .attr("width", 570);

      var defs = legend.append("defs");

      const svg = d3
        .select("#map")
        .append("svg")
        .attr("viewBox", [0, 0, glob.width, glob.height])
        .attr("stroke-linejoin", "round")
        .attr("stroke-linecap", "round")
        .attr("fill", "#000");

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

      function zoomed(e) {
        svg.attr("transform", e.transform);
      }

      const tooltip = d3
        .select("#map")
        .append("div")
        .style("position", "absolute")
        .style("visibility", "hidden")
        .style("z-index", "10")
        .attr("class", "tooltip");

      //Slider on case rate
      d3.select("#slider")
        .attr("x", screen.width * (2 / 5))
        .attr("width", 600)
        .attr("height", 120);

      const path = d3.geoPath();

      glob.list = [];

      glob.list.push(
        ...topo
          .feature(glob.shape_data, glob.shape_data.objects.Visualisation_data)
          .features.filter(function (d) {
            return d;
          })
      );

      // Color scheme settings
      const colorscheme = glob.col_select;
      const colorscale = d3.scaleSequential(colorscheme).domain([1, 0]);

      // const max_rate = 0.017455;
      const max_rate = glob.range[1] / 100;

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
        .attr("fill", function (d) {
          return glob.range[0] <= d.properties.Case_rate * 100 &&
            d.properties.Case_rate * 100 <= glob.range[1]
            ? colorscale(d.properties.Case_rate / max_rate)
            : "rgb(141, 168, 201)";
        })
        .attr("code", (d) => d.properties.code)
        .attr("areaname", (d) => d.properties.areaName)
        .attr("population", (d) => d.properties.pop_count)
        .attr("cases", (d) => Number(d.properties.Cases))
        .attr("case_rate", (d) =>
          Number(d.properties.Case_rate * 100).toFixed(2)
        )
        // .attr("click-indicator", false)
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

      svg
        .selectAll("circles")
        .data(glob.list)
        .enter()
        .append("circle")
        .attr("cx", function (d) {
          return path.centroid(d)[0];
        })
        .attr("cy", function (d) {
          return path.centroid(d)[1];
        })
        .attr("r", function () {
          if (glob.circles_count > 200) {
            return "0.25px";
          } else {
            return "0.45px";
          }
        })
        .attr("class", "circle")
        .attr("class", function (d) {
          const val = d.properties.Case_rate * 100;
          if (val > glob.circles_range[0] && val < glob.circles_range[1]) {
            return "circle";
          } else {
            return "circle_hide";
          }
        });

      // Add Legend
      var colorscale_linear = d3
        .scaleLinear()
        .domain([
          Number(0.0).toFixed(2),
          Number(0.35).toFixed(2),
          Number(0.7).toFixed(2),
          Number(1.05).toFixed(2),
          Number(1.4).toFixed(2),
          Number(1.75).toFixed(2),
        ]);

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
            color: colorscheme(1),
          },
          {
            offset: "20%",
            color: colorscheme(0.8),
          },
          {
            offset: "40%",
            color: colorscheme(0.6),
          },
          {
            offset: "60%",
            color: colorscheme(0.4),
          },
          {
            offset: "80%",
            color: colorscheme(0.2),
          },
          {
            offset: "100%",
            color: colorscheme(0),
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

      var xLeg = d3.scaleLinear().domain([0, 1.75]).range([10, 540]);

      var axisLeg = d3.axisBottom(xLeg).tickValues(colorscale_linear.domain());

      legend
        .append("rect")
        .attr("x", 0)
        .attr("y", 50)
        .attr("width", 550)
        .attr("height", 20)
        .attr("rx", 20)
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
        .attr("y", 65)
        .attr("width", 380)
        .attr("height", 380);

      //Taken from http://bl.ocks.org/mbostock/7555321
      //Date: 23rd August 2021
      //Wraps SVG text
      function wrap(text, width) {
        text.each(function () {
          var text = d3.select(this),
            words = text.text().split(/\s+/).reverse(),
            word,
            line = [],
            lineNumber = 0,
            lineHeight = 1.4, // ems
            y = text.attr("y"),
            x = text.attr("x"),
            dy = parseFloat(text.attr("dy")),
            tspan = text
              .text(null)
              .append("tspan")
              .attr("x", x)
              .attr("y", y)
              .attr("dy", dy + "em");

          while ((word = words.pop())) {
            line.push(word);
            tspan.text(line.join(" "));
            if (tspan.node().getComputedTextLength() > width) {
              line.pop();
              tspan.text(line.join(" "));
              line = [word];
              tspan = text
                .append("tspan")
                .attr("x", x)
                .attr("y", y)
                .attr("dy", ++lineNumber * lineHeight + dy + "em")
                .text(word);
            }
          }
        });
      }

      infoboxes
        .append("rect")
        .attr("x", 10)
        .attr("y", 0)
        .attr("rx", 20)
        .attr("class", "infobox_rect")
        .style("visibility", function () {
          if (glob.select_count < 1) {
            return "hidden";
          }
        });
      infoboxes
        .append("text")
        .attr("x", 190)
        .attr("y", 20)
        .attr("dy", 0)
        .attr("class", "infobox_title")
        .style("visibility", function () {
          if (glob.select_count < 1) {
            return "hidden";
          }
        })
        .style("text-anchor", "middle")
        .html(function () {
          if (glob.select_count >= 1) {
            return glob.selection_data[0][0].areaname;
          } else {
            return "";
          }
        })
        .call(wrap, 320);

      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 40)
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
              "30" +
              " dy=" +
              "25" +
              "> Population: " +
              glob.selection_data[0][0].population +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "25" +
              " dy=" +
              "20" +
              "> Cases: " +
              glob.selection_data[0][0].cases +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "25" +
              " dy=" +
              "20" +
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
        .attr("y", 125)
        .attr("rx", 20)
        .attr("class", "infobox_rect")
        .style("visibility", function () {
          if (glob.select_count < 2) {
            return "hidden";
          }
        });
      infoboxes
        .append("text")
        .attr("x", 190)
        .attr("y", 145)
        .attr("dy", 0)
        .attr("class", "infobox_title")
        .style("visibility", function () {
          if (glob.select_count < 2) {
            return "hidden";
          }
        })
        .style("text-anchor", "middle")
        .html(function () {
          if (glob.select_count >= 2) {
            return glob.selection_data[1][0].areaname;
          } else {
            return "";
          }
        })
        .call(wrap, 320);

      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 165)
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
              "30" +
              " dy=" +
              "25" +
              "> Population: " +
              glob.selection_data[1][0].population +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "25" +
              " dy=" +
              "20" +
              "> Cases: " +
              glob.selection_data[1][0].cases +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "25" +
              " dy=" +
              "20" +
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
        .attr("y", 250)
        .attr("rx", 20)
        .attr("class", "infobox_rect")
        .style("visibility", function () {
          if (glob.select_count < 3) {
            return "hidden";
          }
        });
      infoboxes
        .append("text")
        .attr("x", 190)
        .attr("y", 270)
        .attr("dy", 0)
        .attr("class", "infobox_title")
        .style("visibility", function () {
          if (glob.select_count < 3) {
            return "hidden";
          }
        })
        .style("text-anchor", "middle")
        .html(function () {
          if (glob.select_count >= 3) {
            return glob.selection_data[2][0].areaname;
          } else {
            return "";
          }
        })
        .call(wrap, 320);

      infoboxes
        .append("text")
        .attr("x", 20)
        .attr("y", 290)
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
              "30" +
              " dy=" +
              "25" +
              "> Population: " +
              glob.selection_data[2][0].population +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "25" +
              " dy=" +
              "20" +
              "> Cases: " +
              glob.selection_data[2][0].cases +
              "</tspan>" +
              "<tspan " +
              "x=" +
              "25" +
              " dy=" +
              "20" +
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
    draw_hist() {
      d3.selectAll("#cr-hist > svg").remove();

      const glob = this;

      // get data column containing case rates
      var data = glob.list.map((row) => {
        return +(row.properties.Case_rate * 100).toFixed(2);
      });

      const hist_svg = d3
        .select("#cr-hist")
        .append("svg")
        .attr("x", 0)
        .attr("y", 20)
        .attr("width", 380)
        .attr("height", 400);

      hist_svg
        .append("rect")
        .attr("x", 10)
        .attr("y", 5)
        .attr("class", "hist_bg");

      var x = d3
        .scaleLinear()
        .domain([0, d3.max(data)])
        .range([0, 300]);

      hist_svg
        .append("g")
        .attr("transform", "translate(" + 50 + "," + 250 + ")")
        .call(d3.axisBottom(x));

      var hist = d3
        .histogram()
        .value(function (d) {
          return d;
        })
        .domain(x.domain())
        .thresholds(x.ticks(8));

      var bins = hist(data);

      var y = d3
        .scaleLinear()
        .range([225, 30])
        .domain([
          0,
          d3.max(bins, function (d) {
            return d.length;
          }),
        ]);

      hist_svg
        .append("g")
        .attr("transform", "translate(" + 50 + "," + 25 + ")")
        .call(d3.axisLeft(y));

      hist_svg
        .append("text")
        .attr("text-anchor", "middle")
        .attr("transform", "translate(" + 270 + "," + 35 + ")")
        .attr("class", "chart_title")
        .text("Case rate distribution");

      var bar = hist_svg
        .selectAll(".bar")
        .data(bins)
        .enter()
        .append("g")
        .attr("transform", function (d) {
          return "translate(" + x(d.x0) + "," + y(d.length) + ")";
        });

      bar
        .append("rect")
        .attr("x", 1)
        .attr("transform", function () {
          return "translate(" + 50 + "," + 23 + ")";
        })
        .attr("width", function (d) {
          return x(d.x1) - x(d.x0) - 1;
        })
        .attr("height", 0)
        .attr("class", function (d) {
          if (d3.min(d) >= glob.range[0]) {
            return "bars";
          } else if (d3.min(d) < glob.range[0] && d3.max(d) > glob.range[0]) {
            return "bars_faded";
          } else {
            return "bars_blank";
          }
        })
        .on("click", function () {
          if (d3.select(this).attr("class").includes("click-indicator-bar")) {
            glob.circles_range = [100, 101];
            glob.circles_count = 0;
            d3.select(this).classed("click-indicator-bar", false);
            glob.init();
          } else {
            glob.circles_range = [
              d3.min(d3.select(this).datum()),
              d3.max(d3.select(this).datum()),
            ];
            glob.circles_count = d3.select(this).datum().length;
            d3.selectAll(".click-indicator-bar").classed(
              "click-indicator-bar",
              false
            );
            d3.select(this).classed("click-indicator-bar", true);
            glob.init();
          }
        })
        .transition()
        .duration(2000)
        .attr("height", function (d) {
          return 225 - y(d.length);
        });

      bar
        .append("text")
        .attr("class", "hist_bar_text")
        .attr("transform", function () {
          return "translate(" + 65 + "," + 15 + ")";
        })
        .text(function (d) {
          return d.length;
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
            return Number(d[0].population);
          })
        ),
      ]);

      bar_g
        .append("g")
        .attr("transform", "translate(" + 25 + "," + 350 + ")")
        .call(d3.axisBottom(x_scale))
        .selectAll("text")
        .attr("class", "bar_axis_text")
        .style("text-anchor", "end")
        .attr("dx", "-1.8em")
        .attr("dy", "-0.8em")
        .attr("transform", "rotate(-90)");

      bar_g
        .append("g")
        .attr("transform", "translate(" + 50 + "," + 40 + ")")
        .call(d3.axisLeft(y_scale).ticks(10))
        .append("text")
        .attr("y", 6)
        .attr("dy", "0.71em")
        .attr("text-anchor", "end")
        .text("Num. of Cases");

      bar_g
        .append("text")
        .attr("text-anchor", "middle")
        .attr("transform", "translate(" + 160 + "," + 60 + ")")
        .attr("class", "chart_title")
        .text("Area population");

      bar_g
        .selectAll(".bar")
        .data(glob.selection_data)
        .enter()
        .append("rect")
        .attr("class", "bars")
        .attr("transform", "translate(" + 25 + "," + 40 + ")")
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
        .duration(1000)
        .attr("y", function (d) {
          return y_scale(d[0].population);
        })
        .attr("height", function (d) {
          return 310 - y_scale(d[0].population);
        });
    },
    draw_radar() {
      d3.selectAll("#chart > svg").remove();

      const glob = this;

      const radial = d3
        .select("#chart")
        .append("svg")
        .attr("x", 0)
        .attr("y", 20)
        .attr("width", 330)
        .attr("height", 450);

      radial.append("rect").attr("x", 5).attr("y", 5).attr("class", "chart_bg");

      var default_data = [
        [
          //Selection 1
          { axis: "Population", value: 0 },
          { axis: "Case rate", value: 0 },
          { axis: "Cases", value: 0 },
        ],
      ];

      var data = [];

      // If no items selected show default chart, otherwise plot
      if (glob.select_count === 0) {
        data = default_data;
      } else {
        var max_pop = d3.max(
          glob.selection_data.map((element) => {
            return Number(element[0].population);
          })
        );

        var max_case_rate = d3.max(
          glob.selection_data.map((element) => {
            return Number(element[0].case_rate);
          })
        );

        var max_cases = d3.max(
          glob.selection_data.map((element) => {
            return Number(element[0].cases);
          })
        );

        for (let i of glob.selection_data) {
          data.push([
            {
              axis: "Population",
              value: Number(i[0].population) / max_pop,
            },
            {
              axis: "Case rate",
              value: Number(i[0].case_rate) / max_case_rate,
            },
            {
              axis: "Cases",
              value: Number(i[0].cases) / max_cases,
            },
          ]);
        }
      }

      // Adapted from 'The Radar Chart Function' written by Nadieh Bremer (VisualCinnamon.com)
      // Which was inspired by the code of alangrafu
      // Date: 18th August 2021
      // https://gist.github.com/nbremer/21746a9668ffdf6d8242#file-radarchart-js

      var cfg = {
        w: 250, //Width of the circle
        h: 250, //Height of the circle
        margin: { top: 5, right: 5, bottom: 5, left: 5 }, //The margins of the SVG
        levels: 5, //How many levels or inner circles should there be drawn
        maxValue: Number(1), //What is the value that the biggest circle will represent
        labelFactor: 1.25, //How much farther than the radius of the outer circle should the labels be placed
        wrapWidth: 60, //The number of pixels after which a label needs to be given a new line
        opacityArea: 0.35, //The opacity of the area of the blob
        dotRadius: 2, //The size of the colored circles of each blog
        opacityCircles: 0.1, //The opacity of the circles of each blob
        strokeWidth: 2, //The width of the stroke around each blob
        roundStrokes: true, //If true the area and stroke will follow a round path (cardinal-closed)
        // color: d3.scaleOrdinal().range(["#EDC951", "#CC333F", "#00A0B0","Black"]), //Color function
        color: d3.scaleOrdinal(d3.schemeSet2), //Color function
      };

      //Taken from http://bl.ocks.org/mbostock/7555321
      //Date: 23rd August 2021
      //Wraps SVG text
      function wrap(text, width) {
        text.each(function () {
          var text = d3.select(this),
            words = text.text().split(/\s+/).reverse(),
            word,
            line = [],
            lineNumber = 0,
            lineHeight = 1.4, // ems
            y = text.attr("y"),
            x = text.attr("x"),
            dy = parseFloat(text.attr("dy")),
            tspan = text
              .text(null)
              .append("tspan")
              .attr("x", x)
              .attr("y", y)
              .attr("dy", dy + "em");

          while ((word = words.pop())) {
            line.push(word);
            tspan.text(line.join(" "));
            if (tspan.node().getComputedTextLength() > width) {
              line.pop();
              tspan.text(line.join(" "));
              line = [word];
              tspan = text
                .append("tspan")
                .attr("x", x)
                .attr("y", y)
                .attr("dy", ++lineNumber * lineHeight + dy + "em")
                .text(word);
            }
          }
        });
      }

      // If the supplied maxValue is smaller than the actual one, replace by the max in the data
      // var maxValue = Math.max(
      //   cfg.maxValue,
      //   d3.max(data, function (i) {
      //     return d3.max(
      //       i.map(function (o) {
      //         return o.value;
      //       })
      //     );
      //   })
      // );

      var allAxis = data[0].map(function (i) {
          return i.axis;
        }), //Names of axes
        total = allAxis.length, // Number of axes
        radius = Math.min(cfg.w / 2, cfg.h / 2), //Radius of the outermost circle
        angleSlice = (Math.PI * 2) / total; //The width in radians of each "slice"

      //Scale for the radius
      var rScale = d3
        .scaleLinear()
        .range([0, radius])
        .domain([0, cfg.maxValue]);

      //Append a g element
      var g = radial
        .append("g")
        .attr(
          "transform",
          "translate(" +
            (cfg.w / 2 + cfg.margin.left + 33) +
            "," +
            (cfg.h / 2 + cfg.margin.top + 50) +
            ")"
        );

      //Filter for the outside glow
      var filter = g.append("defs").append("filter").attr("id", "glow");
      filter
        .append("feGaussianBlur")
        .attr("stdDeviation", "3.5")
        .attr("result", "coloredBlur");
      var feMerge = filter.append("feMerge");
      feMerge.append("feMergeNode").attr("in", "coloredBlur");
      feMerge.append("feMergeNode").attr("in", "SourceGraphic");

      //Wrapper for the circular grid & axes
      var axisGrid = g.append("g").attr("class", "axisWrapper");

      //Draw the background circle levels
      axisGrid
        .selectAll(".levels")
        .data(d3.range(1, cfg.levels + 1).reverse())
        .enter()
        .append("circle")
        .attr("class", "gridCircle")
        .attr("r", function (d) {
          return (radius / cfg.levels) * d;
        })
        .style("fill", "#CDCDCD")
        .style("stroke", "#CDCDCD")
        .style("fill-opacity", cfg.opacityCircles)
        .style("filter", "url(#glow)");

      //Text indicating at what value / % each level is
      axisGrid
        .selectAll(".axisLabel")
        .data(d3.range(1, cfg.levels + 1).reverse())
        .enter()
        .append("text")
        .attr("class", "axisLabel")
        .attr("x", 4)
        .attr("y", function (d) {
          return (-d * radius) / cfg.levels;
        })
        .attr("dy", "0.4em")
        .style("font-size", "11px")
        .attr("fill", "#737373")
        .text(function (d) {
          return d3.format(".2")((cfg.maxValue * d) / cfg.levels);
        });

      //Create the straight lines radiating outward from the center
      var axis = axisGrid
        .selectAll(".axis")
        .data(allAxis)
        .enter()
        .append("g")
        .attr("class", "axis2");

      //Append the lines
      axis
        .append("line")
        .attr("x1", 0)
        .attr("y1", 0)
        .attr("x2", function (d, i) {
          return Math.round(
            rScale(cfg.maxValue * 1.1) * Math.sin(angleSlice * i)
          );
        })
        .attr("y2", function (d, i) {
          return Math.round(
            rScale(cfg.maxValue * 1.1) * -Math.cos(angleSlice * i)
          );
        })
        .attr("class", "line")
        .style("stroke", "#C6C6C6")
        .style("stroke-width", "2px");

      //Append the labels at each axis
      axis
        .append("text")
        .attr("class", "legend")
        .style("font-size", "11px")
        .attr("text-anchor", "middle")
        .attr("dy", "0.35em")
        .attr("x", function (d, i) {
          return (
            rScale(cfg.maxValue * cfg.labelFactor) * Math.sin(angleSlice * i)
          );
        })
        .attr("y", function (d, i) {
          return (
            rScale(cfg.maxValue * cfg.labelFactor) * -Math.cos(angleSlice * i)
          );
        })
        .text(function (d) {
          return d;
        })
        .call(wrap, cfg.wrapWidth);

      // The radial line function
      var radarLine = d3
        .lineRadial()
        .curve(d3.curveBasisClosed)
        .radius(function (d) {
          return rScale(d.value);
        })
        .angle(function (d, i) {
          return i * angleSlice;
        });

      if (cfg.roundStrokes) {
        radarLine.curve(d3.curveCardinalClosed);
      }

      //Create a wrapper for the blobs
      var blobWrapper = g
        .selectAll(".radarWrapper")
        .data(data)
        .enter()
        .append("g")
        .attr("class", "radarWrapper");

      //Append the backgrounds
      blobWrapper
        .append("path")
        .attr("class", "radarArea")
        .attr("d", function (d) {
          return radarLine(d);
        })
        .style("fill", function (d, i) {
          return cfg.color(i);
        })
        .style("fill-opacity", cfg.opacityArea)
        .on("mouseover", function () {
          //Dim all blobs
          d3.selectAll(".radarArea")
            .transition()
            .duration(150)
            .style("fill-opacity", 0.1);
          //Bring back the hovered over blob
          d3.select(this).transition().duration(150).style("fill-opacity", 0.8);
        })
        .on("mouseout", function () {
          //Bring back all blobs
          d3.selectAll(".radarArea")
            .transition()
            .duration(150)
            .style("fill-opacity", cfg.opacityArea);
        });

      //Create the outlines
      blobWrapper
        .append("path")
        .attr("class", "radarStroke")
        .attr("d", function (d) {
          return radarLine(d);
        })
        .style("stroke-width", cfg.strokeWidth + "px")
        .style("stroke", function (d, i) {
          return cfg.color(i);
        })
        .style("fill", "none")
        .style("filter", "url(#glow)");

      //Append the circles
      blobWrapper
        .selectAll(".radarCircle")
        .data(function (d) {
          return d;
        })
        .enter()
        .append("circle")
        .attr("class", "radarCircle")
        .attr("r", cfg.dotRadius)
        .attr("cx", function (d, i) {
          return rScale(d.value) * Math.sin(angleSlice * i);
        })
        .attr("cy", function (d, i) {
          return rScale(d.value) * -Math.cos(angleSlice * i);
        })
        .style("fill", function (d, i, j) {
          return cfg.color(j);
        })
        .style("fill-opacity", 0.8);
    },
    chart_toggle() {
      const glob = this;
      glob.show_bar = !glob.show_bar;
      if (glob.show_bar === true) {
        this.draw_bar();
      } else {
        this.draw_radar();
      }
    },
  },
  data() {
    return {
      range: [0, 1.8],
      circles_range: [100, 101],
      select_count: 0,
      selection_data: [],
      show_bar: true,
      col_select: d3.interpolatePlasma,
      col_options: [
        { value: d3.interpolatePlasma, text: "Plasma (default)" },
        { value: d3.interpolateCividis, text: "Cividis" },
        { value: d3.interpolateViridis, text: "Viridis" },
        { value: d3.interpolateWarm, text: "Warm" },
        {
          value: d3
            .scaleLinear()
            .domain([0, 0.25, 0.5, 0.75, 1])
            .range(["#601A4A", "#EE442F", "#CCBE9F", "#E0DCD3", "#ABC3C9"]),
          text: "Deut/Prot friendly",
        },
        {
          value: d3
            .scaleLinear()
            .domain([0, 0.25, 0.5, 0.75, 1])
            .range(["Blue", "Purple", "Red", "Orange", "Yellow"]),
          text: "Trit friendly",
        },
      ],
    };
  },
  watch: {
    async range() {
      await this.init();
      this.draw_hist();
    },
    col_select() {
      this.$nextTick(() => this.init());
    },
    select_count() {
      this.redraw_infoboxes();
      if (this.select_count > 0) {
        if (this.show_bar === true) {
          this.draw_bar();
        } else {
          this.draw_radar();
        }
      }
    },
  },
  async mounted() {
    await this.init();
    this.draw_hist();
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
  font-size: 18pt;
  fill: #000;
}
.headers2 {
  font-size: 12pt;
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
  stroke: rgb(0, 0, 0);
}
.click-indicator-bar {
  stroke-width: 3 !important;
  stroke: rgb(0, 0, 0);
}
.highlight {
  stroke-width: 2;
  stroke: #000;
}
.circle {
  visibility: visible;
  fill: rgb(1, 65, 1);
}
.circle_hide {
  visibility: hidden;
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
  height: 115px;
  width: 360px;
  fill: rgb(255, 255, 255);
  stroke: #000;
  stroke-width: 2;
}
.infobox_title {
  fill: rgb(0, 0, 0);
  font-size: 12pt;
  font-weight: bold;
}
.infobox_text {
  fill: #000;
  font-size: 11pt;
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
.mybtn3 {
  border-color: #000 !important;
  border-width: 2.5px !important;
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
.chart_title {
  text-decoration: underline;
  font-weight: bold;
  font-size: 12pt !important;
}
.bars {
  fill: #69b3a2;
  stroke-width: 1;
  stroke: #000;
}
.bars_faded {
  fill: #69b3a25b;
  stroke-width: 1;
  stroke: #000;
}
.bars_blank {
  fill: #69b3a200;
  stroke-width: 1;
  stroke: #000;
}
.bar_axis_text {
  font-size: 7pt !important;
}
.hist_bg {
  position: absolute;
  height: 280px;
  width: 360px;
  fill: rgb(255, 255, 255);
  stroke: #000;
  stroke-width: 2.5;
}
.hist_bar_text {
  font-size: 8pt !important;
  font-weight: bold;
  text-anchor: middle;
}
</style>
