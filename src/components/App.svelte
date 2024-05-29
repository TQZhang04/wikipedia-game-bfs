<script>
  import { onMount } from 'svelte';
  import { writable, get } from 'svelte/store';
  import * as d3 from 'd3';

  let display = ["University of California, San Diego"];

  let svg;
  const selectedNodeLinks = writable([]);
  const nodes = writable([]);
  const links = writable([]);
  const selectedNode = writable(null);
  const shortestPath = writable([]);

  let startNode = '';
  let endNode = '';
  let animationDelay = 500;
  const noPathFound = writable(false);
  const pathFound = writable(false);
  const bfsPath = writable([]);

  async function loadGraphData() {
    const data = await d3.csv('${base}/wiki.csv');
    const parsedNodes = new Set();
    const parsedLinks = [];

    data.forEach(row => {
      const { page_title_from, page_title_to } = row;
      if (display.includes(page_title_from)) {
        parsedNodes.add(page_title_from);
        parsedNodes.add(page_title_to);
        parsedLinks.push({ source: page_title_from, target: page_title_to });
      }
    });

    const filteredNodes = Array.from(parsedNodes).map(id => ({ id }));
    const filteredLinks = parsedLinks.filter(link =>
      parsedNodes.has(link.source) && parsedNodes.has(link.target)
    );

    nodes.set(filteredNodes);
    links.set(filteredLinks);
  }

  function updateGraph() {
    const width = window.innerWidth;
    const height = window.innerHeight;

    const svgElement = d3.select(svg)
      .attr('width', width)
      .attr('height', height);

    svgElement.selectAll('*').remove();

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

    const allNodes = get(nodes);
    const allLinks = get(links);

    const currentNodes = allNodes.filter(node => display.includes(node.id));
    const currentLinks = allLinks.filter(link => display.includes(link.source) && display.includes(link.target));

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
      .attr('stroke', d => $shortestPath.includes(d.source.id) && $shortestPath.includes(d.target.id) ? 'red' : '#999')
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
      .attr('fill', d => $shortestPath.includes(d.id) ? 'red' : '#69b3a2')
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
        if (currentLinks.filter(link => link.source === d.source && link.target === d.target).length > 1) {
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
      const nodeLinks = allLinks.filter(link => link.source === d.id);
      selectedNodeLinks.set(nodeLinks);
      selectedNode.set(d);
    }
  }

  function handleNodeClick(nodeId) {
    if (!display.includes(nodeId)) {
      display = [...display, nodeId];
      loadGraphData().then(updateGraph);
    }
  }


  function bfs(startId, endId) {
      const queue = [{ id: startId, depth: 0 }];
      const visited = new Set();
      const parent = {};
      const depthOrder = []; 
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
          return {path, depthOrder };
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

  // for (let depth = 0; depth < depthOrder.length; depth++) {
  //   depthOrder[depth].forEach(id => {
  //     nodesSelection.filter(d => d.id === id)
  //       .transition()
  //       .duration(animationDelay / 2)
  //       .attr('fill', 'orange');
  //   });

  //   await new Promise(resolve => setTimeout(resolve, animationDelay));

  //   depthOrder[depth].forEach(id => {
  //     nodesSelection.filter(d => d.id === id)
  //       .transition()
  //       .duration(animationDelay / 2)
  //       .attr('fill', 'green');
  //   });
  // }

  for (let i = 0; i < path.length; i++) {
    const nodeId = path[i];
    nodesSelection.filter(d => d.id === nodeId)
      .transition()
      .duration(animationDelay / 2)
      .attr('fill', 'orange');

    if (i > 0) {
      const sourceId = path[i - 1];
      linksSelection.filter(d => d.source.id === sourceId && d.target.id === nodeId)
        .transition()
        .duration(animationDelay / 2)
        .attr('stroke', 'orange')
        .attr('stroke-width', 3);
    }

    await new Promise(resolve => setTimeout(resolve, animationDelay));
  }

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
    noPathFound.set(false); 
    const startId = startNode.trim();
    const endId = endNode.trim();
    const { path, depthOrder } = bfs(startId, endId);
    bfsPath.set(path || []);
    pathFound.set(false);
    console.log(path)
    if (path) {
      animateBfs(depthOrder, path);
    } else {
      noPathFound.set(true); 
    }
  }

  onMount(() => {
    loadGraphData().then(updateGraph);
  });
</script>

<svg bind:this={svg}></svg>

{#if $selectedNodeLinks.length >= 0 && $selectedNode}
  <div class="link-table">
    <h3><a href={`https://en.wikipedia.org/wiki/${$selectedNode.id}`} target="_blank" rel="noopener noreferrer">{$selectedNode.id}</a></h3>
    <table>
      {#each $selectedNodeLinks as link}
        <tr on:click={() => handleNodeClick(link.target)}>
          <td>{link.target}</td>
        </tr>
      {/each}
    </table>
  </div>
{/if}

{#if $bfsPath.length > 0}
    <div class="path-table">
      <table>
        <tr>
          <th>Step</th>
          <th>wikipedia</th>
        </tr>
        {#each $bfsPath as node, index}
          <tr>
            <td>{index + 1}</td>
            <td>
              <a href={`https://en.wikipedia.org/wiki/${node}`} target="_blank" rel="noopener noreferrer">{node}</a>
            </td>
          </tr>
        {/each}
      </table>
    </div>
  {/if}

<div class="bfs-form">
  <form on:submit={handleBfsSubmit}>
    <label for="startNode">Starting Wiki:</label>
    <select id="startNode" bind:value={startNode} required>
      <option value="" disabled>Select a node</option>
      {#each display as node}
        <option value={node}>{node}</option>
      {/each}
    </select>
    <label for="endNode">Final Wiki:</label>
    <select id="endNode" bind:value={endNode} required>
      <option value="" disabled>Select a node</option>
      {#each display as node}
        <option value={node}>{node}</option>
      {/each}
    </select>
    <button type="submit">Find Shortest Path</button>
  </form>

  {#if $noPathFound}
    <p style="color: red;">No path found between the selected Wiki.</p>
  {/if}
</div>


<div class="speed-control">
  <label for="speed">Animation Speed:</label>
  <input type="range" id="speed" min="100" max="2000" step="100" bind:value={animationDelay}>
  <span>{animationDelay} ms</span>
</div>




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

  .link-table table {
    width: 100%;
    border-collapse: collapse;
  }

  .link-table td {
    border: 1px solid #ccc;
    padding: 5px;
    text-align: left;
    cursor: pointer;
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
  
    .bfs-form label, .bfs-form button {
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
  
    .path-table table {
      width: 100%;
      border-collapse: collapse;
    }
  

</style>
