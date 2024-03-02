<script>
  import { onMount, afterUpdate } from 'svelte';
  import * as d3 from 'd3';
  import { scaleLinear, scaleTime } from 'd3-scale';
  import { line, curveMonotoneX } from 'd3-shape';

  let data;
  let svg;
  let lineGenerator;

  onMount(async () => {
    const res = await fetch('US10_Year_Bond_Yield_20-24.csv');
    const csv = await res.text();
    data = d3.csvParse(csv, d3.autoType);
    
    // Initialize the SVG element
    svg = d3.select(".graph")
      .attr("width", width)
      .attr("height", height)
      .append("g")
      .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

    // Define the scales and line generator
    const x = scaleTime()
      .domain(d3.extent(data, d => new Date(d.Date)))
      .range([0, width]);
      
    const y = scaleLinear()
      .domain([0, d3.max(data, d => d.Price)])
      .range([height, 0]);

    lineGenerator = line()
      .x(d => x(new Date(d.Date)))
      .y(d => y(d.Price))
      .curve(curveMonotoneX);

    // Draw the line
    svg.append("path")
      .datum(data)
      .attr("class", "line")
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 1.5)
      .attr("d", lineGenerator(data));

    // Animate the line
    const totalLength = svg.select('.line').node().getTotalLength();
    svg.select('.line')
      .attr("stroke-dasharray", totalLength + " " + totalLength)
      .attr("stroke-dashoffset", totalLength)
      .transition()
      .duration(2000)
      .ease(d3.easeLinear)
      .attr("stroke-dashoffset", 0);
  });

  afterUpdate(() => {
    // Update the line on window resize
    if (svg) {
      const x = scaleTime()
        .domain(d3.extent(data, d => new Date(d.Date)))
        .range([0, width]);
        
      const y = scaleLinear()
        .domain([0, d3.max(data, d => d.Price)])
        .range([height, 0]);

      const updatedLine = lineGenerator
        .x(d => x(new Date(d.Date)))
        .y(d => y(d.Price));

      svg.select('.line')
        .attr("d", updatedLine(data));
    }
  });
</script>

<style>
  .graph {
    width: 100%;
    height: 100%;
    position: absolute;
    outline: red solid 7px;
  }
</style>