<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let svg;

  onMount(() => {
    const width = window.innerWidth;
    const height = window.innerHeight;

    const nodes = [
      { id: 1 }, { id: 2 }, { id: 3 }, { id: 4 },
      { id: 5 }, { id: 6 }, { id: 7 }, { id: 8 }
    ];

    const links = [
      { source: 1, target: 2 }, { source: 1, target: 3 },
      { source: 2, target: 4 }, { source: 3, target: 4 },
      { source: 4, target: 5 }, { source: 5, target: 6 },
      { source: 6, target: 7 }, { source: 7, target: 8 },
      { source: 8, target: 1 }, { source: 2, target: 5 },
      { source: 3, target: 6 }, { source: 4, target: 7 },
      { source: 5, target: 8 }
    ];

    const svgElement = d3.select(svg)
      .attr('width', width)
      .attr('height', height);

    // Define arrow markers for the links
    svgElement.append('defs').append('marker')
      .attr('id', 'arrowhead')
      .attr('viewBox', '-0 -5 10 10')
      .attr('refX', 13)
      .attr('refY', 0)
      .attr('orient', 'auto')
      .attr('markerWidth', 7)
      .attr('markerHeight', 10)
      .attr('xoverflow', 'visible')
      .append('svg:path')
      .attr('d', 'M 0,-5 L 10 ,0 L 0,5')
      .attr('fill', '#999')
      .style('stroke', 'none');

    const simulation = d3.forceSimulation(nodes)
      .force('link', d3.forceLink(links).id(d => d.id).distance(100))
      .force('charge', d3.forceManyBody().strength(-300))
      .force('center', d3.forceCenter(width / 2, height / 2));

    const link = svgElement.append('g')
        .attr('class', 'links')
      .selectAll('line')
      .data(links)
      .enter().append('line')
        .attr('class', 'link')
        .attr('stroke', '#999')
        .attr('stroke-opacity', 0.6)
        .attr('stroke-width', 2)
        .attr('marker-end', 'url(#arrowhead)');  // Add marker-end to use the arrowhead marker

    const node = svgElement.append('g')
        .attr('class', 'nodes')
      .selectAll('circle')
      .data(nodes)
      .enter().append('circle')
        .attr('class', 'node')
        .attr('r', 10)
        .attr('fill', '#69b3a2')
        .attr('stroke', '#555')
        .attr('stroke-width', 1.5)
        .call(d3.drag()
          .on('start', dragstarted)
          .on('drag', dragged)
          .on('end', dragended));

    node.append('title')
      .text(d => d.id);

    simulation
      .nodes(nodes)
      .on('tick', ticked);

    simulation.force('link')
      .links(links);

    function ticked() {
      link
          .attr('x1', d => d.source.x)
          .attr('y1', d => d.source.y)
          .attr('x2', d => d.target.x)
          .attr('y2', d => d.target.y);

      node
          .attr('cx', d => d.x)
          .attr('cy', d => d.y);
    }

    function dragstarted(event, d) {
      if (!event.active) simulation.alphaTarget(0.3).restart();
      d.fx = d.x;
      d.fy = d.y;
    }

    function dragged(event, d) {
      d.fx = event.x;
      d.fy = event.y;
    }

    function dragended(event, d) {
      if (!event.active) simulation.alphaTarget(0);
      d.fx = null;
      d.fy = null;
    }
  });
</script>

<svg bind:this={svg}></svg>

<style>
  svg {
    width: 100vw;
    height: 100vh;
    display: block;
  }

  .node {
    fill: #69b3a2;
    stroke: #555;
    stroke-width: 1.5px;
  }

  .link {
    stroke: #999;
    stroke-opacity: 0.6;
  }
</style>
