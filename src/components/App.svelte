<script>
    import { onMount } from 'svelte';
    import { writable, get } from 'svelte/store';
    import * as d3 from 'd3';
  
    let svg;
    const selectedNodeLinks = writable([]);
    const eulerianPath = writable([]);
    const iterationTime = writable(0);
    const pathFound = writable(false);
    const foundNodeCoordinates = writable({ x: 0, y: 0 });
    const noPathFound = writable(false);
    let startNodeId = '';
    let animationDelay = 500;
  
    const nodes = writable([]);
    const links = writable([]);
    const selectedNode = writable(null);
    const startNodePosition = writable({ x: 0, y: 0 });
  
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
          .selectAll('path')
          .data(currentLinks)
          .enter().append('path')
            .attr('class', 'link')
            .attr('stroke', '#999')
            .attr('stroke-opacity', 0.6)
            .attr('stroke-width', 2)
            .attr('marker-end', 'url(#arrowhead)');
  
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
            .on('mouseover', mouseover)
            .on('mouseout', mouseout)
            .on('click', click);
  
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
          link.attr('d', d => {
            const dx = d.target.x - d.source.x;
            const dy = d.target.y - d.source.y;
            const dr = Math.sqrt(dx * dx + dy * dy);
            if (currentLinks.filter(link => link.source.id === d.source.id && link.target.id === d.target.id).length > 1) {
              return `M${d.source.x},${d.source.y}A${dr},${dr} 0 0,1 ${d.target.x},${d.target.y}`;
            }
            return `M${d.source.x},${d.source.y}L${d.target.x},${d.target.y}`;
          });
  
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
          selectedNode.set(d);
        }
      });
    });
  
    function isBridge(source, target) {
      const cloneLinks = get(links).filter(l => !(l.source.id === source && l.target.id === target));
      const visited = new Set();
      const queue = [source];
      while (queue.length > 0) {
        const node = queue.shift();
        visited.add(node);
        const neighbors = cloneLinks.filter(l => l.source.id === node).map(l => l.target.id);
        for (const neighbor of neighbors) {
          if (!visited.has(neighbor)) queue.push(neighbor);
        }
      }
      return !visited.has(target);
    }
  
    function fleuryAlgorithm(startId) {
      const currentLinks = get(links).map(link => ({ ...link })); // Clone the links array
      const path = [];
      let currentNode = startId;
  
      function getDegree(nodeId) {
        return currentLinks.filter(link => link.source.id === nodeId).length;
      }
  
      function getOddDegreeVertex() {
        for (const node of get(nodes)) {
          if (getDegree(node.id) % 2 !== 0) {
            return node.id;
          }
        }
        return null;
      }
  
      if (getOddDegreeVertex() === null) {
        noPathFound.set(true);
        return path;
      }
  
      while (currentLinks.length > 0) {
        const neighbors = currentLinks.filter(link => link.source.id === currentNode).map(link => link.target.id);
  
        if (neighbors.length === 0) break;
  
        let nextNode = neighbors[0];
        if (neighbors.length > 1) {
          for (const neighbor of neighbors) {
            if (!isBridge(currentNode, neighbor)) {
              nextNode = neighbor;
              break;
            }
          }
        }
  
        const linkIndex = currentLinks.findIndex(link => link.source.id === currentNode && link.target.id === nextNode);
        path.push(currentLinks[linkIndex]);
        currentLinks.splice(linkIndex, 1); // Remove the edge from the graph
        currentNode = nextNode;
      }
  
      return path;
    }
  
    async function animateFleury(path) {
    const nodesSelection = d3.selectAll('.node');
    const linksSelection = d3.selectAll('.link');
  
    for (let i = 0; i < path.length; i++) {
      iterationTime.set(i + 1);
      const { source, target } = path[i];
  
      // Color the source node
      nodesSelection.filter(d => d.id === source.id)
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('fill', 'blue');
  
      // Wait for half the animation delay
      await new Promise(resolve => setTimeout(resolve, animationDelay / 2));
  
  
      // Color the link being visited
      linksSelection.filter(d => d.source.id === source.id && d.target.id === target.id)
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('stroke', 'blue')
                    .attr('stroke-width', 0);
  
  
      // Revert the color of the previous node
      nodesSelection.filter(d => d.id === source.id)
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('fill', '#69b3a2');
  
      // Color the target node
      nodesSelection.filter(d => d.id === target.id)
                    .transition()
                    .duration(animationDelay / 2)
                    .attr('fill', 'blue');
  
      await new Promise(resolve => setTimeout(resolve, animationDelay / 2));
    }
  
      pathFound.set(true);
      await new Promise(resolve => setTimeout(resolve, animationDelay));
      pathFound.set(false);
  
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
  
    function handleFleurySubmit(event) {
      event.preventDefault();
      noPathFound.set(false);
      const startId = startNodeId.trim();
      const path = fleuryAlgorithm(startId);
      if (path.length > 0) {
        eulerianPath.set(path);
        animateFleury(path);
      } else {
        noPathFound.set(true);
      }
    }
  
    $: if (startNodeId) {
      const node = get(nodes).find(n => n.id === startNodeId);
      if (node) {
        startNodePosition.set({ x: node.x, y: node.y });
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
    <form on:submit={handleFleurySubmit}>
      <label for="startNode">Starting Wiki:</label>
      <select id="startNode" bind:value={startNodeId} required>
        <option value="" disabled>Select a node</option>
        {#each $nodes as node}
          <option value={node.id}>{node.id}</option>
        {/each}
      </select>
      <button type="submit">Find Eulerian Path</button>
    </form>
  
    {#if $noPathFound}
      <p style="color: red;">No Eulerian path found starting from the selected node.</p>
    {/if}
  </div>
  
  <div class="speed-control">
    <label for="speed">Animation Speed:</label>
    <input type="range" id="speed" min="100" max="2000" step="100" bind:value={animationDelay}>
    <span>{animationDelay} ms</span>
  </div>
  
  <div class="iteration-time">
    <h3>Iteration: {$iterationTime}</h3>
  </div>
  
  {#if $eulerianPath.length > 0}
    <div class="path-table">
      <h3>Eulerian Path</h3>
      <div class="scrollable">
        <table>
          <tr>
            <th>Step</th>
            <th>From Node</th>
            <th>To Node</th>
          </tr>
          {#each $eulerianPath as {source, target}, index}
            <tr>
              <td>{index + 1}</td>
              <td>{source.id}</td>
              <td>{target.id}</td>
            </tr>
          {/each}
        </table>
      </div>
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
      max-height: 300px;
      overflow-y: scroll;
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
      width: 160px;
      padding: 5px;
      font: 12px sans-serif;
      background: lightsteelblue;
      border: 1px solid #d3d3d3;
      border-radius: 3px;
      pointer-events: none;
    }
  
    .scrollable {
      max-height: 200px;
      overflow-y: auto;
    }
  </style>
  


<!-- <script>
  import Graph from "./Graph1.svelte";
</script>

<main>
  <h1>Dijkstra's Algorithm Visualization</h1>



  <Graph />
</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 800px;
    margin: 0 auto;
  }

  h1 {
    color: #333;
  }
</style> -->
