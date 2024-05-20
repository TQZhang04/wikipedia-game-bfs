<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { writable } from 'svelte/store';

  let svg;
  const selectedNodeLinks = writable([]);

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
          .on('end', dragended))
        .on('mouseover', mouseover)  // Add mouseover event
        .on('mouseout', mouseout)    // Add mouseout event
        .on('click', click);         // Add click event

    // Create a tooltip div that is hidden by default
    const tooltip = d3.select('body').append('div')
      .attr('class', 'tooltip')
      .style('position', 'absolute')
      .style('background', '#f9f9f9')
      .style('padding', '5px')
      .style('border', '1px solid #d3d3d3')
      .style('border-radius', '3px')
      .style('pointer-events', 'none')
      .style('opacity', 0);

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

    function mouseover(event, d) {
      tooltip.transition()
        .duration(200)
        .style('opacity', .9);
      tooltip.html(`Node ID: ${d.id}`)
        .style('left', (event.pageX + 5) + 'px')
        .style('top', (event.pageY - 28) + 'px');
    }

    function mouseout(event, d) {
      tooltip.transition()
        .duration(500)
        .style('opacity', 0);
    }

    function click(event, d) {
      const nodeLinks = links.filter(link => link.source.id === d.id || link.target.id === d.id);
      selectedNodeLinks.set(nodeLinks);
    }
  });
</script>

<svg bind:this={svg}></svg>

{#if $selectedNodeLinks.length > 0}
  <div class="link-table">
    <h3>Node Links</h3>
    <table>
      <tr>
        <th>Source</th>
        <th>Target</th>
      </tr>
      {#each $selectedNodeLinks as link}
        <tr>
          <td>{link.source.id}</td>
          <td>{link.target.id}</td>
        </tr>
      {/each}
    </table>
  </div>
{/if}

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

  .tooltip {
    position: absolute;
    text-align: center;
    width: 60px;
    height: 28px;
    padding: 2px;
    font: 12px sans-serif;
    background: lightsteelblue;
    border: 0px;
    border-radius: 8px;
    pointer-events: none;
  }

  .link-table {
    position: absolute;
    top: 10px;
    left: 10px;
    background: white;
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 3px;
  }

  .link-table table {
    width: 100%;
    border-collapse: collapse;
  }

  .link-table th, .link-table td {
    border: 1px solid #ccc;
    padding: 5px;
    text-align: left;
  }
</style>
