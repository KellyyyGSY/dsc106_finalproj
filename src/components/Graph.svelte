<script>
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  import { scaleLinear, scaleTime } from 'd3-scale';
  import { line, curveMonotoneX } from 'd3-shape';

  export let index, width, height;

  let data, cpiData;
  let cpiSvg;
  let animatedLineSvg;

  onMount(async () => {
    const res = await fetch('US10_Year_Bond_Yield_20-24.csv');
    const csv = await res.text();
    data = d3.csvParse(csv, d3.autoType);

    const cpiRes = await fetch('monthly_inflation.csv');
    const cpiCsv = await cpiRes.text();
    cpiData = d3.csvParse(cpiCsv, d => ({
      date: d3.timeParse("%Y/%m")(d.Date),
      percentageChange: +d["Percent_Change"]
    }));
  });

  afterUpdate(() => {
    if (index === 5) {
      drawAnimatedLine();
    } else {
      if (animatedLineSvg) animatedLineSvg.remove();
    }

    if (index >= 16 && index <= 19) {
      drawCPIPCBG();
    } else {
      if (cpiSvg) cpiSvg.remove();
    }
  });

  function drawAnimatedLine() {
    const margin = { top: 80, right: 0, bottom: 0, left: 0 };
    const svgWidth = 1350 - margin.left - margin.right;
    const svgHeight = 550;

    // Remove existing svg if exists
    if (animatedLineSvg) animatedLineSvg.remove();

    animatedLineSvg = d3.select('.graph')
      .append('svg')
      .attr('width', svgWidth + margin.left + margin.right)
      .attr('height', svgHeight + margin.top + margin.bottom)
      .append('g')
      .attr('transform', 'translate(' + margin.left + ',' + margin.top + ')');

    const x = scaleTime()
      .domain(d3.extent(data, d => new Date(d.Date)))
      .range([0, svgWidth]);
      
    const y = scaleLinear()
      .domain([0, d3.max(data, d => d.Price)])
      .range([svgHeight, 0]);

    const lineGenerator = line()
      .x(d => x(new Date(d.Date)))
      .y(d => y(d.Price))
      .curve(curveMonotoneX);

    // Append a 'path' element to the SVG
    const path = animatedLineSvg.append('path')
      .datum(data)
      .attr('fill', 'none')
      .attr('stroke', '#ccc')
      .attr('stroke-width', 1.5)
      .attr('d', lineGenerator(data)); // Set initial path data to start position

    // Get the total length of the path
    const totalLength = path.node().getTotalLength();

    // Transition the path to its initial position
    path.attr('stroke-dasharray', totalLength + ',0');

    // Transition the path to its final position
    path.transition()
      .duration(6000) // Adjust duration as needed
      .attr('stroke-dasharray', '0,' + totalLength);
  }

  function drawCPIPCBG() {
    const margin = { top: 50, right: 150, bottom: 200, left: 90 };
    const svgWidth = 1400;
    const svgHeight = 700;
    const plotWidth = svgWidth - margin.left - margin.right;
    const plotHeight = svgHeight - margin.top - margin.bottom;

    // Remove existing svg if exists
    if (cpiSvg) cpiSvg.remove();

    // Define the SVG container
    cpiSvg = d3.select(".graph")
      .append("svg")
      .attr("width", svgWidth)
      .attr("height", svgHeight)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    // Assuming your date parsing has been done correctly when loading the data
    const xScale = d3.scaleTime()
      .domain(d3.extent(cpiData, d => d.date))
      .range([0, plotWidth]);

    const yScale = d3.scaleLinear()
      .domain([-3, d3.max(cpiData, d => d.percentageChange) + 1])
      .range([plotHeight, 0]);

    // Define axes
    const xAxis = d3.axisBottom(xScale).tickFormat(d => d.getFullYear()); // Format ticks to display only the year
    const yAxis = d3.axisLeft(yScale);

    // Append axes to the SVG
    cpiSvg.append("g")
      .attr("transform", `translate(0,${plotHeight})`)
      .call(xAxis);

    cpiSvg.append("g")
      .call(yAxis);

    cpiSvg.append("g")
      .attr("class", "grid")
      .call(d3.axisLeft(yScale)
        .tickSize(-plotWidth)
        .tickFormat("")
      )
      .selectAll(".tick line")
      .attr("stroke", "#ccc");

    cpiSvg.append("line")
      .attr("x1", 0)
      .attr("y1", yScale(0))
      .attr("x2", plotWidth)
      .attr("y2", yScale(0))
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    // Add y-axis label
    cpiSvg.append("text")
      .attr("class", "y label")
      .attr("text-anchor", "end")
      .attr("fill", "white")
      .attr("x", -margin.left - 80)
      .attr("y", -margin.left + 40) // Adjusted positioning
      .attr("dy", ".75em")
      .attr("transform", "rotate(-90)")
      .text("Percentage Change");

    // Area generator for CPI
    const areaCPI = d3.area()
      .x(d => xScale(d.date))
      .y0(yScale(0)) // Set y0 to y-coordinate of the line y=0
      .y1(d => yScale(d.percentageChange))
      .curve(d3.curveMonotoneX);

    // Append the area for CPI
    cpiSvg.append("path")
      .datum(cpiData)
      .attr("fill", "blue")
      .attr("opacity", 0.2)
      .attr("d", areaCPI);

    // Line generator for CPI
    const lineCPI = d3.line()
      .x(d => xScale(d.date))
      .y(d => yScale(d.percentageChange))
      .curve(d3.curveMonotoneX);

    // Append the path for CPI
    cpiSvg.append("path")
      .datum(cpiData)
      .attr("fill", "none")
      .attr("stroke", "blue")
      .attr("stroke-width", 2)
      .attr("d", lineCPI);
  }

</script>

<svg class="graph" width={width} height={height}>
  // do not remove anything in this section
</svg>

<style>
</style>