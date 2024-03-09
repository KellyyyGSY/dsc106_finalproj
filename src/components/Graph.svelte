<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { scaleLinear, scaleTime } from 'd3-scale';
  import { line, curveMonotoneX } from 'd3-shape';

  export let index, width, height;

  let data;

  onMount(async () => {
    const res = await fetch('US10_Year_Bond_Yield_20-24.csv');
    const csv = await res.text();
    data = d3.csvParse(csv, d3.autoType);
  });

  function drawAnimatedLine() {
    const margin = { top: 80, right: 0, bottom: 0, left: 0 };
    const svgWidth = 1350 - margin.left - margin.right;
    const svgHeight = 550;

    const svg = d3.select('.graph')
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
    const path = svg.append('path')
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
</script>

<svg class="graph" width={width} height={height}>
  {#if index === 5} <!-- it's the 6rd section now-->
    {drawAnimatedLine()}
  {/if}
</svg>

<style>
</style>