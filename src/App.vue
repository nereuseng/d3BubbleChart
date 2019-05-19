<template>
  <main id="app">
    <div class="type">
      <button
        :class="{active: currentType === 'all'}"
        class="type-item"
        @click="onTypeClicked($event, 'all')"
      >All</button>
      <button
        :class="{active: currentType === 'stall'}"
        class="type-item"
        @click="onTypeClicked($event, 'stall')"
      >Stall</button>
      <button
        :class="{active: currentType === 'location'}"
        class="type-item"
        @click="onTypeClicked($event, 'location')"
      >Location</button>
    </div>

    <div class="container">
      <div class="chart" ref="chart"></div>
      <Popup
        :style="{ top: `${popupLocation.y}px`, left: `${popupLocation.x}px`}"
        :class="{ active: popupData }"
        :data="popupData"
      />
    </div>
  </main>
</template>

<script>
import * as d3 from "d3";
import Popup from "@/components/Popup.vue";
import debounce from "lodash.debounce";
// import axios from "axios";
// import dataCsv from "../information/data.csv";

export default {
  components: {
    Popup
  },

  data() {
    return {
      dataset: {},
      colorSets: ["#5580c8", "#8bd8b8", "#5ba694", "#bf7ca8"],
      stallColorSet: {
        "Stall A": 0,
        "Stall B": 1,
        "Stall C": 2,
        "Stall D": 3
      },
      popupData: undefined,
      popupLocation: {
        x: NaN,
        y: NaN
      },
      currentType: "all",
      simulation: null,
      diameter: NaN,
      dots: null
    };
  },

  async mounted() {
    window.d3 = d3;
    this.diameter = this.getDiameter();
    this.dataset = await this.getCsvData();
    this.renderD3();

    window.addEventListener(
      "resize",
      debounce(() => {
        this.diameter = this.getDiameter();

        // set SVG height
        d3.select("#bubbleChart")
          .attr("width", this.diameter)
          .attr("height", this.diameter);

        // resize dots
        this.dots.selectAll("circle").attr("r", d => this.rscale(d.price));

        this.checkDotsText();

        // rerender the force layout
        this.splitBubbles(this.currentType);
      }, 300)
    );
  },

  methods: {
    getCsvData() {
      return d3.csv("/data.csv").then(data => data);
    },

    changeNetwork() {
      d3.selectAll("g.dot").attr("transform", d => `translate(${d.x}, ${d.y})`);
    },

    rscale(number) {
      let dotsSize = null;
      if (window.innerWidth >= 1024) {
        dotsSize = [15, 40];
      } else if (window.innerWidth >= 768) {
        dotsSize = [10, 30];
      } else if (window.innerWidth >= 414) {
        dotsSize = [10, 20];
      } else {
        dotsSize = [5, 15];
      }

      const rscale = d3
        .scaleLinear()
        .domain([0, d3.max(this.dataset, d => +d.price)])
        .range(dotsSize);
      return rscale(number);
    },

    renderD3() {
      const color = d3.scaleOrdinal(this.colorSets);
      let self = this;

      let svg = d3
        .select(this.$refs.chart)
        .append("svg")
        .attr("width", this.diameter)
        .attr("height", this.diameter)
        .attr("id", "bubbleChart")
        .attr("class", "chart")
        .on("click", this.svgBgOnclick);

      this.dots = svg
        .selectAll(".dot")
        .data(this.dataset)
        .enter()
        .append("g")
        .attr("class", "dot");

      this.dots
        .append("circle")
        .attr("r", d => this.rscale(d.price))
        .attr("fill", d => color(this.stallColorSet[d.stall]))
        .on("click", this.showPopup)
        .on("mouseover", this.showPopup)
        .on("mouseleave", this.onMouseLeave);

      this.simulation = d3
        .forceSimulation()
        .force("charge", d3.forceManyBody().strength(1));
      this.simulation.force("x", this.createAllForces().x);
      this.simulation
        .force("y", this.createAllForces().y)
        .force("collision", d3.forceCollide(d => this.rscale(d.price) + 2))
        .nodes(this.dataset)
        .on("tick", this.changeNetwork);

      this.checkDotsText();

      d3.select(self.frameElement).style("height", this.diameter + "px");
    },

    showPopup(d) {
      // console.log("kontinentalist: showPopup -> d", d);
      const { name, location, price, stall } = d;
      this.popupData = { name, location, price, stall };
      this.popupLocation = {
        x: d3.event.pageX + 30,
        y: d3.event.pageY + 30
      };
    },

    svgBgOnclick() {
      const svgBg = d3.select("#bubbleChart")._groups[0][0];
      const isSvgBg = svgBg === d3.event.target;
      console.log("TCL: hidePopup -> isSvgBg", isSvgBg);
      if (isSvgBg) {
        this.clearPopupData();
      }
    },

    onMouseLeave() {
      this.clearPopupData();
    },

    onTypeClicked(e, type) {
      console.log(e, type);
      this.setCurrentType(type);
      this.splitBubbles(type);
    },

    checkDotsText() {
      if (window.innerWidth > 768) {
        this.dots.selectAll("text").remove();
        this.setDotsText();
      } else {
        this.dots.selectAll("text").remove();
      }
    },

    setDotsText() {
      this.dots
        .append("text")
        .attr("dy", ".2em")
        .style("text-anchor", "middle")
        .text(d => d.name.substring(0, this.rscale(d.price) / 3))
        .attr("font-family", "sans-serif")
        .attr("font-size", d => this.rscale(d.price) / 3.5)
        .attr("fill", "white");
    },

    setCurrentType(type) {
      this.currentType = type;
    },

    getTypeMovement(type) {
      const typeMovements = {
        all: this.createAllForces(),
        stall: this.createStallForces(),
        location: this.createLocationForces()
      };
      return typeMovements[type];
    },

    createAllForces() {
      return {
        x: d3
          .forceX()
          .strength(0.3)
          .x(this.diameter / 2),
        y: d3
          .forceY()
          .strength(0.3)
          .y(this.diameter / 2)
      };
    },

    createStallForces() {
      return {
        x: d3
          .forceX()
          .strength(0.7)
          .x(this.createStallForceX),
        y: d3
          .forceY()
          .strength(0.7)
          .y(this.createStallForceY)
      };
    },

    createStallForceX(d) {
      if (d.stall === "Stall A" || d.stall === "Stall D") {
        return this.diameter / 4;
      }
      if (d.stall === "Stall B") {
        return this.diameter / 2;
      }
      if (d.stall === "Stall C") {
        return (this.diameter / 4) * 3;
      }
    },

    createStallForceY(d) {
      if (d.stall === "Stall D") {
        return (this.diameter / 4) * 3;
      }
      return this.diameter / 4;
    },

    createLocationForces() {
      return {
        x: d3
          .forceX()
          .strength(0.7)
          .x(this.createLocationForceX),
        y: d3
          .forceY()
          .strength(0.7)
          .y(this.createLocationForceY)
      };
    },

    createLocationForceY() {
      return this.diameter / 3.5;
    },

    createLocationForceX(d) {
      if (d.location === "Jurong") {
        return this.diameter / 5;
      }
      if (d.location === "Orchard") {
        return this.diameter / 2;
      }
      if (d.location === "Bt Merah") {
        return (this.diameter / 5) * 4;
      }
    },

    hideTitles() {
      d3.select("#bubbleChart")
        .selectAll(".title")
        .remove();
    },

    showTitles(type) {
      const capitalizeType = type.charAt(0).toUpperCase() + type.slice(1);
      const titles = d3.map(this.dataset, d => d[type]).keys();
      // console.log("Kontinentalist: showTitles -> titles", titles);

      const formattedDatums = titles.map(e => ({ [type]: e }));
      const titlePositionX = formattedDatums.map(d =>
        this[`create${capitalizeType}ForceX`](d)
      );
      const titlePositionY = formattedDatums.map(d =>
        this[`create${capitalizeType}ForceY`](d)
      );

      const titleNodes = d3
        .select("#bubbleChart")
        .selectAll(".title")
        .data(titles);

      titleNodes
        .enter()
        .append("text")
        .attr("class", "title")
        .merge(titleNodes)
        .attr("x", d => titlePositionX[titles.indexOf(d)])
        .attr(
          "y",
          d => titlePositionY[titles.indexOf(d)] - this.diameter * 0.15
        )
        .attr("text-anchor", "middle")
        .text(d => d);
    },

    splitBubbles(type) {
      const typeMovementHandler = this.getTypeMovement(type);

      if (type === "all") {
        this.hideTitles();
      } else {
        this.hideTitles();
        this.showTitles(type);
      }

      this.simulation
        .force("center", null)
        .force("collision", d3.forceCollide(d => this.rscale(d.price) + 2))
        .alphaDecay(0.0005)
        .velocityDecay(0.6);
      this.simulation.force("x", typeMovementHandler.x);
      this.simulation.force("y", typeMovementHandler.y);
      this.simulation.alpha(0.1).restart();
    },

    getDiameter() {
      const chart = this.$refs.chart;
      const oldChart = d3.select("#bubbleChart");
      const containerWidth =
        (chart && chart.clientWidth) || oldChart.clientWidth;
      return Math.min(containerWidth, 800);
    },

    clearPopupData() {
      this.popupData = undefined;
    }
  }
};
</script>

<style lang=stylus>
@import "./assets/style/reset.styl"

$orange = #f2784b

#app
  width 100vw
  height 100vh
  background #f8f4ef
  padding 10px
  overflow hidden

.container
  width 100%

.chart
  background #f7f7f7
  margin-top 10px
  display flex
  justify-content center

.type
  display flex

  @media screen and (min-width: 768px)
    padding-left 10%

  // type-item
  &-item
    background #fff
    color #ccc
    padding 0 8px
    height 36px
    line-height 36px
    border 0
    width 33.33%

    &.active
      color $orange

    @media screen and (min-width: 768px)
      width auto

    & + &
      border-left 1px solid #eee

.popup
  position absolute

.title
  font-family "LL Akkurat Pro", Helvetica, sans-serif
  font-weight 600
  font-size 18px
</style>
