<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { scaleLinear, scaleTime } from 'd3-scale';
  import { line, curveMonotoneX } from 'd3-shape';
  import { axisBottom, axisLeft } from 'd3-axis';

  import Graph from "./Graph.svelte";
  import Scroller from "@sveltejs/svelte-scroller";

  let count, index, offset, progress;
  let width, height;

  let data;
  onMount(async () => {
    const res = await fetch('US10_Year_Bond_Yield_20-24.csv');
    const csv = await res.text();
    data = d3.csvParse(csv, d3.autoType);

    drawLinePlot();
  });

  function drawLinePlot() {
    const margin = { top: 50, right: 150, bottom: 80, left: 80 };
    const width = 1000 - margin.left - margin.right;
    const height = 400 - margin.top - margin.bottom;

    const svg = d3.select("#line-plot")
      .append("svg")
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

    const x = scaleTime()
      .domain(d3.extent(data, d => new Date(d.Date)))
      .range([0, width]);
      
    const y = scaleLinear()
      .domain([0, d3.max(data, d => d.Price)])
      .range([height, 0]);

    const xAxis = axisBottom(x);
    const yAxis = axisLeft(y);

    svg.append("g")
      .attr("class", "x-axis")
      .attr("transform", "translate(0," + height + ")")
      .call(xAxis);

    svg.append("g")
      .attr("class", "y-axis")
      .call(yAxis);
    
    svg.selectAll(".x-axis line")
      .attr("stroke", "#ccc");
    svg.selectAll(".y-axis line")
      .attr("stroke", "#ccc");

    svg.append("g")
      .attr("class", "grid")
      .call(d3.axisLeft(y)
          .tickSize(-width)
          .tickFormat("")
      )
      .selectAll(".tick line")
      .attr("stroke", "#ccc");

    const lineGenerator = line()
      .x(d => x(new Date(d.Date)))
      .y(d => y(d.Price))
      .curve(curveMonotoneX);

    svg.append("path")
      .datum(data)
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 1.5)
      .attr("d", lineGenerator);
      

    // Add x-axis label
    svg.append("text")
      .attr("class", "x label")
      .attr("text-anchor", "end")
      .attr("x", width - 380)
      .attr("y", height + 50) // Adjusted positioning
      .text("Date");

    // Add y-axis label
    svg.append("text")
      .attr("class", "y label")
      .attr("text-anchor", "end")
      .attr("x", -margin.left - 65)
      .attr("y", -margin.left + 20) // Adjusted positioning
      .attr("dy", ".75em")
      .attr("transform", "rotate(-90)")
      .text("Price");
    
    // Add graph title
    svg.append("text")
        .attr("class", "title")
        .attr("text-anchor", "middle")
        .attr("x", width -650)
        .attr("y", -margin.top / 2) // Adjusted positioning
        .text("United States 10-Year Bond Yield")
        .style("font-size", "18px");

    // Add data points
    svg.selectAll(".dot")
      .data(data)
      .enter().append("circle")
      .attr("class", "dot")
      .attr("cx", d => x(new Date(d.Date)))
      .attr("cy", d => y(d.Price))
      .attr("r", 4)
      .style("fill", "white")
      .style("opacity", 0.0)
      .on("mouseover", (event, d) => {
        tooltip.style("display", null);
        tooltip.attr("transform", `translate(${x(new Date(d.Date))},${y(d.Price)})`);
        tooltipText.text(`Date: ${d.Date}`);
        tooltipText.append("tspan").attr("x", 10).attr("dy", "1.2em").text(`Price: ${d.Price}`);
        tooltipText.append("tspan").attr("x", 10).attr("dy", "1.2em").text(`Open: ${d.Open}`);
        tooltipText.append("tspan").attr("x", 10).attr("dy", "1.2em").text(`High: ${d.High}`);
        tooltipText.append("tspan").attr("x", 10).attr("dy", "1.2em").text(`Low: ${d.Low}`);
        tooltipText.append("tspan").attr("x", 10).attr("dy", "1.2em").text(`Change%: ${d.ChangePercentage}`);
      })
      .on("mouseout", () => {
        tooltip.style("display", "none");
      });
      

    // Add a tooltip
    const tooltip = svg.append("g")
      .attr("class", "tooltip")
      .style("display", "none");

    tooltip.append("rect")
      .attr("width", 150)
      .attr("height", 125)
      .attr("fill", "white")
      .style("opacity", 0.9)
      .attr("stroke", "steelblue")
      .attr("rx", 5) // rounded corners
      .attr("ry", 5);

    const tooltipText = tooltip.append("text")
      .attr("x", 10)
      .attr("y", 20);

    tooltip.on("mouseover", () => {
      tooltip.style("display", null);
    })
    .on("mouseout", () => {
      tooltip.style("display", "none");
    });
  }

</script>

<Scroller
  top={0.0}
  bottom={1}
  threshold={0.5}
  bind:count
  bind:index
  bind:offset
  bind:progress
>
  <div
    class="background"
    slot="background"
    bind:clientWidth={width}
    bind:clientHeight={height}
  >
    <Graph {index} {width} {height}/>
  </div>
  
  <div class="foreground" slot="foreground">
    <section>
      <h1>Navigating the Dynamics of 10-Year Treasury Yield and Economic Indicators: <br>An Interactive Exploration</h1>
      <p> Name: Kelly Gong, Andrew Guo, Yishan Cai </p>
      <a href="https://github.com/KellyyyGSY/dsc106_finalproj/blob/main/README.md"
        target="_blank"
        rel="noopener noreferrer"
        class="center-link">
        Here is a detailed write-up for the project.
      </a>
    </section>
    <section>
      <h2> some introduction of treasury bond </h2>
      <h4> The U.S. 10-Year Bond is a debt obligation note by The United States Treasury, 
      that has the eventual maturity of 10 years. The yield on a Treasury bill represents 
      the return an investor will receive by holding the bond to maturity, 
      and should be monitored closely as an indicator of the government debt situation. 
      Investing resources into a 10 year treasury note is often considered favorable 
      due to federal government securities being exempt from state and local income tax. 
      </h4>
    </section>
    <section>
      <h2> continue on treasury bond </h2>
      <div id="line-plot"></div>
    </section>
    <section>
      <h2> some introduction of GDP </h2>
      <h4> blablablabla </h4>
    </section>
    <section>
      <h2> some introduction of inflation </h2>
      <h4> Inflation, measured by the Consumer Price Index (CPI), 
      reflects the fluctuation in prices of goods and services commonly 
      bought by specific household groups. Inflation is measured 
      in terms of the annual growth rate and in index. 
      This includes breakdowns for food, energy, and overall excluding 
      food and energy, capturing the impact on living standards. <br><br>
      The CPI is derived from proportional changes in prices 
      of a fixed basket of goods and services used or paid for 
      by the reference population. Each component is a weighted average 
      of numerous elementary aggregate indices, derived from price samples 
      collected regionally. </h4>
    </section>
    <section>This is the fifth section.</section>
    <section>This is the sixth section.</section>
  </div>
</Scroller>


<style>
  .background {
    width: 100%;
    height: 100vh;
    position: relative;
    outline: green solid 3px;
  }

  .foreground {
    width: 80%;
    margin: 0 auto;
    height: auto;
    position: relative;
    /*outline: red solid 3px;*/
  }

  section {
    height: 90vh;      /* height of each section */
    /* background-color: rgba(0, 0, 0, 0.2); */
    /* outline: magenta solid 3px; */

    text-align: center;
    max-width: 100%; /* adjust at will */
    color: black;    /* color of title */
    padding: 5.5em;    /* the distance from top to the title*/
    margin: 0 0 2em 0;
  }

  h1 {
    font-size: 3em; /* Adjust the font size here */
    margin-bottom: 0.8em; /* Add margin below the title */
  }

</style>
