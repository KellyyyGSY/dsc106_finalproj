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
  import PageProgress from "svelte-page-progress";

  let count, index, offset, progress;
  let width, height;
  let data, gdpData, rGDPData, cpindexData, cpiData;

  let selectedYear = "2000";
  let allowedYears = Array.from({length: 2024 - 2000}, (v, i) => (2000 + i).toString());
  let filteredGDP = [];
  let filteredGrow = [];
  let yearIndex = allowedYears.indexOf(selectedYear);

  let isCorrect;

  let isClient = typeof window !== 'undefined';

  const messages_bonds = [
  { date: "2008-10-01", Yield: 3.97, message: "Point A reached" },
  { date: "2022-01-01", price: 150, message: "Point B reached" }];

  const timeline = gsap.timeline({defaults: {duration: 2, opacity: 0}});
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
        end: "bottom 90%",
        pinSpacing: false,
        pin: true,
        onEnter: () => gsap.to(section, { autoAlpha: 1 }), // Fade in the current section
        onLeaveBack: () => gsap.to(section, { autoAlpha: 0 }), // Fade out the current section when scrolling back
      });
    });

    if (window.location.hash === '') {
      window.scrollTo(10, 10);
    }

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
      duration: 1, // Animation duration of 1 second
      scrollTrigger: {
        trigger: "#hook1",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    gsap.from("#hook2", {
      opacity: 0, // Start from invisible
      duration: 1, // Animation duration of 1 second
      delay:2,
      scrollTrigger: {
        trigger: "#hook1",
        start: 'top 50%', 
        end: 'top 20%', 
        toggleActions: 'play none none reverse', // Play the animation on scroll down and reverse on scroll up
      },
    });

    gsap.from("#hook3", {
      opacity: 0, // Start from invisible
      duration: 1, // Animation duration of 1 second
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
        end: '+=300%%', 
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
    drawGDPLinePlot()

    const response = await fetch('rGDP_growth.csv');
    const text = await response.text();
    rGDPData = d3.csvParse(text, d => ({
      date: d3.timeParse("%YQ%q")(d.Quarterly), 
      growth: +d['Growth']
    }));
    drawBarChart();
    drawLinePlot();

    const path = d3.select(".dataline").node();
    const pathLength = path.getTotalLength();
    
    path.style.strokeDasharray = pathLength;
    path.style.strokeDashoffset = pathLength;

    // ScrollTrigger setup to animate the drawing
    gsap.registerPlugin(ScrollTrigger);

    gsap.to(path.style, {
      strokeDashoffset: 0,
      ease: "none",
      scrollTrigger: {
        trigger: "#line-plot",
        start: "top 50%", // Animation starts when the top of the plot enters the view
        end: "bottom top", // Animation ends when the bottom of the plot exits the view
        scrub: 3
      }
    });

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

    initializeGDPQuiz();
  });

  function drawLinePlot() {
    const margin = { top: 0, right: 150, bottom: 200, left: 80 };
    const width = 1200 - margin.left - margin.right;
    const height = 600 - margin.top - margin.bottom;

    const svg = d3.select("#line-plot")
      .append("svg")
      .attr("width", width + margin.left + margin.right)
      .attr("height", height + margin.top + margin.bottom)
      .append("g")
      .attr("transform", "translate(" + margin.left + "," + margin.top + ")");
    
    data.sort((a, b) => new Date(a.Date) - new Date(b.Date));
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

    const lineGenerator = line()
      .x(d => x(new Date(d.Date)))
      .y(d => y(d.Price))
      .curve(curveMonotoneX);

    svg.append("path")
      .attr("class", "dataline")
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
        tooltipText.append("tspan").attr("x", 10).attr("dy", "1.2em").text(`Yield: ${d.Price}`);
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


// GDP overview
  let feedbackVisible = false;
  let correctAnswers = ['A', 'B', 'C', 'D'];
  let selectedAnswers = [];

  // Function to handle the click event
  function checkAnswers() {
      feedbackVisible = true;
  }

  // Helper to update selected answers
  function updateSelectedAnswers(value, checked) {
      if (checked) {
          selectedAnswers = [...selectedAnswers, value];
      } else {
          selectedAnswers = selectedAnswers.filter((answer) => answer !== value);
      }
  }

  // Computed state to determine if the answer is correct
  $: isCorrect = correctAnswers.length === selectedAnswers.length && correctAnswers.every(answer => selectedAnswers.includes(answer));


  function drawGDPLinePlot() {
    const margin = { top: 0, right: 180, bottom: 90, left: 160};
    const svgWidth = 1200;
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

    data.sort((a, b) => new Date(a.Date) - new Date(b.Date));
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
    
    // Add X-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("x", plotWidth / 2) // Center the text
      .attr("y", svgHeight - margin.bottom / 4) // Position below the X-axis
      .text("Year")
      .attr("fill", "white");

    // Add Y-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-90)") // Rotate the text for vertical alignment
      .attr("y", -margin.left / 2) // Position left of the Y-axis
      .attr("x", -plotHeight / 2.5) // Center the text vertically
      .text("Billion Dollars")
      .attr("fill", "white");


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
      .attr("fill", "steelblue")
      .attr("opacity", 0.1)
      .attr("d", areaCurrent);

    // Append the area for GDP in chained 2017 dollars
    svg.append("path")
      .datum(gdpData)
      .attr("fill", "white")
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
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2)
      .attr('class' , 'gdpline')
      .attr("d", lineCurrent);

    // Append the path for GDP in chained 2017 dollars
    svg.append("path")
      .datum(gdpData)
      .attr("fill", "none")
      .attr("stroke", "white")
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
      .style("fill", "steelblue")
      .style("opacity", 0.5)
      .on("mouseover", (event, d) => {
        const xOffset = -160; // Adjust this value for horizontal offset
        const yOffset = 5; // Adjust this value for vertical offset

        // Calculate the adjusted position
        const adjustedX = xScale(d.date) + xOffset;
        const adjustedY = yScale(d.gdpCurrent) + yOffset;

        // Set the position of the tooltip
        tooltipCurrent.style("display", "block")
          .attr("transform", `translate(${adjustedX}, ${adjustedY})`);
        
        // Update tooltip text
        tooltipTextCurrent.select(".date").text(`Date: ${d3.timeFormat("%Y Q%q")(d.date)}`);
        tooltipTextCurrent.select(".gdpCurrent").text(`GDP (Current Dollars): $${d.gdpCurrent} Billion`);
      })
      .on("mouseout", () => {
        tooltipCurrent.style("display", "none");
      });

    // Add a tooltip for GDP in current dollars
    const tooltipCurrent = svg.append("g")
      .attr("class", "tooltip")
      .style("display", "none");

    tooltipCurrent.append("rect")
      .attr("width", 320)
      .attr("height", 50)
      .attr("fill", "white")
      .style("opacity", 0.9)
      .attr("stroke", "steelblue")
      .attr("rx", 5) // rounded corners
      .attr("ry", 5);

    const tooltipTextCurrent = tooltipCurrent.append("text")
      .attr("x", 10)
      .attr("y", 20);

    tooltipTextCurrent.append("tspan")
      .attr("class", "date")
      .attr("x", 10)
      .attr("dy", "0.1em");

    tooltipTextCurrent.append("tspan")
      .attr("class", "gdpCurrent")
      .attr("x", 10)
      .attr("dy", "1.2em");

    // Add circles for GDP in chained 2017 dollars
    svg.selectAll(".dot-chained")
      .data(gdpData)
      .enter().append("circle")
      .attr("class", "dot-chained")
      .attr("cx", d => xScale(d.date))
      .attr("cy", d => yScale(d.gdpChained))
      .attr("r", 3)
      .style("fill", "white")
      .style("opacity", 0.5)
      .on("mouseover", (event, d) => {
        const xOffset = -160; // Adjust this value for horizontal offset
        const yOffset = 5; // Adjust this value for vertical offset

        // Calculate the adjusted position
        const adjustedX = xScale(d.date) + xOffset;
        const adjustedY = yScale(d.gdpChained) + yOffset;

        // Set the position of the tooltip
        tooltipChained.style("display", "block")
          .attr("transform", `translate(${adjustedX}, ${adjustedY})`);
        
        // Update tooltip text
        tooltipTextChained.select(".date").text(`Date: ${d3.timeFormat("%Y Q%q")(d.date)}`);
        tooltipTextChained.select(".gdpChained").text(`GDP (Chained 2017 Dollars): $${d.gdpChained} Billion`);
      })
      .on("mouseout", () => {
        tooltipChained.style("display", "none");
      });

    // Add a tooltip for GDP in chained 2017 dollars
    const tooltipChained = svg.append("g")
      .attr("class", "tooltip")
      .style("display", "none");

    tooltipChained.append("rect")
      .attr("width", 360)
      .attr("height", 50)
      .attr("fill", "white")
      .style("opacity", 0.9)
      .attr("stroke", "steelblue")
      .attr("rx", 5) // rounded corners
      .attr("ry", 5);

    const tooltipTextChained = tooltipChained.append("text")
      .attr("x", 10)
      .attr("y", 20);

    tooltipTextChained.append("tspan")
      .attr("class", "date")
      .attr("x", 10)
      .attr("dy", "0.1em");

    tooltipTextChained.append("tspan")
      .attr("class", "gdpChained")
      .attr("x", 10)
      .attr("dy", "1.2em");

    // Annotation for GDP in current dollars
    svg.append("text")
      .attr("x", plotWidth + 25) // Position to the right of the plot
      .attr("y", 20) // Adjust this value as needed
      .text("GDP in current dollars")
      .attr("fill", "steelblue") // Match the line color
      .attr("font-size", "12px");

    // Line next to annotation for GDP in current dollars
    svg.append("line")
      .attr("x1", plotWidth + 5)
      .attr("y1", 15) // Adjust this value as needed to align with text
      .attr("x2", plotWidth + 18) // Short horizontal line
      .attr("y2", 15) // Same as y1 to keep it horizontal
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2);

    // Annotation for GDP in chained 2017 dollars
    svg.append("text")
      .attr("x", plotWidth + 25) // Position to the right of the plot
      .attr("y", 40) // Adjust this value as needed, place below the first annotation
      .text("GDP in chained 2017 dollars")
      .attr("fill", "white") // Match the line color
      .attr("font-size", "12px");

    // Line next to annotation for GDP in chained 2017 dollars
    svg.append("line")
      .attr("x1", plotWidth + 5)
      .attr("y1", 35) // Adjust this value as needed to align with text
      .attr("x2", plotWidth + 18) // Short horizontal line
      .attr("y2", 35) // Same as y1 to keep it horizontal
      .attr("stroke", "white")
      .attr("stroke-width", 2);
  }


// GDP by quarter
  function updateYear(event) {
    yearIndex = +event.target.value;
    selectedYear = allowedYears[yearIndex];
    updatePlot(); // Call updatePlot to redraw the plot with the new year
    updateExplanation(selectedYear);
  }

  function updateExplanation(year) {
    let explanationText = '';
    if (year == 2000) {
      explanationText += "• 2000: The U.S. economy was at the tail end of the '90s boom. The GDP growth rate was strong early in the year but began to falter as the bubble burst.\n\n";
      explanationText += "• 2000 annual GDP growth rate: 4.1%.\n";
    }

    if (year == 2001) {
      explanationText += "• 2001: The economy entered a recession in March 2001, which was compounded by the September 11 terrorist attacks. The recession was relatively mild, with GDP declining in the third quarter and recovering in the fourth. The Federal Reserve responded with aggressive interest rate cuts.\n\n";
      explanationText += "• 2001 annual GDP growth rate: 1.0%.\n";
    }

    if (year == 2002) {
      explanationText += "• 2002: Following the recession, the economy began to recover slowly, in part due to continued low interest rates and the beginning of significant government spending.\n\n";
      explanationText += "• 2002 annual GDP growth rate: 1.7%.\n";
    }

    if (year >= 2003 && year <= 2006) {
      explanationText += "• 2003 - 2006 (Pre-Crisis Stability): The economy gained momentum, with GDP growth rates picking up, helped by tax cuts, increased defense spending, and a housing boom.\n\n";
      if (year >= 2003) {
        explanationText += "• 2003 annual GDP growth rate: 2.9%.\n";
      }
      if (year >= 2004) {
        explanationText += "• 2004 annual GDP growth rate: 3.8%.\n";
      }
      if (year >= 2005) {
        explanationText += "• 2005 annual GDP growth rate: 3.5%.\n";
      }
      if (year >= 2006) {
        explanationText += "• 2006 annual GDP growth rate: 2.9%.\n";
      }
    }

    if (year >= 2007 && year <= 2009) {
      if (year >= 2007) {
        explanationText += '• 2007 (Onset of Crisis): With the onset of the financial crisis, GDP growth began to slow down, marking the beginning of economic uncertainty.\n'; // Add bullet point for 2007
        explanationText += "• 2007 annual GDP growth rate: 1.9%.\n\n";
      }
      if (year >= 2008) {
        explanationText += '• 2008 (Deepening Crisis): The crisis deepened, with GDP levels falling sharply as financial markets tumbled, credit froze up, and consumer spending and investment declined. Governments and central banks responded with unprecedented fiscal and monetary stimulus measures to stabilize and boost the economy.\n'; // Add bullet point for 2008
        explanationText += "• 2008 annual GDP growth rate: -0.1%.\n\n";
      }
      if (year >= 2009) {
        explanationText += '• 2009 (Recession): Although the financial markets began to recover in 2009, the real economy lagged, with GDP growth remaining negative or sluggish in many countries, indicating the economy was in recession.\n'; // Add bullet point for 2009
        explanationText += '• Beginning of Recovery: Towards the end of 2009, signs of recovery emerged, with GDP levels stabilizing and beginning to grow again, although the pace of recovery was uneven across different regions.\n';
        explanationText += "• 2009 annual GDP growth rate: -2.5%.\n\n";
      }
    }

    if (year >= 2010 && year <= 2011) {
      explanationText += "• 2010 - 2011: The U.S. economy started to recover from the Great Recession. Stimulus measures and monetary policy were starting to have an effect. The economy was beginning to stabilize.\n\n";
      if (year >= 2010) {
        explanationText += "• 2010 annual GDP growth rate: 2.6%.\n";
      }
      if (year >= 2011) {
        explanationText += "• 2004 annual GDP growth rate: 1.6%.\n";
      }
    }

    if (year >= 2012 && year <= 2019) {
      explanationText += "• 2012 - 2019: The economy kept improving slowly and remained steady.\n\n";
      if (year >= 2012) {
        explanationText += "• 2012 annual GDP growth rate: 2.2%.\n";
      }
      if (year >= 2013) {
        explanationText += "• 2013 annual GDP growth rate: 1.8%.\n";
      }
      if (year >= 2014) {
        explanationText += "• 2014 annual GDP growth rate: 2.5%.\n";
      }
      if (year >= 2015) {
        explanationText += "• 2015 annual GDP growth rate: 2.9%.\n";
      }
      if (year >= 2016) {
        explanationText += "• 2016 annual GDP growth rate: 1.6%.\n";
      }
      if (year >= 2017) {
        explanationText += "• 2017 annual GDP growth rate: 2.4%.\n";
      }
      if (year >= 2018) {
        explanationText += "• 2018 annual GDP growth rate: 2.9%.\n";
      }
      if (year >= 2019) {
        explanationText += "• 2019 annual GDP growth rate: 2.3%.\n";
      }
    }

    if (year >= 2020 && year <= 2023) {
      if (year >= 2020) {
        explanationText += '• 2020 (Pandemic): The U.S. GDP contracted sharply due to economic shutdowns from the COVID-19 pandemic, despite a significant rebound in the second half of the year following stimulus measures.\n';
        explanationText += "• 2020 annual GDP growth rate: -2.8%.\n\n";
      }
      if (year >= 2021) {
        explanationText += '• 2021 (Pandemic): With vaccine distribution and further stimulus, the economy bounced back, as consumer spending and business activity resumed.\n';
        explanationText += "• 2021 annual GDP growth rate: 5.9%.\n\n";
      }
      if (year >= 2022) {
        explanationText += '• 2022 (Post-Pandemic): Growth continued but faced headwinds from supply chain disruptions, inflationary pressures, and shifts in monetary policy, leading to a more moderate and uneven expansion.\n';
        explanationText += "• 2022 annual GDP growth rate: 1.9%.\n\n";
      }
      if (year >= 2023) {
        explanationText += '• 2023 (Post-Pandemic): The outlook remained cautious with moderate growth as the economy adjusted to post-pandemic conditions, inflation, and changes in fiscal and monetary policies. \n';
        explanationText += "• 2023 annual GDP growth rate: 1.4%.\n\n";
      }
    }

    // Set the explanation text in the HTML
    document.getElementById('explanation').innerHTML = explanationText;
  }


  function updatePlot() {
    // Filter gdpData based on selectedYear
    filteredGDP = gdpData.filter(d => d.date.getFullYear().toString() === selectedYear);
    filteredGrow = rGDPData.filter(d => d.date.getFullYear().toString() === selectedYear);
    
    if (filteredGDP.length > 0) {
      drawGDPbyQuarter();
      drawBarQuarter();
    }
  }

  function drawGDPbyQuarter() {
    // Clear the existing SVG to avoid overlaying multiple plots
    d3.select("#gdp-quarter").select("svg").remove();

    const margin = { top: 50, right: 180, bottom: 90, left: 160};
    const svgWidth = 800;
    const svgHeight = 300;
    const plotWidth = svgWidth - margin.left - margin.right;
    const plotHeight = svgHeight - margin.top - margin.bottom;

    // Define the SVG container
    const svg = d3.select("#gdp-quarter")
      .append("svg")
      .attr("width", svgWidth)
      .attr("height", svgHeight)
      .append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`);

    filteredGDP.sort((a, b) => new Date(a.Date) - new Date(b.Date));
    // Get the earliest and latest dates from your data
    const startDate = d3.min(filteredGDP, d => d.date);
    const endDate = d3.max(filteredGDP, d => d.date);

    // Adjust start date to be one month earlier
    const newStartDate = new Date(startDate.getFullYear(), startDate.getMonth() - 1, startDate.getDate());

    // Adjust end date to be one quarter (3 months) later
    const newEndDate = new Date(endDate.getFullYear(), endDate.getMonth() + 3, endDate.getDate());

    // Now set the domain of your xScale using these new dates
    const xScale = d3.scaleTime()
        .domain([newStartDate, newEndDate])
        .range([0, plotWidth]);

    const yScale = d3.scaleLinear()
      .domain([0, d3.max(filteredGDP, d => Math.max(d.gdpCurrent, d.gdpChained))])
      .range([plotHeight, 0]);

    // Define axes
    const yAxis = d3.axisLeft(yScale);

    // Define the tick format for the x-axis
    const xAxisTickFormat = (date) => {
      const year = date.getFullYear();
      // Assuming the first month of a quarter is representative of the quarter
      const quarter = Math.floor(date.getMonth() / 3) + 1;
      return `Year ${year}, Quarter ${quarter}`;
    };

    // Set tick values based on your data points
    const xTickValues = filteredGDP.map(d => d.date);

    // Define the x-axis with custom tick format and tick values
    const xAxis = d3.axisBottom(xScale)
      .tickFormat(xAxisTickFormat)
      .tickValues(xTickValues); // Ensure a tick for each data point

    // Append the x-axis to the SVG
    svg.append("g")
      .attr("transform", `translate(0,${plotHeight})`)
      .call(xAxis);

    // Append axes to the SVG
    svg.append("g")
      .call(yAxis);

    // Add X-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("x", plotWidth / 2) // Center the text
      .attr("y", plotHeight + margin.bottom / 2) // Position below the X-axis
      .text("Year")
      .attr("fill", "white");

    // Add Y-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-90)") // Rotate the text for vertical alignment
      .attr("y", -margin.left / 3) // Position left of the Y-axis
      .attr("x", -plotHeight / 4) // Center the text vertically
      .text("Billion Dollars")
      .attr("fill", "white");

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
      .datum(filteredGDP)
      .attr("fill", "steelblue")
      .attr("opacity", 0.3)
      .attr("d", areaCurrent);

    // Append the area for GDP in chained 2017 dollars
    svg.append("path")
      .datum(filteredGDP)
      .attr("fill", "white")
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
      .datum(filteredGDP)
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2)
      .attr('class' , 'gdpline')
      .attr("d", lineCurrent);

    // Append the path for GDP in chained 2017 dollars
    svg.append("path")
      .datum(filteredGDP)
      .attr("fill", "none")
      .attr("stroke", "white")
      .attr("stroke-width", 2)
      .attr("d", lineChained);

    // Add circles for GDP in current dollars
    svg.selectAll(".dot-current")
      .data(filteredGDP)
      .enter().append("circle")
      .attr("class", "dot-current")
      .attr("cx", d => xScale(d.date))
      .attr("cy", d => yScale(d.gdpCurrent))
      .attr("r", 5)
      .style("fill", "steelblue")
      .style("opacity", 1)
      .on("mouseover", (event, d) => {
        const xOffset = -160; // Adjust this value for horizontal offset
        const yOffset = 5; // Adjust this value for vertical offset

        // Calculate the adjusted position
        const adjustedX = xScale(d.date) + xOffset;
        const adjustedY = yScale(d.gdpCurrent) + yOffset;

        // Set the position of the tooltip
        tooltipCurrent.style("display", "block")
          .attr("transform", `translate(${adjustedX}, ${adjustedY})`);
        
        // Update tooltip text
        tooltipTextCurrent.select(".date").text(`Date: ${d3.timeFormat("%Y Q%q")(d.date)}`);
        tooltipTextCurrent.select(".gdpCurrent").text(`GDP (Current Dollars): $${d.gdpCurrent} Billion`);
      })
      .on("mouseout", () => {
        tooltipCurrent.style("display", "none");
      });

    // Add a tooltip for GDP in current dollars
    const tooltipCurrent = svg.append("g")
      .attr("class", "tooltip")
      .style("display", "none");

    tooltipCurrent.append("rect")
      .attr("width", 320)
      .attr("height", 50)
      .attr("fill", "white")
      .style("opacity", 0.9)
      .attr("stroke", "steelblue")
      .attr("rx", 5) // rounded corners
      .attr("ry", 5);

    const tooltipTextCurrent = tooltipCurrent.append("text")
      .attr("x", 10)
      .attr("y", 20);

    tooltipTextCurrent.append("tspan")
      .attr("class", "date")
      .attr("x", 10)
      .attr("dy", "0.1em");

    tooltipTextCurrent.append("tspan")
      .attr("class", "gdpCurrent")
      .attr("x", 10)
      .attr("dy", "1.2em");

    // Add circles for GDP in chained 2017 dollars
    svg.selectAll(".dot-chained")
      .data(filteredGDP)
      .enter().append("circle")
      .attr("class", "dot-chained")
      .attr("cx", d => xScale(d.date))
      .attr("cy", d => yScale(d.gdpChained))
      .attr("r", 5)
      .style("fill", "white")
      .style("opacity", 1)
      .on("mouseover", (event, d) => {
        const xOffset = -160; // Adjust this value for horizontal offset
        const yOffset = 5; // Adjust this value for vertical offset

        // Calculate the adjusted position
        const adjustedX = xScale(d.date) + xOffset;
        const adjustedY = yScale(d.gdpChained) + yOffset;

        // Set the position of the tooltip
        tooltipChained.style("display", "block")
          .attr("transform", `translate(${adjustedX}, ${adjustedY})`);
        
        // Update tooltip text
        tooltipTextChained.select(".date").text(`Date: ${d3.timeFormat("%Y Q%q")(d.date)}`);
        tooltipTextChained.select(".gdpChained").text(`GDP (Chained 2017 Dollars): $${d.gdpChained} Billion`);
      })
      .on("mouseout", () => {
        tooltipChained.style("display", "none");
      });

    // Add a tooltip for GDP in chained 2017 dollars
    const tooltipChained = svg.append("g")
      .attr("class", "tooltip")
      .style("display", "none");

    tooltipChained.append("rect")
      .attr("width", 360)
      .attr("height", 50)
      .attr("fill", "white")
      .style("opacity", 0.9)
      .attr("stroke", "steelblue")
      .attr("rx", 5) // rounded corners
      .attr("ry", 5);

    const tooltipTextChained = tooltipChained.append("text")
      .attr("x", 10)
      .attr("y", 20);

    tooltipTextChained.append("tspan")
      .attr("class", "date")
      .attr("x", 10)
      .attr("dy", "0.1em");

    tooltipTextChained.append("tspan")
      .attr("class", "gdpChained")
      .attr("x", 10)
      .attr("dy", "1.2em");

    // Annotation for GDP in current dollars
    svg.append("text")
      .attr("x", plotWidth) // Position to the right of the plot
      .attr("y", 20) // Adjust this value as needed
      .text("GDP in current dollars")
      .attr("fill", "steelblue") // Match the line color
      .attr("font-size", "12px");

    // Line next to annotation for GDP in current dollars
    svg.append("line")
      .attr("x1", plotWidth - 7)
      .attr("y1", 15) // Adjust this value as needed to align with text
      .attr("x2", plotWidth - 2) // Short horizontal line
      .attr("y2", 15) // Same as y1 to keep it horizontal
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2);

    // Annotation for GDP in chained 2017 dollars
    svg.append("text")
      .attr("x", plotWidth) // Position to the right of the plot
      .attr("y", 40) // Adjust this value as needed, place below the first annotation
      .text("GDP in chained 2017 dollars")
      .attr("fill", "white") // Match the line color
      .attr("font-size", "12px");

    // Line next to annotation for GDP in chained 2017 dollars
    svg.append("line")
      .attr("x1", plotWidth - 7)
      .attr("y1", 35) // Adjust this value as needed to align with text
      .attr("x2", plotWidth - 2) // Short horizontal line
      .attr("y2", 35) // Same as y1 to keep it horizontal
      .attr("stroke", "white")
      .attr("stroke-width", 2);
  }


// Growth overview
  function drawBarChart() {
    const margin = { top: 0, right: 140, bottom: 90, left: 80 };
    const svgWidth = 1200;
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
        .domain([d3.min(rGDPData, d => d.growth), d3.max(rGDPData, d => d.growth)])
        .range([plotHeight, 0]);

    const xAxis = d3.axisBottom(xScale).tickFormat(d3.timeFormat("%Y"));
    const yAxis = d3.axisLeft(yScale);

    svg.append("g")
        .call(yAxis);

    // Add X-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("x", plotWidth / 2) // Center the text
      .attr("y", svgHeight - margin.bottom / 4) // Position below the X-axis
      .text("Year")
      .attr("fill", "white");

    // Add Y-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-90)") // Rotate the text for vertical alignment
      .attr("y", -margin.left / 2) // Position left of the Y-axis
      .attr("x", -plotHeight / 2.5) // Center the text vertically
      .text("Percent (%)")
      .attr("fill", "white");

    // Calculate bar width based on the number of data points
    let barWidth = plotWidth / rGDPData.length;

    // Bars
    svg.selectAll(".bar")
        .data(rGDPData)
        .enter().append("rect")
        .attr("class", "bar")
        .attr("x", d => xScale(d.date) - barWidth / 2) // Center the bar
        // Set the y position based on whether the growth value is positive or negative
        .attr("y", d => yScale(Math.max(0, d.growth)))
        // Set the height based on whether the growth value is positive or negative
        .attr("height", d => Math.abs(yScale(d.growth) - yScale(0)))
        .attr("width", barWidth)
        .attr("fill", "#69b3a2")
        .on("mouseover", (event, d) => {
            tooltip.style("display", "block")
              .attr("transform", `translate(${xScale(d.date)}, ${yScale(d.growth)})`);
            tooltipText.select(".date").text(`Date: ${d3.timeFormat("%Y Q%q")(d.date)}`);
            tooltipText.select(".growth").text(`Growth: ${d.growth}`);
        })
        .on("mouseout", () => {
            tooltip.style("display", "none");
        });
    
    // Append axes to the SVG
    svg.append("g")
      .attr("transform", `translate(0,${yScale(0)})`) // Position the X-axis at y=0
      .call(xAxis);

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
      .attr("class", "growth")
      .attr("x", 10)
      .attr("dy", "1.2em");
  }

// Growth by quarter
  function drawBarQuarter() {
    // Clear the existing SVG to avoid overlaying multiple plots
    d3.select("#quartergrowth").select("svg").remove();

    const margin = { top: 0, right: 140, bottom: 90, left: 80 };
    const svgWidth = 750;
    const svgHeight = 300;
    const plotWidth = svgWidth - margin.left - margin.right;
    const plotHeight = svgHeight - margin.top - margin.bottom;

    // Define the SVG container
    const svg = d3.select("#quartergrowth")
        .append("svg")
        .attr("width", svgWidth)
        .attr("height", svgHeight)
        .append("g")
        .attr("transform", `translate(${margin.left},${margin.top})`);

    // Get the last date from your data
    const lastDate = d3.max(filteredGrow, d => d.date);
    // Create a new date one quarter (3 months) past the last date
    const oneQuarterLater = new Date(lastDate.getFullYear(), lastDate.getMonth() + 3, lastDate.getDate());
        
    // Scales and axes
    const xScale = d3.scaleTime()
      .domain([d3.min(filteredGrow, d => d.date), oneQuarterLater]) // Extend domain to oneQuarterLater
      .range([0, plotWidth]);

    const yScale = d3.scaleLinear()
        .domain([d3.min(rGDPData, d => d.growth), d3.max(rGDPData, d => d.growth)])
        .range([plotHeight, 0]);

    // Define the tick format for the x-axis
    const xAxisTickFormat = (date) => {
      const year = date.getFullYear();
      // Assuming the first month of a quarter is representative of the quarter
      const quarter = Math.floor(date.getMonth() / 3) + 1;
      return `Year ${year}, Quarter ${quarter}`;
    };

    // Set tick values based on your data points
    const xTickValues = filteredGrow.map(d => d.date);

    // Define the x-axis with custom tick format and tick values
    const xAxis = d3.axisBottom(xScale)
      .tickFormat(xAxisTickFormat)
      .tickValues(xTickValues) // Ensure a tick for each data point
      .tickSizeOuter(0); // Remove the square end ticks if desired
    
    const yAxis = d3.axisLeft(yScale);

    // Add X-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("x", plotWidth) // Center the text
      .attr("y", svgHeight / 2) // Position below the X-axis
      .text("Year")
      .attr("fill", "white");

    // Add Y-axis label
    svg.append("text")
      .attr("text-anchor", "end")
      .attr("transform", "rotate(-90)") // Rotate the text for vertical alignment
      .attr("y", -margin.left / 2) // Position left of the Y-axis
      .attr("x", -plotHeight / 2.5) // Center the text vertically
      .text("Percent (%)")
      .attr("fill", "white");

    // Calculate bar width based on the number of data points
    let barWidth = 80;

    // Bars
    svg.selectAll(".bar")
        .data(filteredGrow)
        .enter().append("rect")
        .attr("class", "bar")
        .attr("x", d => xScale(d.date)) // Center the bar
        // Set the y position based on whether the growth value is positive or negative
        .attr("y", d => yScale(Math.max(0, d.growth)))
        // Set the height based on whether the growth value is positive or negative
        .attr("height", d => Math.abs(yScale(d.growth) - yScale(0)))
        .attr("width", barWidth)
        .attr("fill", "#69b3a2")
        .on("mouseover", (event, d) => {
            tooltip.style("display", "block")
              .attr("transform", `translate(${xScale(d.date)}, ${yScale(d.growth)})`);
            tooltipText.select(".date").text(`Date: ${d3.timeFormat("%Y Q%q")(d.date)}`);
            tooltipText.select(".growth").text(`Growth: ${d.growth}`);
        })
        .on("mouseout", () => {
            tooltip.style("display", "none");
        });
    
    // Append axes to the SVG

      const xAxisGroup = svg.append("g")
      .attr("transform", `translate(0,${yScale(0)})`)
      .call(xAxis);
    // Select all tick texts and set their text-anchor to 'start' after the axis has been drawn
    xAxisGroup.selectAll("text")
      .style("text-anchor", "start");
    
    svg.append("g")
        .call(yAxis);

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
      .attr("class", "growth")
      .attr("x", 10)
      .attr("dy", "1.2em");
  }

  function scrollToSection(sectionId) {
    const section = document.getElementById(sectionId);
    if (section) {
      const scrollOptions = {
        behavior: 'smooth',
        block: 'start' // Start aligns the top of the element with the top of the scrollable ancestor (consider 'center' if you want it in the middle).
      };

      // Fade in the section if it's not fully opaque.
      gsap.to(section, { autoAlpha: 1, duration: 0.5 });

      // Use GSAP's scrollTo plugin for a smooth scroll, if available, or fall back to native scrollIntoView.
      if (gsap.plugins.ScrollTo) {
        gsap.to(window, { scrollTo: { y: section.offsetTop, autoKill: false }, duration: 1 });
      } else {
        section.scrollIntoView(scrollOptions);
      }
    }
  }


  // Inflation MCQ
  let feedbackVisibleCPI = false;
  let correctAnswersCPI = ['B'];
  let selectedAnswersCPI = [];

  // Function to handle the click event
  function checkAnswersCPI() {
      feedbackVisibleCPI = true;
  }

  // Helper to update selected answers
  function updateSelectedAnswersCPI(value, checked) {
      if (checked) {
          selectedAnswersCPI = [...selectedAnswersCPI, value];
      } else {
          selectedAnswersCPI = selectedAnswersCPI.filter((answer) => answer !== value);
      }
  }

  // Computed state to determine if the answer is correct
  $: isCorrect = correctAnswersCPI.length === selectedAnswersCPI.length && correctAnswersCPI.every(answer => selectedAnswersCPI.includes(answer));


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

{#if isClient}
  <div><PageProgress color="grey" height="1vh" /></div>
{/if}

<Scroller
  top={0.0}
  bottom={1}
  threshold={0.25}
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
      <h1 class="text-reveal" style="margin-top: 0px;">
        Navigating the Dynamics of 10-Year Treasury Yield and Economic Indicators:
      <div class="title" style="margin-top: 50px;">
        {#each "An Interactive Exploration".split('').map((char) => char === ' ' ? '\u00A0' : char) as letter}
        <span>{letter}</span>
        {/each}
      </div>
      </h1>
      <p class = 'author' style="margin-top: 100px;"> 
        {#each "Kelly Gong, Andrew Guo, Yishan Cai".split('').map((char) => char === ' ' ? '\u00A0' : char) as letter}
        <span>{letter}</span>
        {/each}
        </p><br>
    </section>

    <section id = "hook">
      <p class="style1" id="hook1">In a world where fluctuation of numbers and rates gauged the pulse of the economy, the dynamics of the 10-Year Treasury Yield stand out as a critical barometer of economic health and investor sentiment.</p>
      <p class="style1" id="hook2">But what if the seemingly arcane interplay between GDP growth, and inflation rates could unlock the secrets to predicting treasury yields and crafting savvy investment strategies?</p>
      <p class="style1" id="hook3">Join us on a journey that transforms complex economic data into a captivating narrative,  where each trend line tells a story of opportunity, risk, and the relentless quest.</p>

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
        <div class="slider-navigation">
          <button class="button-style" on:click={() => scrollToSection('rela')}>Quant Model</button>
          <li>Enpower you with ability to produce customized quant prediction model</li>
        </div>
      </div>
    </section>

    <section id = "bond">
      <h2> Treasury Bond </h2>
      <p>
      The U.S. 10-Year Bond is a debt obligation note by The United States Treasury. <br>
      The yield on a Treasury bill represents the return an investor will receive by holding the bond to maturity.
      Investing resources into a 10 year treasury note is often considered favorable 
      due to federal government securities being exempt from state and local income tax. 
      </p>
      <li style="margin-top: 50px;"><a href="#zero" style="color: #42393B;">Back to main menu</a>
    </section>

    <section id="bond_takeaway">
      <h2 style="margin-bottom: 2em;"> Quick Tips </h2>
      <ul>
        <li>
          <label>
            <input type="checkbox">
            Bonds price and bonds yield have an inverse relationship (if you have to pay a higher price to buy, the less you earn).
          </label>
        </li>
        <li>
          <label>
            <input type="checkbox">
            People buy bonds when they are panic; the more people buy, the price will go up, yield will go down.
          </label>
        </li>
        <li>
          <label>
            <input type="checkbox">
            People will also buy bonds when they believe their money preserves value (low inflation).
          </label>
        </li>
      </ul>
    </section>   

    <section id = "bondviz">
      <h2> United States 10-Year Bond Yield </h2>
      <div id="line-plot"></div> <!-- Container for the bond line plot -->
    </section>

    
    <section id = 'bond-explain1'> 
      <div class="bond_frame">
        <p> Inflation Expectations: Think of inflation like the rate at which money loses its value over time. In 2020, people expected money to hold its value better than usual because everyone was spending less. </p>
      </div>
    </section>
    <section id = 'bond-explain2'> 
      <div class="bond_frame">
        <p> When people think their money will stay valuable, they like investing in government bonds, which pay back with interest over time. Because so many people wanted these bonds, their yields (or the interest you earn from them) went down.</p>
      </div>
    </section>
    <section id = 'bond-explain3'> 
      <div class="bond_frame">
        <p> Economic Activity (GDP): The economy took a big hit in 2020 due to the pandemic. Businesses closed, and many people stopped buying things, leading to a drop in the country's total production and services. </p>
      </div>
    </section>
    <section id = 'bond-explain4'> 
      <div class="bond_frame">
        <p> During such times, people prefer putting their money into something safe, like government bonds, rather than riskier investments. This rush to buy bonds made their yields lower, as there was less need for the government to offer high interest to attract buyers.</p>
      </div>
    </section>
    <section id = 'bond-explain5'> 
      <div class="bond_frame">
        <p> Inflation Expectations: By 2023, people started to notice prices going up faster than before, which means the money in your pocket doesn't go as far as it used to. This happens when the economy gets too hot, and there's too much money chasing too few goods. </p>
      </div>
    </section>
    <section id = 'bond-explain6'> 
      <div class="bond_frame">
        <p> When people expect prices to keep rising, they're less keen on bonds that pay fixed interest because the money they get back in the future won't buy as much. So, to attract buyers, the interest rates or yields on these bonds need to go up.</p>
      </div>
    </section>
    <section id = 'bond-explain7'> 
      <div class="bond_frame">
        <p> Economic Activity (GDP): The economy bounced back strongly in 2023 after the tough times caused by the pandemic. Businesses reopened, people went back to work, and spending increased. This recovery is good news, but it can also lead to too much money flowing in the economy, contributing to higher inflation. </p>
      </div>
    </section>
    <section id = 'bond-explain8'> 
      <div class="bond_frame">
        <p> To keep things in balance and ensure the economy doesn't overheat, interest rates on government bonds were raised. This means if you're buying bonds, you can expect to earn more interest than before, reflecting the higher yields seen during this period.</p>
      </div>
    </section>

    <section id="gdp">
      <h2>What is GDP?</h2>
      <p>Gross Domestic Product (GDP) is a crucial economic indicator that measures the total value of all goods and services produced within a country over a specific period, typically a quarter or a year.</p>
      <p>In the context of this project, GDP serves as a fundamental economic indicator that can significantly influence the dynamics of the 10-Year Treasury Yield. Changes in GDP growth rates can lead to adjustments in monetary policy, which in turn can affect interest rates and thus the Treasury yields.</p>
      <div id="gdp-quiz" style="margin-top: 50px;">
        <p><strong>Question: What is included in GDP?</strong></p>
        <ul style="list-style-type:none;">
            <li><input type="checkbox" id="answerA" value="A" on:change="{(e) => updateSelectedAnswers(e.target.value, e.target.checked)}"> <label for="answerA">Cars</label></li>
            <li><input type="checkbox" id="answerB" value="B" on:change="{(e) => updateSelectedAnswers(e.target.value, e.target.checked)}"> <label for="answerB">Clothes</label></li>
            <li><input type="checkbox" id="answerC" value="C" on:change="{(e) => updateSelectedAnswers(e.target.value, e.target.checked)}"> <label for="answerC">A Haircut</label></li>
            <li><input type="checkbox" id="answerD" value="D" on:change="{(e) => updateSelectedAnswers(e.target.value, e.target.checked)}"> <label for="answerD">A Doctor's Care</label></li>
        </ul>
        <div class="feedback-container">
          <button on:click={checkAnswers} style="margin-top: 20px;">Check Answer</button>
          <p id="feedback" class={feedbackVisible ? 'feedback-visible' : 'feedback-hidden'}>
              {#if isCorrect}
                  Correct! All options (Cars, Clothes, A Haircut, A Doctor's Care) are included in GDP.
              {:else}
                  Incorrect. The correct answers are A (Cars), B (Clothes), C (A Haircut), and D (A Doctor's Care).
              {/if}
          </p>
        </div>
      </div>
      <li style="margin-top: 50px; background: transparent;"><a href="#zero" style="color: #42393B;">Back to main menu</a>
    </section>

    <section id="gdp_takeaway">
      <h2 style="margin-bottom: 2em;"> Quick Tips </h2>
      <ul>
        <li>
          <label>
            <input type="checkbox">
            GDP is a broad measure of economic activity, and growth rates reflect the economy's health. 
          </label>
        </li>
        <li>
          <label>
            <input type="checkbox">
            A rising GDP typically indicates a growing economy, while declining GDP suggests contraction.
          </label>
        </li>
        <li>
          <label>
            <input type="checkbox">
            Moderate GDP growth is associated with healthy inflation levels, indicating robust economic activity. 
          </label>
        </li>
        <li>
          <label>
            <input type="checkbox">
            Rising GDP often leads to higher bond yields, which inversely affects bond prices.
          </label>
        </li>
      </ul>
    </section> 

    <section id="gdpoverview">
      <h2>GDP Overview from 2000 to 2023</h2>
      <div id="gdp-line-plot"></div> <!-- Container for the GDP line plot -->
      <p style="font-size: 0.9em; margin-bottom: 10px;">Note: GDP in chained 2017 dollars adjusts for inflation, providing a more accurate measure of real economic growth. The base year of 2017 is used to reflect the economy's structure and prices in a relatively recent period.</p>
    </section>
  
  

    <section>
      <h2>Real GDP Growth Rate Overview from 2000 to 2023</h2>
      <div id="gdpgrowth"></div>
    </section>

    <section id="gdpviz" style="display: flex; flex-direction: column; align-items: center;">
      <div class="title-slider-container">
        <h2 style="margin-top: 0; padding-top: 10px;">GDP Breakdown by Year</h2>
        <div class="slider-container">
          <input
            type="range"
            min="0"
            max="{allowedYears.length - 1}"
            step="1"
            bind:value="{yearIndex}"
            on:input="{updateYear}"
          />
          <span class="year-label">Year {selectedYear}</span>
        </div>
      </div>
      <div class="content-container">
        <!-- Container for the vertically stacked graphs -->
        <div class="graphs-container" style="display: flex; flex-direction: column; margin-right: 20px; justify-content: flex-start;">
          <div id="gdp-quarter" style="margin-bottom: 20px;"></div> <!-- Graph 1 container -->
          <div id="quartergrowth"></div> <!-- Graph 2 container -->
        </div>
        
        <!-- Container for the explanation text -->
        <div id="explanation" style="padding: 0 15px; margin-left: 20px; white-space: pre-line;">
          <!-- Explanation content goes here -->
        </div>
      </div>
    </section>
  
    <section id = "inflation">
      <h2>What is Inflation?</h2>
      <p>Inflation, the rise in prices over time, is managed by central banks to ensure economic stability, commonly measured by the Consumer Price Index (CPI), which tracks the price changes of a standard set of goods and services.</p>
      <p>Inflation reduces the real return on investments like Treasury bonds, leading investors to seek higher yields as compensation for expected purchasing power declines. Central banks adjust monetary policies to manage inflation, affecting interest rates and consequently Treasury yields.</p>
      <li style="margin-top: 50px; background: transparent;"><a href="#zero" style="color: #42393B;">Back to main menu</a>
    </section>

    <section id = "inflation_mcq">
      <h2 style="margin-bottom: 2em;"> Check your understanding </h2>
      <div id="inflation-quiz" style="margin-top: 40px;">
        <p><strong>Question: Which of the following best explains why the consumer price index (CPI) may not accurately measure changes in the cost of living?</strong></p>
        <ul class="custom-list-style">
            <li><input type="checkbox" id="answerACPI" value="A" on:change="{(e) => updateSelectedAnswersCPI(e.target.value, e.target.checked)}"> <label for="answerACPI">A. When real GDP increases, the CPI doesn't change</label></li>
            <li><input type="checkbox" id="answerBCPI" value="B" on:change="{(e) => updateSelectedAnswersCPI(e.target.value, e.target.checked)}"> <label for="answerBCPI">B. When prices of some goods go up, consumers buy less of those and more of goods that are cheaper</label></li>
            <li><input type="checkbox" id="answerCCPI" value="C" on:change="{(e) => updateSelectedAnswersCPI(e.target.value, e.target.checked)}"> <label for="answerCCPI">C. It doesn't include all prices, such as input costs to firms</label></li>
            <li><input type="checkbox" id="answerDCPI" value="D" on:change="{(e) => updateSelectedAnswersCPI(e.target.value, e.target.checked)}"> <label for="answerDCPI">D. It only includes goods purchased every day</label></li>
            <li><input type="checkbox" id="answerECPI" value="E" on:change="{(e) => updateSelectedAnswersCPI(e.target.value, e.target.checked)}"> <label for="answerECPI">E. It does not account for changes in consumers' incomes</label></li>
        </ul>
        <div class="feedback-container">
          <button on:click={checkAnswersCPI} style="margin-top: 20px;">Check Answer</button>
          <p id="feedback" class={feedbackVisibleCPI ? 'feedback-visible' : 'feedback-hidden'}>
              {#if isCorrect}
                  Correct! A problem with the CPI as a measure of the cost of living is that there might be "substitution bias." The CPI assumes that consumers always buy the same quantity of every good. But, in reality, if the price of one good increases, people might buy more of a substitute for that instead. This can lead to the CPI overstating an increase in the cost of living.
              {:else}
                  Incorrect. The correct answers is B (When prices of some goods go up, consumers buy less of those and more of goods that are cheaper).
              {/if}
          </p>
        </div>
      </div>
    </section>

    <section id = "inflation_takeaway">
      <h2 style="margin-bottom: 2em;"> Quick Tips </h2>
      <ul>
        <li>
          <label>
            <input type="checkbox">
            Inflation is how quick your money lose value naturally as time goes
          </label>
        </li>
        <li>
          <label>
            <input type="checkbox">
            Analogy: if inflation is 10%, the money you can buy 100 apples this year, can only buy 91 apples next year.
          </label>
        </li>
        <li>
          <label>
            <input type="checkbox">
            People prefer bonds when their money preserve value (low inflation)
          </label>
        </li>
      </ul>
    </section>

    <section id = 'cpi-viz'>
      <h2> Year to Year U.S. Consumer Price Index (CPI) </h2>
      <div id="cpi-line-plot"></div> 
    </section>

    <section id = 'cpi-pc-viz'>
      <h2> CPI 12-Month Percent Change </h2>
      <div id="cpi-pc-line-plot"></div>
    </section>

    
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

  
    <section id = "rela"> 
      <h2> Treasury bond quant model </h2>
      <br>
      <h4>Datasets</h4>
      <label><input type="checkbox" name="data" value="dataset1">GDP</label>
      <label><input type="checkbox" name="data" value="dataset2">Inflation</label>
      <label><input type="checkbox" name="data" value="dataset2">wage rate</label>
      <label><input type="checkbox" name="data" value="dataset2">unemployment rate</label>
      <label><input type="checkbox" name="data" value="dataset2">consumer confidence</label>
      <br>
      <h4>lambda-slider</h4>
      <input type="range" id="lambda-slider" min="0" max="1" step="0.01" value="0.1">
      <br>
      <li style="margin-top: 50px;"><a href="#zero" style="color: #42393B;">Back to main menu</a>
    </section>

    <section>
      <h1>The end</h1><br>
      <h1>Robust model coming soon</h1>
    </section>
    

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
  }

  section {
    height: 100vh;      /* height of each section */
    /* background-color: rgba(0, 0, 0, 0.2); */
    /* outline: magenta solid 3px; */
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
    /* background-color: rgba(255, 255, 255, 0.7); Adjust the opacity as needed */
    display: inline-block; /* Or as per your layout needs */
    padding-top: 2em; /* Adjust top padding */
    padding-bottom: 1em; /* Adjust bottom padding */
  }

  p {
    font-family: "Gill Sans", sans-serif;
    font-size: 1.2em; /* Adjust the font size here */
    padding: 10px; /* Adds some space around the text */
    display: inline-block;
  }

  p.style1{
    font-family: "Gill Sans", sans-serif;
    font-size: 1.5em; /* Adjust the font size here */
    padding: 10px; /* Adds some space around the text */
    display: inline-block;
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

  .feedback-container {
      display: flex;
      flex-direction: column;
      align-items: center;
  }

  .feedback-hidden {
      visibility: hidden;
      height: auto;
      opacity: 0;
      transition: opacity 0.5s;
  }

  .feedback-visible {
      visibility: visible;
      opacity: 1;
      transition: opacity 0.5s;
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
  
  #hook1 {
    position: relative;
    color: #D3C4BE;
    font-family: "Gill Sans", sans-serif;
    font-size: 1.7em; /* Adjust the font size here */
    padding: 10px; /* Adds some space around the text */
    padding-top: 12%; /* Adjusted padding for responsive positioning */
    display: inline-block;
    text-shadow: 0 0 10px #999999;
    max-width: 100%;
    word-wrap: break-word; /* Added to allow word wrapping */
  } 
  
  #hook2 {
    position: relative;
    color: #D3C4BE;
    font-family: "Gill Sans", sans-serif;
    font-size: 1.7em; /* Adjust the font size here */
    padding: 10px; /* Adds some space around the text */
    display: inline-block;
    text-shadow: 0 0 10px #999999;
    max-width: 100%;
    word-wrap: break-word; /* Added to allow word wrapping */
  }

  #hook3 {
    position: relative;
    color: #D3C4BE;
    font-family: "Gill Sans", sans-serif;
    font-size: 1.7em; /* Adjust the font size here */
    padding: 10px; /* Adds some space around the text */
    display: inline-block;
    text-shadow: 0 0 10px #999999;
    max-width: 100%;
    word-wrap: break-word; /* Added to allow word wrapping */
  }

  #bond {
      background-color: #f5f4d3; /* Change frame color */
  }
  #bond {
      color: #000; /* Change text color */
  }

  #bond_takeaway ul {
    list-style-type: none;
    padding: 0;
    margin: 0;
    text-align: left; 
  }

  #bond_takeaway li {
    font-size: 1.5em;
    display: block;
    margin-bottom: 3em;
  }

  #bond_takeaway input[type="checkbox"] {
    margin-right: 0.5em;
  }

  #gdp_takeaway ul {
    list-style-type: none;
    padding: 0;
    margin: 0;
    text-align: left; 
  }

  #gdp_takeaway li {
    font-size: 1.5em;
    display: block;
    margin-bottom: 3em;
  }

  #gdp_takeaway input[type="checkbox"] {
    margin-right: 0.5em;
  }

  #gdp {
      background-color: #f5f4d3; /* Change frame color */
  }
  #gdp {
      color: #000; /* Change text color */
  }

  /* Container for the title and slider */
  .title-slider-container {
    text-align: center; /* Center the text elements */
    padding: 20px; /* Add some padding around the elements */
  }

  /* Container for the slider */
  .slider-container {
    display: flex;
    justify-content: center; /* Center the slider horizontally */
    gap: 10px; /* Add a gap between the slider and the label */
    margin-top: 10px; /* Space below the slider */
  }

  /* Flex container for both the graphs and explanation */
  .content-container {
    display: flex; /* Use flexbox to lay out children */
    width: 100%; /* Full width to use the entire available space */
    margin-top: 30px;
  }

  /* Container to hold graphs vertically */
  .graphs-container {
    display: flex;
    flex-direction: column; /* Stack children vertically */
    width: 50%; /* Take up half the width */
  }

  /* Style for the explanation */
  #explanation {
    text-align: left;
    padding: 0 15px;
    white-space: pre-line; /* Ensure newline characters are respected */
    width: 50%; /* Take up the remaining space */
    margin-top: 150px;
  }

  /* Individual graph containers */
  #gdp-quarter, #quartergrowth {
    width: 100%; /* Ensure graphs use the full width of their container */
    margin-bottom: 20px; /* Add space below each graph */
  }

  #inflation {
      background-color: #f5f4d3; /* Change frame color */
  }
  #inflation {
      color: #000; /* Change text color */
  }

  #inflation_takeaway ul {
    list-style-type: none;
    padding: 0;
    margin: 0;
    text-align: left; 
  }

  #inflation_takeaway li {
    font-size: 1.5em;
    display: block;
    margin-bottom: 3em;
  }

  #inflation_takeaway input[type="checkbox"] {
    margin-right: 0.5em;
  }

  #cpi-pc-explain1 {
    text-align: center; /* Center-align the content */
  }

  .frame {
    display: inline-block; /* Ensure the frame doesn't stretch to full width */
    padding: 30px; /* Adjust padding as needed */
    padding-top: 20px; /* Adjust top padding to change distance from top */
    background-color: white; /* White background */
    border: 2px solid black; /* Black border */
    border-radius: 10px; /* Rounded corners */
    margin-top: 150px; /* Adjust margin as needed */
    width: 600px; /* Adjust width as needed */
    height: 190px; /* Adjust height as needed */
    text-align: left; /* Center-align the text content within the frame */
  }

  .frame p {
    color: black; /* Black text color */
  }

  .bond_frame {
    display: inline-block; /* Ensure the frame doesn't stretch to full width */
    padding: 30px; /* Adjust padding as needed */
    padding-top: 15px; /* Adjust top padding to change distance from top */
    background-color: white; /* White background */
    border: 2px solid black; /* Black border */
    border-radius: 10px; /* Rounded corners */
    margin-top: 110px; /* Adjust margin as needed */
    width: 600px; /* Adjust width as needed */
    height: 170px; /* Adjust height as needed */
    text-align: left; /* Center-align the text content within the frame */
  }

  .bond_frame p {
    color: black; /* Black text color */
  }

  /* Custom styling for the list items in the "Check your understanding" section */
  .custom-list-style li {
    display: inline-block;
    margin-bottom: 30px; /* Adjust the spacing between list items as desired */
    margin-right: 10px; /* Adjust the spacing between list items */
    margin-bottom: 10px; /* Adjust the vertical spacing if needed */
  }



</style>
