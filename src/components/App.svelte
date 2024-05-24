<script>
  import { onMount } from 'svelte';
  import { writable, get } from 'svelte/store';
  import * as d3 from 'd3';

  let svg;
  const selectedNodeLinks = writable([]);
  const bfsPath = writable([]);
  const iterationTime = writable(0);
  const pathFound = writable(false);
  const foundNodeCoordinates = writable({ x: 0, y: 0 });
  let startNodeId = '';
  let endNodeId = '';
  let animationDelay = 500; // Default delay for animation in milliseconds

  const nodes = writable([]);
  const links = writable([]);
  const selectedNode = writable(null); // Store to hold the selected node

  async function loadGraphData() {
    const data = await d3.csv('src/components/small_wiki.csv'); 
    const parsedNodes = new Set();
    const parsedLinks = [];

    data.forEach(row => {
      const { page_title_from, page_title_to } = row;
      parsedNodes.add(page_title_from);
      parsedNodes.add(page_title_to);
      parsedLinks.push({ source: page_title_from, target: page_title_to });
    });

    nodes.set(Array.from(parsedNodes).map(id => ({ id })));
    links.set(parsedLinks);
  }

  onMount(() => {
    loadGraphData().then(() => {
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
        .attr('markerWidth', 7)
        .attr('markerHeight', 10)
        .attr('xoverflow', 'visible')
        .append('svg:path')
        .attr('d', 'M 0,-5 L 10 ,0 L 0,5')
        .attr('fill', '#999')
        .style('stroke', 'none');

      const currentNodes = get(nodes);
      const currentLinks = get(links);

      const simulation = d3.forceSimulation(currentNodes)
        .force('link', d3.forceLink(currentLinks).id(d => d.id).distance(100))
        .force('charge', d3.forceManyBody().strength(-300))
        .force('center', d3.forceCenter(width / 2, height / 2));

      const link = svgElement.append('g')
          .attr('class', 'links')
        .selectAll('line')
        .data(currentLinks)
        .enter().append('line')
          .attr('class', 'link')
          .attr('stroke', '#999')
          .attr('stroke-opacity', 0.6)
          .attr('stroke-width', 2)
          .attr('marker-end', 'url(#arrowhead)');  // Add marker-end to use the arrowhead marker

      const node = svgElement.append('g')
          .attr('class', 'nodes')
        .selectAll('circle')
        .data(currentNodes)
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
        .nodes(currentNodes)
        .on('tick', ticked);

      simulation.force('link')
        .links(currentLinks);

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
        tooltip.html(`${d.id}`)
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
        const nodeLinks = currentLinks.filter(link => link.source.id === d.id || link.target.id === d.id);
        selectedNodeLinks.set(nodeLinks);
        selectedNode.set(d); // Set the selected node
      }
    });
  });

  function bfs(startId, endId) {
    const queue = [{ id: startId, depth: 0 }];
    const visited = new Set();
    const parent = {};
    const depthOrder = []; // To store nodes by depth level
    visited.add(startId);

    while (queue.length > 0) {
      const { id, depth } = queue.shift();
      if (!depthOrder[depth]) depthOrder[depth] = [];
      depthOrder[depth].push(id);

      if (id === endId) {
        const path = [];
        let step = endId;
        while (step) {
          path.unshift(step);
          step = parent[step];
        }
        return { path, depthOrder };
      }
      const neighbors = get(links).filter(link => link.source.id === id)
                             .map(link => link.target.id);
      for (const neighbor of neighbors) {
        if (!visited.has(neighbor)) {
          visited.add(neighbor);
          parent[neighbor] = id;
          queue.push({ id: neighbor, depth: depth + 1 });
        }
      }
    }
    return { path: null, depthOrder };
  }

  async function animateBfs(depthOrder, path) {
    const nodesSelection = d3.selectAll('.node');
    const linksSelection = d3.selectAll('.link');

    for (let depth = 0; depth < depthOrder.length; depth++) {
      iterationTime.set(depth + 1);

      // Highlight all nodes at the current depth level
      depthOrder[depth].forEach(id => {
        nodesSelection.filter(d => d.id === id)
                      .transition()
                      .duration(animationDelay / 2)
                      .attr('fill', 'orange');
      });

      await new Promise(resolve => setTimeout(resolve, animationDelay));

      // Highlight visited
      depthOrder[depth].forEach(id => {
        nodesSelection.filter(d => d.id === id)
                      .transition()
                      .duration(animationDelay / 2)
                      .attr('fill', 'green');
      });
    }

    // Highlight found
    const foundNode = nodesSelection.filter(d => d.id === path[path.length - 1]);
    foundNode.transition()
              .duration(animationDelay / 2)
              .attr('fill', 'purple');

    foundNodeCoordinates.set({ x: foundNode.datum().x, y: foundNode.datum().y });
    pathFound.set(true);

    // Hide the message after the animation delay
    setTimeout(() => {
      pathFound.set(false);
    }, animationDelay * 2);

    // Highlight visited
    for (let i = 1; i < path.length; i++) {
      iterationTime.set(depthOrder.length + i);
      linksSelection.filter(d => d.source.id === path[i - 1] && d.target.id === path[i])
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('stroke', 'blue')
                    .attr('stroke-width', 3);
      
      nodesSelection.filter(d => d.id === path[i - 1])
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('fill', 'blue');

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
    const startId = startNodeId.trim();
    const endId = endNodeId.trim();
    const { path, depthOrder } = bfs(startId, endId);
    bfsPath.set(path || []);
    pathFound.set(false);
    if (path) {
      animateBfs(depthOrder, path);
    }
  }
</script>

<svg bind:this={svg}></svg>

{#if $selectedNodeLinks.length > 0 && $selectedNode}
  <div class="link-table">
    <h3><a href={`https://en.wikipedia.org/wiki/${$selectedNode.id}`} target="_blank" rel="noopener noreferrer">{$selectedNode.id}</a></h3>
    <table>
      <tr>
        <th>Linked From</th>
        <th>Linked To</th>
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
    <label for="startNode">Starting Wiki:</label>
    <select id="startNode" bind:value={startNodeId} required>
      <option value="" disabled>Select a node</option>
      {#each $nodes as node}
        <option value={node.id}>{node.id}</option>
      {/each}
    </select>
    <label for="endNode">Final Wiki:</label>
    <select id="endNode" bind:value={endNodeId} required>
      <option value="" disabled>Select a node</option>
      {#each $nodes as node}
        <option value={node.id}>{node.id}</option>
      {/each}
    </select>
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

{#if $pathFound}
  <div class="found-message" style="top: {$foundNodeCoordinates.y}px; left: {$foundNodeCoordinates.x}px;">
    <h3>Path Found!</h3>
  </div>
{/if}

<style>
  svg {
    width: 100vw;
    height: 100vh;
    display: block;
  }

  .link-table {
    position: absolute;
    top: 10px;
    right: 10px;
    background: white;
    border: 1px solid #ccc;
    padding: 10px;
    border-radius: 3px;
    margin-top: 20px;
  }
  
  .path-table {
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

  .bfs-form label, .bfs-form select, .bfs-form button {
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
    bottom: 70px;
    right: 10px;
    background: white;
    border: 1px solid #ccc;
    padding: 10px;
  }

  .found-message {
    position: absolute;
    background: white;
    border: 2px solid #4CAF50;
    padding: 10px;
    border-radius: 5px;
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
</style>
