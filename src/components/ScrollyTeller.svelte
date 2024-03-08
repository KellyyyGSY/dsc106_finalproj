<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { scaleLinear, scaleTime } from 'd3-scale';
  import { line, curveMonotoneX } from 'd3-shape';
  import { axisBottom, axisLeft } from 'd3-axis';
  import { gsap } from "gsap";
  import Graph from "./Graph.svelte";
  import Scroller from "@sveltejs/svelte-scroller";

  let count, index, offset, progress;
  let width, height;
  let data, gdpData;
  const timeline = gsap.timeline({defaults: {duration: 2, opacity: 0}});

  onMount(async () => {
    const res = await fetch('US10_Year_Bond_Yield_20-24.csv');
    const csv = await res.text();
    data = d3.csvParse(csv, d3.autoType);
    timeline.from(".text-reveal", {opacity: 0})
            .to(".title span", {
                    opacity: 1,
                    duration: 0.2,
                    stagger: 0.05,
                    ease: "power1.inOut"})
            .to(".author span", {
                    opacity: 1,
                    duration: 0.1,
                    stagger: 0.05,
                    ease: "power1.inOut"}, "<.5") 
            .from(".center-link", {duration: 1, opacity: 0}); 

    // Fetch and parse the GDP data
    const gdpRes = await fetch('Quarterly GDP.csv');
    const gdpCsv = await gdpRes.text();
    gdpData = d3.csvParse(gdpCsv, d => ({
      date: d3.timeParse("%YQ%q")(d.Quarterly),
      gdpCurrent: +d["GDP in billions of current dollars"],
      gdpChained: +d["GDP in billions of chained 2017 dollars"]
    }));

    drawLinePlot();
    drawGDPLinePlot();
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


  function drawGDPLinePlot() {
    const margin = { top: 50, right: 30, bottom: 30, left: 60 };
    const svgWidth = 1000;
    const svgHeight = 500;
    const plotWidth = svgWidth - margin.left - margin.right;
    const plotHeight = svgHeight - margin.top - margin.bottom;

    // Define the SVG container
    const svg = d3.select("#gdp-line-plot")
      .append("svg")
      .attr("width", svgWidth)
      .attr("height", svgHeight)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    // Assuming your date parsing has been done correctly when loading the data
    const xScale = d3.scaleTime()
      .domain(d3.extent(gdpData, d => d.date))
      .range([0, plotWidth]);

    const yScale = d3.scaleLinear()
      .domain([0, d3.max(gdpData, d => Math.max(d.gdpCurrent, d.gdpChained))])
      .range([plotHeight, 0]);

    // Define axes
    const xAxis = d3.axisBottom(xScale).tickFormat(d3.timeFormat("%Y"));
    const yAxis = d3.axisLeft(yScale);

    // Append axes to the SVG
    svg.append("g")
      .attr("transform", `translate(0,${plotHeight})`)
      .call(xAxis);

    svg.append("g")
      .call(yAxis);

    svg.append("g")
      .attr("class", "grid")
      .call(d3.axisLeft(yScale)
          .tickSize(-plotWidth)
          .tickFormat("")
      )
      .selectAll(".tick line")
      .attr("stroke", "#ccc");

    svg.append("g")
      .attr("class", "grid")
      .attr("transform", `translate(0,${plotHeight})`)
      .call(d3.axisBottom(xScale)
          .tickSize(-plotHeight)
          .tickFormat("")
      )
      .selectAll(".tick line")
      .attr("stroke", "#ccc");

    // Area generator for GDP in current dollars
    const areaCurrent = d3.area()
      .x(d => xScale(d.date))
      .y0(plotHeight)
      .y1(d => yScale(d.gdpCurrent))
      .curve(d3.curveMonotoneX);

    // Area generator for GDP in chained 2017 dollars
    const areaChained = d3.area()
      .x(d => xScale(d.date))
      .y0(plotHeight)
      .y1(d => yScale(d.gdpChained))
      .curve(d3.curveMonotoneX);

    // Append the area for GDP in current dollars
    svg.append("path")
      .datum(gdpData)
      .attr("fill", "blue")
      .attr("opacity", 0.1)
      .attr("d", areaCurrent);

    // Append the area for GDP in chained 2017 dollars
    svg.append("path")
      .datum(gdpData)
      .attr("fill", "green")
      .attr("opacity", 0.1)
      .attr("d", areaChained);

    // Line generator for GDP in current dollars
    const lineCurrent = d3.line()
      .x(d => xScale(d.date))
      .y(d => yScale(d.gdpCurrent))
      .curve(d3.curveMonotoneX);

    // Line generator for GDP in chained 2017 dollars
    const lineChained = d3.line()
      .x(d => xScale(d.date))
      .y(d => yScale(d.gdpChained))
      .curve(d3.curveMonotoneX);

    // Append the path for GDP in current dollars
    svg.append("path")
      .datum(gdpData)
      .attr("fill", "none")
      .attr("stroke", "blue")
      .attr("stroke-width", 2)
      .attr("d", lineCurrent);

    // Append the path for GDP in chained 2017 dollars
    svg.append("path")
      .datum(gdpData)
      .attr("fill", "none")
      .attr("stroke", "green")
      .attr("stroke-width", 2)
      .attr("d", lineChained);

    // Add circles for GDP in current dollars
    svg.selectAll(".dot-current")
      .data(gdpData)
      .enter().append("circle")
      .attr("class", "dot-current")
      .attr("cx", d => xScale(d.date))
      .attr("cy", d => yScale(d.gdpCurrent))
      .attr("r", 3)
      .style("fill", "blue")
      .style("opacity", 0.5)
      .on("mouseover", (event, d) => displayTooltip(event, d, "Current Dollars"))
      .on("mouseout", () => tooltip.style("display", "none"));

    // Add circles for GDP in chained 2017 dollars
    svg.selectAll(".dot-chained")
      .data(gdpData)
      .enter().append("circle")
      .attr("class", "dot-chained")
      .attr("cx", d => xScale(d.date))
      .attr("cy", d => yScale(d.gdpChained))
      .attr("r", 3)
      .style("fill", "green")
      .style("opacity", 0.5)
      .on("mouseover", (event, d) => displayTooltip(event, d, "Chained 2017 Dollars"))
      .on("mouseout", () => tooltip.style("display", "none"));

    // Tooltip setup...
    const tooltip = d3.select("body").append("div")
      .attr("class", "tooltip")
      .style("position", "absolute")
      .style("background-color", "white")
      .style("opacity", 0.9)
      .style("border", "1px solid black")
      .style("border-radius", "5px")
      .style("padding", "10px")
      .style("display", "none");

    function displayTooltip(event, d, type) {
      tooltip
        .html(`Date: ${d3.timeFormat("%Y Q%q")(d.date)}<br/>GDP (${type}): $${type === "Current Dollars" ? d.gdpCurrent : d.gdpChained} Billion`)
        .style("left", `${event.pageX + 15}px`)
        .style("top", `${event.pageY - 28}px`)
        .style("display", "block");
  }

    // Hide the tooltip when clicking anywhere on the page outside the data points
    d3.select("body").on("click", function() {
      tooltip.style("display", "none");
    }, true); // True for capturing phase
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
    <section id="custom-background-section">
      <h1 class="text-reveal">
        Navigating the Dynamics of 10-Year Treasury Yield and Economic Indicators:
      <div class="title">
        {#each "An Interactive Exploration".split('').map((char) => char === ' ' ? '\u00A0' : char) as letter}
        <span>{letter}</span>
        {/each}
      </div>
      </h1>
      <p class = 'author'> 
        {#each "Kelly Gong, Andrew Guo, Yishan Cai".split('').map((char) => char === ' ' ? '\u00A0' : char) as letter}
        <span>{letter}</span>
        {/each}
        </p><br>
      <a href="https://github.com/KellyyyGSY/dsc106_finalproj/blob/main/README.md"
        target="_blank"
        rel="noopener noreferrer"
        class="center-link">
        Project Write-up
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
      <h2> What is GDP? </h2>
      <p> Gross Domestic Product (GDP) is a crucial economic indicator that measures the total value of all goods and services produced within a country over a specific period, typically a quarter or a year. It is used as a comprehensive gauge of a country's overall economic health, reflecting the size and growth rate of its economy. GDP can be calculated using three approaches: the production (or output or value added) approach, the income approach, and the expenditure approach, each offering a different perspective but theoretically arriving at the same total. </p>
      <p> In the context of this project, GDP serves as a fundamental economic indicator that can significantly influence the dynamics of the 10-Year Treasury Yield. Changes in GDP growth rates can lead to adjustments in monetary policy, which in turn can affect interest rates and thus the Treasury yields. A strong and growing GDP may lead to higher yields, as investors demand more return in a booming economy, whereas a weak or contracting GDP can lead to lower yields, reflecting a move towards safer investments and potential monetary easing by the central bank to stimulate growth. </p>
    </section>
    <section>
      <h2>Quarterly GDP</h2>
      <div id="gdp-line-plot"></div> <!-- Container for the GDP line plot -->
    </section>
    <section>
      <h2>What is Inflation?</h2>
      <p>Inflation is a key economic metric that denotes the rate at which the general level of prices for goods and services is rising, and subsequently, how purchasing power is falling. Central banks attempt to limit inflation, and avoid deflation, in order to keep the economy running smoothly. Inflation can be measured through various indices, the most common being the Consumer Price Index (CPI) and the Wholesale Price Index (WPI). CPI measures the average price change over time of a basket of goods and services that a typical household might purchase, while WPI measures the price change of goods sold and traded in bulk by wholesale businesses to other businesses.</p>
      <p>In the context of this project, understanding inflation is vital as it directly impacts the dynamics of the 10-Year Treasury Yield. Inflation erodes the real return on investments, including those in government securities such as Treasury bonds. As inflation expectations rise, investors may demand higher yields to compensate for the anticipated decrease in the purchasing power of their future interest payments. Conversely, low inflation rates may lead to lower yields, as the real return on investments becomes more stable, making government securities more attractive. Central banks may adjust monetary policy in response to inflation levels to stabilize the economy, influencing interest rates and thus impacting Treasury yields.</p>
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
  }

  .foreground {
    width: 80%;
    margin: 0 auto;
    height: auto;
    position: relative;
    /*outline: red solid 3px;*/
  }

  section {
    height: 100vh;      /* height of each section */
    /* background-color: rgba(0, 0, 0, 0.2); */
    /* outline: magenta solid 3px; */
    text-align: center;
    max-width: 100%; /* adjust at will */
    color: black;    /* color of title */
    padding: 5.5em;    /* the distance from top to the title*/
  }

  h1 {
    font-size: 3em; /* Adjust the font size here */
    margin-bottom: 0.8em; /* Add margin below the title */
  }

  .text-reveal{
    background-color: rgba(255, 255, 255, 0.7); /* Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
  }

  .author{
    font-size: 1.2em;
    background-color: rgba(255, 255, 255, 0.7); /* Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
  }

  .author span{
    opacity: 0;
    display: inline-block;
  }

  .text-reveal span {
    opacity: 0;
    display: inline-block;
  }

  .center-link{
    background-color: rgba(255, 255, 255, 0.7); /* Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
  }

  .title span {
    opacity: 0;
    display: inline-block;
  }

  #custom-background-section {
  animation: switchBackground 20s infinite;
  background-size: cover; 
  background-position: center; 
  height: 650px; 
}

  @keyframes switchBackground {
              0%,25% {
                  background-image: url('./assets/pic0.jpeg');
              }
              25%,50% {
                  background-image: url('./assets/pic0.jpeg');
              }
              50%,75% {
                  background-image: url('./assets/pic0.jpeg');
              }
              100%,0% {
                  background-image: url('./assets/pic0.jpeg');
              }
          }
</style>
