<script>
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';
  import * as d3 from 'd3';

  let svg;
  const selectedNodeLinks = writable([]);
  const bfsPath = writable([]);
  const iterationTime = writable(0);
  let startNodeId = '';
  let endNodeId = '';
  let animationDelay = 500; // Default delay for animation in milliseconds

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

  onMount(() => {
    const width = window.innerWidth;
    const height = window.innerHeight;

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
      .attr('markerWidth', 13)
      .attr('markerHeight', 13)
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
      d3.select(this).classed('glow', true);
      tooltip.transition()
        .duration(200)
        .style('opacity', .9);
      tooltip.html(`Node ID: ${d.id}`)
        .style('left', (event.pageX + 5) + 'px')
        .style('top', (event.pageY - 28) + 'px');
    }

    function mouseout(event, d) {
      d3.select(this).classed('glow', false);
      tooltip.transition()
        .duration(500)
        .style('opacity', 0);
    }

    function click(event, d) {
      const nodeLinks = links.filter(link => link.source.id === d.id || link.target.id === d.id);
      selectedNodeLinks.set(nodeLinks);
    }
  });

  function bfs(startId, endId) {
    const queue = [startId];
    const visited = new Set();
    const parent = {};
    const order = []; // To store the order of exploration
    visited.add(startId);

    while (queue.length > 0) {
      const current = queue.shift();
      order.push(current);
      if (current === endId) {
        const path = [];
        let step = endId;
        while (step) {
          path.unshift(step);
          step = parent[step];
        }
        return { path, order };
      }
      const neighbors = links.filter(link => link.source.id === current)
                             .map(link => link.target.id);
      for (const neighbor of neighbors) {
        if (!visited.has(neighbor)) {
          visited.add(neighbor);
          parent[neighbor] = current;
          queue.push(neighbor);
        }
      }
    }
    return { path: null, order };
  }

  async function animateBfs(order, path) {
    const nodesSelection = d3.selectAll('.node');
    const linksSelection = d3.selectAll('.link');

    for (let i = 0; i < order.length; i++) {
      iterationTime.set(i + 1);
      // Highlight the current node in the order of BFS
      nodesSelection.filter(d => d.id === order[i])
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('fill', 'orange');

      await new Promise(resolve => setTimeout(resolve, animationDelay));

      // If the current node is part of the path, keep it highlighted
      if (path.includes(order[i])) {
        nodesSelection.filter(d => d.id === order[i])
                      .transition()
                      .duration(animationDelay / 2)
                      .attr('fill', 'green');
      } else {
        // Otherwise, reset its color
        nodesSelection.filter(d => d.id === order[i])
                      .transition()
                      .duration(animationDelay / 2)
                      .attr('fill', '#69b3a2');
      }
    }

    // Highlight the path from start to end
    for (let i = 1; i < path.length; i++) {
      iterationTime.set(order.length + i);
      linksSelection.filter(d => d.source.id === path[i - 1] && d.target.id === path[i])
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('stroke', 'green')
                    .attr('stroke-width', 4);

      await new Promise(resolve => setTimeout(resolve, animationDelay));
    }

    // Reset everything back to normal after the animation
    await new Promise(resolve => setTimeout(resolve, animationDelay * 2));
    resetGraph();
    iterationTime.set(0);
  }

  function resetGraph() {
    const nodesSelection = d3.selectAll('.node');
    const linksSelection = d3.selectAll('.link');

    nodesSelection.transition().duration(animationDelay / 2).attr('fill', '#69b3a2');
    linksSelection.transition().duration(animationDelay / 2).attr('stroke', '#999').attr('stroke-width', 2);
  }

  function handleBfsSubmit(event) {
    event.preventDefault();
    const startId = parseInt(startNodeId);
    const endId = parseInt(endNodeId);
    const { path, order } = bfs(startId, endId);
    bfsPath.set(path || []);
    if (path) {
      animateBfs(order, path);
    }
  }
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

<div class="bfs-form">
  <form on:submit={handleBfsSubmit}>
    <label for="startNode">Start Node ID:</label>
    <input type="number" id="startNode" bind:value={startNodeId} required>
    <label for="endNode">End Node ID:</label>
    <input type="number" id="endNode" bind:value={endNodeId} required>
    <button type="submit">Find BFS Path</button>
  </form>
</div>

<div class="speed-control">
  <label for="speed">Animation Speed:</label>
  <input type="range" id="speed" min="100" max="2000" step="100" bind:value={animationDelay}>
  <span>{animationDelay} ms</span>
</div>

<div class="iteration-time">
  <h3>Iteration: {$iterationTime}</h3>
</div>

{#if $bfsPath.length > 0}
  <div class="path-table">
    <h3>BFS Path</h3>
    <table>
      <tr>
        <th>Step</th>
        <th>Node ID</th>
      </tr>
      {#each $bfsPath as node, index}
        <tr>
          <td>{index + 1}</td>
          <td>{node}</td>
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

  .link-table, .path-table {
    position: absolute;
    top: 10px;
    left: 10px;
    background: white;
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 3px;
    margin-top: 20px;
  }

  .link-table table, .path-table table {
    width: 100%;
    border-collapse: collapse;
  }

  .link-table th, .link-table td, .path-table th, .path-table td {
    border: 1px solid #ccc;
    padding: 5px;
    text-align: left;
  }

  .glow {
    stroke: yellow !important;
    stroke-width: 3px;
    filter: drop-shadow(0 0 10px yellow);
  }

  .bfs-form {
    position: absolute;
    bottom: 10px;
    left: 10px;
    background: white;
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 3px;
  }

  .bfs-form form {
    display: flex;
    flex-direction: column;
  }

  .bfs-form label, .bfs-form input, .bfs-form button {
    margin: 5px 0;
  }

  .speed-control {
    position: absolute;
    bottom: 10px;
    right: 10px;
    background: white;
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 3px;
  }

  .speed-control label, .speed-control input, .speed-control span {
    margin: 5px 0;
  }

  .iteration-time {
    position: absolute;
    top: 10px;
    right: 10px;
    background: white;
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 3px;
  }
</style>
