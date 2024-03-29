<script>
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  import { scaleLinear, scaleTime } from 'd3-scale';
  import { line, curveMonotoneX } from 'd3-shape';
  import { axisBottom, axisLeft } from 'd3-axis';

  export let index, width, height;

  let data, cpiData;
  let animatedLineSvg;
  let bondSvg;
  let bondSvg2;
  let cpiSvg;

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
    if (index === 30) {
      drawAnimatedLine();
    } else {
      if (animatedLineSvg) animatedLineSvg.remove();
    }

    if (index >= 6 && index <= 9) {
      drawLinePlot();
    } else {
      if (bondSvg) bondSvg.remove();
    }

    if (index >= 10 && index <= 13) {
      drawLinePlot2();
    } else {
      if (bondSvg2) bondSvg2.remove();
    }

    if (index >= 24 && index <= 28) {
      drawCPIPCBG();
    } else {
      if (cpiSvg) cpiSvg.remove();
    }
  });

  function drawAnimatedLine() {
    const margin = { top: 20, right: 0, bottom: 0, left: 0 };
    const svgWidth = window.innerWidth;
    const svgHeight = window.innerHeight * 0.95;

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

  function drawLinePlot() {
    const margin = { top: 100, right: 70, bottom: 100, left: 70 };
    const width = window.innerWidth - margin.left - margin.right;
    const height = window.innerHeight - margin.top - margin.bottom;

    // Filter data based on the range [2016, 2024]
    const filteredData = data.filter(d => new Date(d.Date) >= new Date('2018-01-01') && new Date(d.Date) <= new Date('2024-12-31'));

    // Remove existing svg if exists
    if (bondSvg) bondSvg.remove();

    bondSvg = d3.select(".graph")
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom)
        .append("g")
        .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

    const x = d3.scaleTime()
        .domain(d3.extent(filteredData, d => new Date(d.Date)))
        .range([0, width]);

    const y = d3.scaleLinear()
        .domain([0, d3.max(filteredData, d => d.Price)])
        .range([height, 0]);

    const xAxis = d3.axisBottom(x);
    const yAxis = d3.axisLeft(y);

    bondSvg.append("g")
        .attr("class", "x-axis")
        .attr("transform", "translate(0," + height + ")")
        .call(xAxis);

    bondSvg.append("g")
        .attr("class", "y-axis")
        .call(yAxis);

    bondSvg.selectAll(".x-axis line")
        .attr("stroke", "#ccc");
    bondSvg.selectAll(".y-axis line")
        .attr("stroke", "#ccc");

    bondSvg.append("g")
        .attr("class", "grid")
        .call(d3.axisLeft(y)
            .tickSize(-width)
            .tickFormat("")
        )
        .selectAll(".tick line")
        .attr("stroke", "#ccc");

    bondSvg.append("text")
        .attr("class", "y label")
        .attr("text-anchor", "end")
        .attr("fill", "white")
        .attr("x", -margin.left - 150)
        .attr("y", -margin.left + 20) // Adjusted positioning
        .attr("dy", ".75em")
        .attr("transform", "rotate(-90)")
        .text("Price");

    const lineGenerator = d3.line()
        .x(d => x(new Date(d.Date)))
        .y(d => y(d.Price))
        .curve(d3.curveMonotoneX);

    bondSvg.append("path")
        .datum(filteredData)
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1.5)
        .attr("d", lineGenerator);
    
    // Add annotations
    const annotations = [
        { date: new Date("07/01/2020"), price: 2.5, text: "covid-19 pandemic:\n2020 low point" },
        // add more if needed
    ];

    // Define arrowhead marker
    bondSvg.append("defs").append("marker")
      .attr("id", "arrowhead")
      .attr("viewBox", "0 -5 10 10")
      .attr("refX", 8)
      .attr("refY", 0)
      .attr("orient", "auto")
      .attr("markerWidth", 6)
      .attr("markerHeight", 6)
      .append("path")
      .attr("d", "M0,-5L10,0L0,5")
      .attr("fill", "white");

    // Add arrows and annotations
    annotations.forEach(annotation => {
      const xPos = x(annotation.date);
      const yPos = y(annotation.price);

      // Add annotation text
      const lines = annotation.text.split('\n');
      lines.forEach((line, i) => {
        bondSvg.append("text")
          .attr("x", xPos)
          .attr("y", yPos + i * 20) // Adjust dy for line spacing
          .attr("dy", "-0.5em")
          .text(line)
          .attr("fill", "white");
      });

      // Find the closest point on the line to the annotation
      const closestPoint = findClosestPointOnLine(data, { x: xPos, y: yPos });

      // Add arrow line
      bondSvg.append("line")
        .attr("x1", xPos)
        .attr("y1", yPos)
        .attr("x2", closestPoint.x)
        .attr("y2", closestPoint.y)
        .attr("stroke", "white")
        .attr("stroke-width", 1)
        .attr("marker-end", "url(#arrowhead)");
    });

    // Function to find the closest point on the line
    function findClosestPointOnLine(data, point) {
      let minDistance = Infinity;
      let closestPoint = null;

      data.forEach(d => {
        const distance = Math.abs(point.x - x(new Date(d.Date)));
        if (distance < minDistance) {
          minDistance = distance;
          closestPoint = { x: x(new Date(d.Date)), y: y(d.Price) };
        }
      });

      return closestPoint;
    }
  }

  function drawLinePlot2() {
    const margin = { top: 100, right: 70, bottom: 100, left: 70 };
    const width = window.innerWidth - margin.left - margin.right;
    const height = window.innerHeight - margin.top - margin.bottom;

    // Filter data based on the range [2016, 2024]
    const filteredData = data.filter(d => new Date(d.Date) >= new Date('2018-01-01') && new Date(d.Date) <= new Date('2024-12-31'));

    // Remove existing svg if exists
    if (bondSvg2) bondSvg2.remove();

    bondSvg2 = d3.select(".graph")
        .append("svg")
        .attr("width", width + margin.left + margin.right)
        .attr("height", height + margin.top + margin.bottom)
        .append("g")
        .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

    const x = d3.scaleTime()
        .domain(d3.extent(filteredData, d => new Date(d.Date)))
        .range([0, width]);

    const y = d3.scaleLinear()
        .domain([0, d3.max(filteredData, d => d.Price)])
        .range([height, 0]);

    const xAxis = d3.axisBottom(x);
    const yAxis = d3.axisLeft(y);

    bondSvg2.append("g")
        .attr("class", "x-axis")
        .attr("transform", "translate(0," + height + ")")
        .call(xAxis);

    bondSvg2.append("g")
        .attr("class", "y-axis")
        .call(yAxis);

    bondSvg2.selectAll(".x-axis line")
        .attr("stroke", "#ccc");
    bondSvg2.selectAll(".y-axis line")
        .attr("stroke", "#ccc");

    bondSvg2.append("g")
        .attr("class", "grid")
        .call(d3.axisLeft(y)
            .tickSize(-width)
            .tickFormat("")
        )
        .selectAll(".tick line")
        .attr("stroke", "#ccc");

    bondSvg2.append("text")
        .attr("class", "y label")
        .attr("text-anchor", "end")
        .attr("fill", "white")
        .attr("x", -margin.left - 150)
        .attr("y", -margin.left + 20) // Adjusted positioning
        .attr("dy", ".75em")
        .attr("transform", "rotate(-90)")
        .text("Price");

    const lineGenerator = d3.line()
        .x(d => x(new Date(d.Date)))
        .y(d => y(d.Price))
        .curve(d3.curveMonotoneX);

    bondSvg2.append("path")
        .datum(filteredData)
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1.5)
        .attr("d", lineGenerator);
    
    // Add annotations
    const annotations = [
        { date: new Date("01/01/2023"), price: 2.0, text: "post-pandemic:\n2023 economic rebound" }
        // add more if needed
    ];

    // Define arrowhead marker
    bondSvg2.append("defs").append("marker")
      .attr("id", "arrowhead")
      .attr("viewBox", "0 -5 10 10")
      .attr("refX", 8)
      .attr("refY", 0)
      .attr("orient", "auto")
      .attr("markerWidth", 6)
      .attr("markerHeight", 6)
      .append("path")
      .attr("d", "M0,-5L10,0L0,5")
      .attr("fill", "white");

    // Add arrows and annotations
    annotations.forEach(annotation => {
      const xPos = x(annotation.date);
      const yPos = y(annotation.price);

      // Add annotation text
      const lines = annotation.text.split('\n');
      lines.forEach((line, i) => {
        bondSvg2.append("text")
          .attr("x", xPos)
          .attr("y", yPos + i * 20) // Adjust dy for line spacing
          .attr("dy", "-0.5em")
          .text(line)
          .attr("fill", "white");
      });

      // Find the closest point on the line to the annotation
      const closestPoint = findClosestPointOnLine(data, { x: xPos, y: yPos });

      // Add arrow line
      bondSvg2.append("line")
        .attr("x1", xPos)
        .attr("y1", yPos)
        .attr("x2", closestPoint.x)
        .attr("y2", closestPoint.y)
        .attr("stroke", "white")
        .attr("stroke-width", 1)
        .attr("marker-end", "url(#arrowhead)");
    });

    // Function to find the closest point on the line
    function findClosestPointOnLine(data, point) {
      let minDistance = Infinity;
      let closestPoint = null;

      data.forEach(d => {
        const distance = Math.abs(point.x - x(new Date(d.Date)));
        if (distance < minDistance) {
          minDistance = distance;
          closestPoint = { x: x(new Date(d.Date)), y: y(d.Price) };
        }
      });

      return closestPoint;
    }
  }


  function drawCPIPCBG() {
    const margin = { top: 100, right: 150, bottom: 100, left: 90 };
    const plotWidth = window.innerWidth - margin.left - margin.right;
    const plotHeight = window.innerHeight - margin.top - margin.bottom;

    // Remove existing svg if exists
    if (cpiSvg) cpiSvg.remove();

    // Define the SVG container
    cpiSvg = d3.select(".graph")
      .append("svg")
      .attr("width", plotWidth + margin.left + margin.right)
      .attr("height", plotHeight + margin.top + margin.bottom)
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

    // Add annotations
    const annotations = [
        { date: new Date("2008/07"), percentChange: 7.2, text: "Global Financial Crisis:\nSep 2008" },
        { date: new Date("2020/11"), percentChange: 3.8, text: "First vaccinations:\nDec 14th 2020" }
        // add more if needed
    ];

    // Define arrowhead marker
    cpiSvg.append("defs").append("marker")
      .attr("id", "arrowhead")
      .attr("viewBox", "0 -5 10 10")
      .attr("refX", 8)
      .attr("refY", 0)
      .attr("orient", "auto")
      .attr("markerWidth", 6)
      .attr("markerHeight", 6)
      .append("path")
      .attr("d", "M0,-5L10,0L0,5")
      .attr("fill", "white");

    // Add arrows and annotations
    annotations.forEach(annotation => {
      const xPos = xScale(annotation.date);
      const yPos = yScale(annotation.percentChange);

      // Split text into lines
      const lines = annotation.text.split('\n');

      // Add annotation text
      lines.forEach((line, i) => {
        cpiSvg.append("text")
          .attr("x", xPos)
          .attr("y", yPos + i * 20) // Adjust dy for line spacing
          .attr("dy", "-0.5em")
          .text(line)
          .attr("fill", "white");
      });

      // Find the closest point on the line to the annotation
      const closestPoint = findClosestPointOnLine(cpiData, { x: xPos, y: yPos });

      // Add arrow line
      cpiSvg.append("line")
        .attr("x1", xPos)
        .attr("y1", yPos)
        .attr("x2", closestPoint.x)
        .attr("y2", closestPoint.y)
        .attr("stroke", "white")
        .attr("stroke-width", 1)
        .attr("marker-end", "url(#arrowhead)");
    });

    // Function to find the closest point on the line
    function findClosestPointOnLine(data, point) {
      let minDistance = Infinity;
      let closestPoint = null;

      data.forEach(d => {
        const distance = Math.abs(point.x - xScale(d.date));
        if (distance < minDistance) {
          minDistance = distance;
          closestPoint = { x: xScale(d.date), y: yScale(d.percentageChange) };
        }
      });

      return closestPoint;
    }
  }

</script>

<svg class="graph" width={width} height={height}>
  // do not remove anything in this section
</svg>

<style>
</style>