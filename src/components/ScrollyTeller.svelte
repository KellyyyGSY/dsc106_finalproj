<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { scaleLinear, scaleTime } from 'd3-scale';
  import { line, curveMonotoneX } from 'd3-shape';
  import { axisBottom, axisLeft } from 'd3-axis';
  import { gsap } from "gsap/dist/gsap";
  import Graph from "./Graph.svelte";
  import Scroller from "@sveltejs/svelte-scroller";
  import { ScrollTrigger } from "gsap/dist/ScrollTrigger";

  let count, index, offset, progress;
  let width, height;
  let data, gdpData, rGDPData, cpindexData, cpiData;

  const timeline = gsap.timeline({defaults: {duration: 2, opacity: 0}});
  const timeline2 = gsap.timeline({
    scrollTrigger: {
      trigger: "#hook", // Assuming this is your intended trigger element
      start: 'top 50%', 
      end: 'top 20%', 
      toggleActions: 'play none none reverse',
      markers: true
    }
  });
  gsap.registerPlugin(ScrollTrigger);

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
                    stagger: 0.04,
                    ease: "power1.inOut"}, "<.5") 
            .from(".center-link", {duration: 1.5, opacity: 0}, "<1")

    
    gsap.utils.toArray('section').forEach((section, index, sections) => {
      const prevSection = sections[index - 1];
      const nextSection = sections[index + 1];

      ScrollTrigger.create({
        trigger: section,
        start:"top top",
        end: "bottom 50%",
        pinSpacing: false,
        pin: true,
        onEnter: () => gsap.to(section, { autoAlpha: 1 }), // Fade in the current section
        onLeaveBack: () => gsap.to(section, { autoAlpha: 0 }), // Fade out the current section when scrolling back
      });
    });

    window.scrollTo(10, 10);


    gsap.from("#zero-title", {
      opacity: 0, // Start from invisible
      y: 20, // Start 20px lower than the final position
      duration: 1, // Animation duration of 1 second
      scrollTrigger: {
        trigger: "#zero-title",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    gsap.from("#hook1", {
      opacity: 0, // Start from invisible
      y: 20, // Start 20px lower than the final position
      duration: 3, // Animation duration of 1 second
      scrollTrigger: {
        trigger: "#hook1",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    gsap.from("#hook2", {
      opacity: 0, // Start from invisible
      duration: 3, // Animation duration of 1 second
      delay:2,
      scrollTrigger: {
        trigger: "#hook1",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    gsap.from("#hook3 span", {
      opacity: 0, // Start from invisible
      y: 20, // Start 20px lower than the final position
      duration: 3, // Animation duration of 1 second
      stagger: 0.05,
      delay:4,
      scrollTrigger: {
        trigger: "#hook1",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    gsap.from(".button-style", {
      opacity: 0, // Start from invisible
      y: 20, // Start 20px lower than the final position
      duration: 1, // Animation duration of 1 second
      scrollTrigger: {
        trigger: "#zero-title",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    gsap.from("#bond", {
      opacity: 0, // Start from invisible
      y: 20, // Start 20px lower than the final position
      duration: 1, // Animation duration of 1 second
      scrollTrigger: {
        trigger: "#bond",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    const gdpRes = await fetch('Quarterly GDP.csv');
    const gdpCsv = await gdpRes.text();
    gdpData = d3.csvParse(gdpCsv, d => ({
      date: d3.timeParse("%YQ%q")(d.Quarterly),
      gdpCurrent: +d["GDP in billions of current dollars"],
      gdpChained: +d["GDP in billions of chained 2017 dollars"]
    }));

    const response = await fetch('rGDP_growth.csv');
    const text = await response.text();
    rGDPData = d3.csvParse(text, d => ({
      date: d3.timeParse("%YQ%q")(d.Quarterly), 
      growth: +d['Growth']
    }));

    drawLinePlot();
    drawGDPLinePlot();
    drawBarChart();

    const cpindexRes = await fetch('cpi.csv');
    const cpindexCsv = await cpindexRes.text();
    cpindexData = d3.csvParse(cpindexCsv, d => ({
      date: d3.timeParse("%Y/%m")(d.Date),
      cpindex: +d["Index"]
    }));

    drawCPILinePlot();

    const cpiRes = await fetch('monthly_inflation.csv');
    const cpiCsv = await cpiRes.text();
    cpiData = d3.csvParse(cpiCsv, d => ({
      date: d3.timeParse("%Y/%m")(d.Date),
      percentageChange: +d["Percent_Change"]
    }));
    drawCPIPCLinePlot();
  });

  function drawLinePlot() {
    const margin = { top: 50, right: 150, bottom: 200, left: 80 };
    const width = 1200 - margin.left - margin.right;
    const height = 600 - margin.top - margin.bottom;

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
      .attr("fill", "white")
      .attr("x", width - 470)
      .attr("y", height + 50) // Adjusted positioning
      .text("Date");

    // Add y-axis label
    svg.append("text")
      .attr("class", "y label")
      .attr("text-anchor", "end")
      .attr("fill", "white")
      .attr("x", -margin.left - 65)
      .attr("y", -margin.left + 20) // Adjusted positioning
      .attr("dy", ".75em")
      .attr("transform", "rotate(-90)")
      .text("Price");
    

    // Add data points
    svg.selectAll(".dot")
      .data(data)
      .enter().append("circle")
      .attr("class", "dot")
      .attr("cx", d => x(new Date(d.Date)))
      .attr("cy", d => y(d.Price))
      .attr("r", 5)
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
    const margin = { top: 50, right: 30, bottom: 90, left: 60 };
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
      .style("background-color", "#262626")
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

  function drawBarChart() {
    const margin = { top: 50, right: 30, bottom: 90, left: 60 };
    const svgWidth = 1000;
    const svgHeight = 500;
    const plotWidth = svgWidth - margin.left - margin.right;
    const plotHeight = svgHeight - margin.top - margin.bottom;

    // Define the SVG container
    const svg = d3.select("#gdpgrowth")
        .append("svg")
        .attr("width", svgWidth)
        .attr("height", svgHeight)
        .append("g")
        .attr("transform", `translate(${margin.left},${margin.top})`);

    // Scales and axes
    const xScale = d3.scaleTime()
        .domain(d3.extent(rGDPData, d => d.date))
        .range([0, plotWidth]);

    const yScale = d3.scaleLinear()
        .domain([0, d3.max(rGDPData, d => d.growth)])
        .range([plotHeight, 0]);

    const xAxis = d3.axisBottom(xScale).tickFormat(d3.timeFormat("%Y"));
    const yAxis = d3.axisLeft(yScale);

    // Append axes to the SVG
    const xAxisGroup = svg.append("g")
        .attr("transform", `translate(0,${plotHeight})`)
        .call(xAxis);

    svg.append("g")
        .call(yAxis);

    // Tooltip for displaying detailed information
    const tooltip = d3.select("body").append("div")
        .attr("class", "tooltip")
        .style("opacity", 0.8)
        .style("position", "absolute")
        .style("background", "white")
        .style("color", "#000") // black text
        .style("border", "solid 1px #000")
        .style("padding", "15px")
        .style("pointer-events", "none");

    // Calculate bar width based on the number of data points
    let barWidth = plotWidth / rGDPData.length;

    // Bars
    const bars = svg.selectAll(".bar")
        .data(rGDPData)
        .enter().append("rect")
        .attr("class", "bar")
        .attr("x", d => xScale(d.date) - barWidth / 2) // Center the bar
        .attr("y", d => yScale(d.growth))
        .attr("width", barWidth)
        .attr("height", d => plotHeight - yScale(d.growth))
        .attr("fill", "#69b3a2")
        .on("click", (event, d) => {
            // Determine the scale for the zoomed-in view
            const domainPadding = (xScale.range()[1] - xScale.range()[0]) / 10; // 10% padding on each side
            const newDomain = [new Date(d.date.getTime() - domainPadding), new Date(d.date.getTime() + domainPadding)];

            // Update the xScale domain
            xScale.domain(newDomain);

            // Recalculate the bar width based on the new scale
            barWidth = (xScale.range()[1] - xScale.range()[0]) / rGDPData.length;

            // Rescale the x-axis
            xAxisGroup.transition().duration(500).call(xAxis);

            // Update bars
            bars.transition().duration(500)
                .attr("x", d => xScale(d.date) - barWidth / 2)
                .attr("width", barWidth);
        })
        .on("mouseover", (event, d) => {
            tooltip.transition()
                .duration(200)
                .style("opacity", .9);
            tooltip.html("Date: " + d3.timeFormat("%Y Q%q")(d.date) + "<br/>Growth: " + d.growth)
                .style("left", (event.pageX) + "px")
                .style("top", (event.pageY - 28) + "px");
        })
        .on("mouseout", (event, d) => {
            tooltip.transition()
                .duration(500)
                .style("opacity", 0);
        });
  }


  function scrollToSection(sectionId) {
    const section = document.getElementById(sectionId);
    if (section) {
      section.scrollIntoView({ behavior: 'smooth' });
    }
  }


  function drawCPILinePlot() {
    const margin = { top: 50, right: 150, bottom: 200, left: 90};
    const svgWidth = 1200;
    const svgHeight = 600;
    const plotWidth = svgWidth - margin.left - margin.right;
    const plotHeight = svgHeight - margin.top - margin.bottom;

    // Define the SVG container
    const svg = d3.select("#cpi-line-plot")
      .append("svg")
      .attr("width", svgWidth)
      .attr("height", svgHeight)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    // Assuming your date parsing has been done correctly when loading the data
    const xScale = d3.scaleTime()
      .domain(d3.extent(cpindexData, d => d.date))
      .range([0, plotWidth]);

    const yScale = d3.scaleLinear()
      .domain([100, d3.max(cpindexData, d => d.cpindex) + 1])
      .range([plotHeight, 0]);

    // Define axes
    const xAxis = d3.axisBottom(xScale).tickFormat(d => d.getFullYear()); // Format ticks to display only the year
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

    // Add x-axis label
    svg.append("text")
      .attr("class", "x label")
      .attr("text-anchor", "end")
      .attr("fill", "white")
      .attr("x", plotWidth - 460)
      .attr("y", plotHeight + 50) // Adjusted positioning
      .text("Date");

    // Add y-axis label
    svg.append("text")
      .attr("class", "y label")
      .attr("text-anchor", "end")
      .attr("fill", "white")
      .attr("x", -margin.left - 20)
      .attr("y", -margin.left + 40) // Adjusted positioning
      .attr("dy", ".75em")
      .attr("transform", "rotate(-90)")
      .text("Consumer Price Index");

    // Area generator for CPIndex
    const areaCPIndex = d3.area()
      .x(d => xScale(d.date))
      .y0(yScale(100)) // Set y0 to y-coordinate of the line y=100
      .y1(d => yScale(d.cpindex))
      .curve(d3.curveMonotoneX);

    // Append the area for CPIndex
    svg.append("path")
      .datum(cpindexData)
      .attr("fill", "blue")
      .attr("opacity", 0.1)
      .attr("d", areaCPIndex);

    // Line generator for CPIndex
    const lineCPIndex = d3.line()
      .x(d => xScale(d.date))
      .y(d => yScale(d.cpindex))
      .curve(d3.curveMonotoneX);

    // Append the path for CPIndex
    svg.append("path")
      .datum(cpindexData)
      .attr("fill", "none")
      .attr("stroke", "blue")
      .attr("stroke-width", 2)
      .attr("d", lineCPIndex);

    // Add data points
    svg.selectAll(".dot-cpi")
      .data(cpindexData)
      .enter().append("circle")
      .attr("class", "dot-cpi")
      .attr("cx", d => xScale(d.date))
      .attr("cy", d => yScale(d.cpindex))
      .attr("r", 5)
      .style("fill", "white")
      .style("opacity", 0.0)
      .on("mouseover", (event, d) => {
        tooltip.style("display", "block")
          .attr("transform", `translate(${xScale(d.date)}, ${yScale(d.cpindex)})`);
        tooltipText.select(".date").text(`Date: ${d3.timeFormat("%b %Y")(d.date)}`);
        tooltipText.select(".index").text(`Index: ${d.cpindex}`);
      })
      .on("mouseout", () => {
        tooltip.style("display", "none");
      });

    // Add a tooltip
    const tooltip = svg.append("g")
      .attr("class", "tooltip")
      .style("display", "none");

    tooltip.append("rect")
      .attr("width", 140)
      .attr("height", 50)
      .attr("fill", "white")
      .style("opacity", 0.9)
      .attr("stroke", "steelblue")
      .attr("rx", 5) // rounded corners
      .attr("ry", 5);

    const tooltipText = tooltip.append("text")
      .attr("x", 10)
      .attr("y", 20);

    tooltipText.append("tspan")
      .attr("class", "date")
      .attr("x", 10)
      .attr("dy", "0.1em");

    tooltipText.append("tspan")
      .attr("class", "index")
      .attr("x", 10)
      .attr("dy", "1.2em");
  }

  function drawCPIPCLinePlot() {
    const margin = { top: 50, right: 150, bottom: 200, left: 90};
    const svgWidth = 1200;
    const svgHeight = 600;
    const plotWidth = svgWidth - margin.left - margin.right;
    const plotHeight = svgHeight - margin.top - margin.bottom;

    // Define the SVG container
    const svg = d3.select("#cpi-pc-line-plot")
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

    svg.append("line")
      .attr("x1", 0)
      .attr("y1", yScale(0))
      .attr("x2", plotWidth)
      .attr("y2", yScale(0))
      .attr("stroke", "white")
      .attr("stroke-width", 2);

    // Add x-axis label
    svg.append("text")
      .attr("class", "x label")
      .attr("text-anchor", "end")
      .attr("fill", "white")
      .attr("x", plotWidth - 460)
      .attr("y", plotHeight + 50) // Adjusted positioning
      .text("Date");

    // Add y-axis label
    svg.append("text")
      .attr("class", "y label")
      .attr("text-anchor", "end")
      .attr("fill", "white")
      .attr("x", -margin.left - 20)
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
    svg.append("path")
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
    svg.append("path")
      .datum(cpiData)
      .attr("fill", "none")
      .attr("stroke", "blue")
      .attr("stroke-width", 2)
      .attr("d", lineCPI);

    // Add data points
    svg.selectAll(".dot-cpi")
      .data(cpiData)
      .enter().append("circle")
      .attr("class", "dot-cpi")
      .attr("cx", d => xScale(d.date))
      .attr("cy", d => yScale(d.percentageChange))
      .attr("r", 5)
      .style("fill", "white")
      .style("opacity", 0.0)
      .on("mouseover", (event, d) => {
        tooltip.style("display", "block")
          .attr("transform", `translate(${xScale(d.date)}, ${yScale(d.percentageChange)})`);
        tooltipText.select(".date").text(`Date: ${d3.timeFormat("%b %Y")(d.date)}`);
        tooltipText.select(".change").text(`Change%: ${d.percentageChange}`);
      })
      .on("mouseout", () => {
        tooltip.style("display", "none");
      });

    // Add a tooltip
    const tooltip = svg.append("g")
      .attr("class", "tooltip")
      .style("display", "none");

    tooltip.append("rect")
      .attr("width", 140)
      .attr("height", 50)
      .attr("fill", "white")
      .style("opacity", 0.9)
      .attr("stroke", "steelblue")
      .attr("rx", 5) // rounded corners
      .attr("ry", 5);

    const tooltipText = tooltip.append("text")
      .attr("x", 10)
      .attr("y", 20);

    tooltipText.append("tspan")
      .attr("class", "date")
      .attr("x", 10)
      .attr("dy", "0.1em");

    tooltipText.append("tspan")
      .attr("class", "change")
      .attr("x", 10)
      .attr("dy", "1.2em");
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

  <script src="https://unpkg.com/scrollreveal"></script>
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

    <!-- add for better transit, do not delete! -->
    <section></section>

    <section id = "hook">
      <p class="style1" id="hook1">In a world where fluctuation of numbers and rates gauged the pulse of the economy, the dynamics of the 10-Year Treasury Yield stand out as a critical barometer of economic health and investor sentiment.</p>

      <h3 id='hook2'>But what if the seemingly arcane interplay between GDP growth, and inflation rates could unlock the secrets to predicting treasury yields and crafting savvy investment strategies?</h3>

      <p class="style2" id="hook3">
        {#each "Join us on a journey that transforms complex economic data into a captivating narrative," .split('').map((char) => char === ' ' ? '\u00A0' : char) as letter}
        <span>{letter}</span>
        {/each}
      </p>
      <p class='style3' id='hook3'>
        {#each " where each trend line tells a story of opportunity, risk, and the relentless quest." .split('').map((char) => char === ' ' ? '\u00A0' : char) as letter}
        <span>{letter}</span>
        {/each}
      </p>
    </section>

    <section id = 'zero'>
      <h1 id="zero-title">
        {#each "Tutorial we've prepared for you".split('').map((char) => char === ' ' ? '\u00A0' : char) as letter}
        <span>{letter}</span>
        {/each}
      </h1>
      <div id="buttons-section">
        <div class="slider-navigation">
          <button class="button-style" on:click={() => scrollToSection('bond')}>Understanding Treasury Bonds</button>
          <li>Explore the basics and significance of Treasury Bonds in the financial market.</li>
        </div>
        <div class="slider-navigation">
          <button class="button-style" on:click={() => scrollToSection('gdp')}>Gross Domestic Product(GDP)</button>
          <li>Dive into GDP growth factors and its role as an economic health indicator.</li>
        </div>
        <div class="slider-navigation">
          <button class="button-style" on:click={() => scrollToSection('inflation')}>Inflation Dynamics</button>
          <li>Unpack the causes and effects of inflation on the economy and purchasing power.</li>
        </div>
      </div>
      <!-- <p class="style2"> By simplifying complex economic interactions into an intuitive and interactive format, the concept of treasury yield becomes more accessible to a broader audience. Unlike traditional explanations that rely heavily on textual descriptions and static charts, the interactive elements below allow users to explore and understand the dynamic relationship between the 10-Year Treasury Yield, GDP growth, and inflation rates on their own terms. This hands-on approach not only enhances comprehension but also engages users in a more meaningful exploration of economic principles. </p> -->
    </section>

    <!-- add for better transit, do not delete! -->
    <section></section>

    <section id = "bond">
      <h2> Treasury Bond </h2>
      <p>The U.S. 10-Year Bond is a debt obligation note by The United States Treasury, 
      that has the eventual maturity of 10 years. The yield on a Treasury bill represents 
      the return an investor will receive by holding the bond to maturity, 
      and should be monitored closely as an indicator of the government debt situation. 
      Investing resources into a 10 year treasury note is often considered favorable 
      due to federal government securities being exempt from state and local income tax. 
      </p>
      <li style="margin-top: 50px;"><a href="#zero" style="color: #42393B;">Back to main menu</a>
    </section>

    <section id = "bondviz">
      <h2> United States 10-Year Bond Yield </h2>
      <div id="line-plot"></div> <!-- Container for the bond line plot -->
    </section>

    <!-- add for better transit, do not delete! -->
    <section></section>

    <section id = "gdp">
      <h2> What is GDP? </h2>
      <p> Gross Domestic Product (GDP) is a crucial economic indicator that measures the total value of all goods and services produced within a country over a specific period, typically a quarter or a year. It is used as a comprehensive gauge of a country's overall economic health, reflecting the size and growth rate of its economy. GDP can be calculated using three approaches: the production (or output or value added) approach, the income approach, and the expenditure approach, each offering a different perspective but theoretically arriving at the same total. </p>
      <p> In the context of this project, GDP serves as a fundamental economic indicator that can significantly influence the dynamics of the 10-Year Treasury Yield. Changes in GDP growth rates can lead to adjustments in monetary policy, which in turn can affect interest rates and thus the Treasury yields. A strong and growing GDP may lead to higher yields, as investors demand more return in a booming economy, whereas a weak or contracting GDP can lead to lower yields, reflecting a move towards safer investments and potential monetary easing by the central bank to stimulate growth. </p>
      <li style="margin-top: 50px;"><a href="#zero" style="color: #42393B;">Back to main menu</a>
    </section>

    <section id = "gdpviz">
      <h2>Quarterly GDP</h2>
      <div id="gdp-line-plot"></div> <!-- Container for the GDP line plot -->
    </section>

    <!-- GDP Growth Rate Calculator here -->
    <!-- debug needed -->
    <section>
      <h2>Real GDP Growth Rate</h2>
      <div id="gdpgrowth"></div>
    </section>
    

    <section> </section>

    <section id = "inflation">
      <h2>What is Inflation?</h2>
      <p>Inflation is a key economic metric that denotes the rate at which the general level of prices for goods and services is rising, and subsequently, how purchasing power is falling. Central banks attempt to limit inflation, and avoid deflation, in order to keep the economy running smoothly. Inflation can be measured through various indices, the most common being the Consumer Price Index (CPI) and the Wholesale Price Index (WPI). CPI measures the average price change over time of a basket of goods and services that a typical household might purchase, while WPI measures the price change of goods sold and traded in bulk by wholesale businesses to other businesses.</p>
      <p>In the context of this project, understanding inflation is vital as it directly impacts the dynamics of the 10-Year Treasury Yield. Inflation erodes the real return on investments, including those in government securities such as Treasury bonds. As inflation expectations rise, investors may demand higher yields to compensate for the anticipated decrease in the purchasing power of their future interest payments. Conversely, low inflation rates may lead to lower yields, as the real return on investments becomes more stable, making government securities more attractive. Central banks may adjust monetary policy in response to inflation levels to stabilize the economy, influencing interest rates and thus impacting Treasury yields.</p>
      <li style="margin-top: 50px;"><a href="#zero" style="color: #42393B;">Back to main menu</a>
    </section>

    <section id = 'cpi-viz'>
      <h2> Year to Year U.S. Consumer Price Index (CPI) </h2>
      <div id="cpi-line-plot"></div> <!-- Container for the CPI line plot -->
    </section>

    <section id = 'cpi-pc-viz'>
      <h2> CPI 12-Month Percent Change </h2>
      <div id="cpi-pc-line-plot"></div> <!-- Container for the CPI PC line plot -->
    </section>

    <section id = 'cpi-pc-explain0'></section>
    <section id = 'cpi-pc-explain1'> 
      <div class="frame">
        <p> This line graph displays the trend of percentage change of consumer price index in each month from 2000 to 2024. We may observe from the line graph that the trend before 2008 did not fluctate greatly day to day.</p>
      </div>
    </section>
    <section id = 'cpi-pc-explain2'> 
      <div class="frame">
        <p> However, in the midst of 2008, commodity prices experienced a significant downturn, coinciding with the onset of the global financial crisis in September of that year. Initially, the decrease in overall economic activity was moderate, but it rapidly intensified during the autumn of 2008, reaching a critical point as financial market pressures peaked.</p>
      </div>
    </section>
    <section id = 'cpi-pc-explain3'> 
      <div class="frame">
        <p> In December 2008, the Consumer Price Index for All Urban Consumers (CPI-U) showed a 0.7 percent decline on a seasonally adjusted basis, marking the third consecutive month of decrease. This drop was mainly driven by lower energy prices, particularly for gasoline, with the energy index falling by 8.3 percent in December. </p>
      </div>
    </section>
    <section id = 'cpi-pc-explain4'> 
      <div class="frame">
        <p> It is worth to notice the post-pandemic recovery. When vaccinations became available, the trend in CPI shows a significant increase. This upward trend persists throughout 2021 and into early 2022, with the CPI steadily rising from 1.4% in January 2021 to 9.1% in June 2022.</p>
      </div>
    </section>
    <section id = 'cpi-pc-explain5'> 
      <div class="frame">
        <p> From June 2022 until 2024, the trend in the Consumer Price Index (CPI) appears to continue with a relatively stable or gradually increasing trajectory. </p>
      </div>
    </section>

    <section> </section>

    <section id = "rela"> 
      <h2> Interconnections </h2>
    </section>

    <section> </section>

  </div>
</Scroller>


<style>

  * {
    font-family: "Gill Sans", sans-serif;
  }

  :global(body) {
    color: white;
    background-color: black;
  }

  .background {
    width: 100%;
    height: 100vh;
    position: relative;
  }

  .foreground {
    width: 100%;
    margin: 0 auto;
    height: auto;
    position: relative;
    outline: red solid 3px;
  }

  section {
    height: 80vh;      /* height of each section */
    /* background-color: rgba(0, 0, 0, 0.2); */
    outline: magenta solid 3px;
    text-align: center;
    max-width: 100%; /* adjust at will */
    color: white;    /* color of title */
    padding: 5.5em;    /* the distance from top to the title*/
    opacity: 0;
    transition: opacity 0.1s ease-in-out;
  }

  h1 {
    font-family: "Gill Sans", sans-serif;
    font-size: 3em; /* Adjust the font size here */
    margin-bottom: 0.8em; /* Add margin below the title */
  }

  h2 {
    font-family: "Gill Sans", sans-serif;
    font-size: 2.2em; /* Adjust the font size here */
    margin-bottom: 0.8em; /* Add margin below the title */
    /* background-color: rgba(255, 255, 255, 0.7); Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
  }

  h3 {
    font-family: "Gill Sans", sans-serif;
    font-size: 1.7em; /* Adjust the font size here */
    /* background-color: rgba(255, 255, 255, 0.7); Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
  }

  p.style1{
    font-family: "Gill Sans", sans-serif;
    font-size: 1.5em; /* Adjust the font size here */
    padding: 10px; /* Adds some space around the text */
    display: inline-block;
  }

  p.style2 {
    font-family: "Gill Sans", sans-serif;
    font-size: 1em;
    display: inline-block;
    margin-top: 80px;
    padding-left: 80px;
    padding-right: 80px;
    font-weight: normal;
    color: white;
    text-align: left;
    margin-bottom: 0px; /* Reduce the bottom margin to decrease space */
  }

  p.style3 {
    font-family: "Gill Sans", sans-serif;
    font-size: 1em;
    display: inline-block;
    padding-left: 80px;
    padding-right: 80px;
    font-weight: normal;
    color: white;
    text-align: left;
    margin-top: 0px; /* Reduce the top margin to pull closer to the previous paragraph */
  }

  li {
    font-family: "Gill Sans", sans-serif;
    font-size: 1.3em; /* Adjust the font size here */
    display: inline-block; /* Or as per your layout needs */
    padding: 5px; /* Adds some space around the text */
    background-color: #bdaa96; /* Adjust the opacity as needed */
    border: None;
    border-radius: 10px;
    font-style: italic;
  }

  .text-reveal{
    /* background-color: rgba(255, 255, 255, 0.7); Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
  }

  .author{
    font-family: "Gill Sans", sans-serif;
    font-size: 1.2em;
    /* background-color: rgba(255, 255, 255, 0.7); Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
  }

  .author span{
    font-family: "Gill Sans", sans-serif;
    opacity: 0;
    display: inline-block;
    font-style: italic;
    padding: .5px
  }

  .text-reveal span {
    opacity: 0;
    display: inline-block;
  }

  .center-link{
    font-family: "Gill Sans", sans-serif;
    background-color: rgba(255, 255, 255, 0.7); /* Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding: 10px; /* Adds some space around the text */
    font-style: italic;
    font-size:large;
  }

  .title span {
    font-family: "Gill Sans", sans-serif;
    opacity: 0;
    display: inline-block;
    color: #F2F2F0;
    text-shadow: 0 0 5px #262626;
    font-style: italic;
    font-size: 35px;
  }

  #zero {
    position: relative; /* Needed for the absolute positioning of the pseudo-element */
    height: 700px; /* Adjust based on your needs */
  }

  #zero::before {
    content: "";
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-image: url("https://storage.googleapis.com/pic0_dsc106/pic5.jpeg");
    background-size: cover; /* Cover the entire section */
    background-position: center; /* Center the background image */
    opacity: 0.5; /* Apply opacity to the background image */
    z-index: -1; /* Ensure the pseudo-element is behind the content */
  }

  /* Ensure content of #zero is positioned above the pseudo-element */
  #zero > * {
    position: relative;
    z-index: 1;
  }

  #zero-title span{
    color: white;
    text-shadow: 0 0 10px #bdaa96;
  }

  #buttons-section {
    display: flex;
    flex-direction: column; /* or column, depending on your design */
    align-items: self-start;
  }
  
  .button-style {
    padding: 10px 10px;
    margin: 20px;
    background-color: transparent; /* Make the background transparent */
    color: #fff; /* Use the warm shade for text, the same as the previous background */
    border: 2px solid #bdaa96; /* Outline the button with a border in the warm shade */
    border-radius: 10px; /* Slightly rounded edges remain */
    font-size: 25px; /* Readable size */
    cursor: pointer; /* Indicates the button is clickable */
    box-shadow: none; /* Remove the shadow for a cleaner outline look */
    transition: background-color 0.3s, box-shadow 0.3s, transform 0.3s, color 0.3s, border-color 0.3s; /* Update transitions */
    font-style: normal;

    /* Add hover effects to invert colors on hover for better interaction feedback */
    &:hover {
      background-color: #E9CCB1; /* Fill color on hover */
      color: #fff; /* Text color back to white on hover */
      border-color: #E9CCB1; /* Keep the border color consistent on hover */
    }
  }

  .button-style:active {
    background-color: #bdaa96; /* A darker shade for the active state */
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2); /* Less shadow to simulate being pressed */
    transform: translateY(0); /* Return to normal position when clicked */
  }

  .slider-navigation {
    margin-top: 50px;
  }

  #custom-background-section {
      position: relative; /* Needed for the absolute positioning of the pseudo-element */
      height: 700px; /* Adjust based on your needs */
      overflow: hidden; /* Ensures the pseudo-element doesn't extend outside this container */
  }

  #custom-background-section::before {
      content: '';
      position: absolute;
      top: 0;
      left: 0;
      right: 0;
      bottom: 0;
      background-size: cover; /* Cover the entire section */
      background-position: center; /* Center the background image */
      opacity: 0.5; /* Adjust opacity as needed */
      z-index: -1; /* Ensures the background is behind the content */
      animation: switchBackground 40s infinite; /* Apply the animation */
  }

  /* Make sure the content inside #custom-background-section has a higher z-index if needed */
  #custom-background-section > * {
      position: relative;
      z-index: 1;
  }

  /* Up-and-down bobbing animation for the h3 tag */
  @keyframes bobbing {
    0%, 100% {
      transform: translateY(0);
    }
    50% {
      transform: translateY(-10px);
    }
  }

  @keyframes switchBackground {
              0%, 20% {
                  background-image: url("https://storage.googleapis.com/pic0_dsc106/pic0.jpeg");
              }
              45%,55% {
                  background-image: url("https://storage.googleapis.com/pic0_dsc106/pic1.jpeg");
              }
              55%,70% {
                  background-image: url('https://storage.googleapis.com/pic0_dsc106/pic2.jpeg');
              }
              100%,0% {
                  background-image: url('https://storage.googleapis.com/pic0_dsc106/pic3.jpeg');
              }
          }

  #hook h3 {
    animation: bobbing 3s ease-in-out infinite;
  }

  #hook3 {
    position: relative;
    color: #D3C4BE;
    font-family: "Gill Sans", sans-serif;
    font-size: 2em; /* Adjust the font size here */
    padding: 10px; /* Adds some space around the text */
    display: inline-block;
    text-shadow: 0 0 10px #999999;
    max-width: 90%;
  }

  #cpi-pc-explain1 {
    text-align: center; /* Center-align the content */
  }

  .frame {
    display: inline-block; /* Ensure the frame doesn't stretch to full width */
    padding: 30px; /* Adjust padding as needed */
    background-color: white; /* White background */
    border: 2px solid black; /* Black border */
    border-radius: 10px; /* Rounded corners */
    margin-top: 150px; /* Adjust margin as needed */
    width: 600px; /* Adjust width as needed */
    height: 150px; /* Adjust height as needed */
    text-align: left; /* Center-align the text content within the frame */
  }

  .frame p {
    color: black; /* Black text color */
  }


</style>
