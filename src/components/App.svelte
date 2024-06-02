<script>
  import { onMount } from "svelte";
  import { writable, get } from "svelte/store";
  import * as d3 from "d3";
  import Modal from "./Modal.svelte";

  let display = ["University of California, San Diego"];

  let svg;
  const selectedNodeLinks = writable([]);
  const nodes = writable([]);
  const links = writable([]);
  const selectedNode = writable(null);
  const shortestPath = writable([]);

  let startNode = "";
  let endNode = "";
  let animationDelay = 500;
  const noPathFound = writable(false);
  const pathFound = writable(false);
  const bfsPath = writable([]);

  let showModal = true;
  let showLinkTable = writable(true);

  // colors
  let nodeColor = "#498a5a";
  let highlightColor = "orange";
  let edgeColor = "#999"

  async function loadGraphData() {
    const url =
      "https://raw.githubusercontent.com/TQZhang04/wikipedia-game-bfs/main/static/wiki.csv";

    try {
      const response = await fetch(url);
      if (!response.ok) {
        throw new Error("Network response was not ok: " + response.statusText);
      }
      const csvText = await response.text();
      const data = d3.csvParse(csvText);

      const parsedNodes = new Set();
      const parsedLinks = [];

      data.forEach((row) => {
        const { page_title_from, page_title_to } = row;
        if (display.includes(page_title_from)) {
          parsedNodes.add(page_title_from);
          parsedNodes.add(page_title_to);
          parsedLinks.push({ source: page_title_from, target: page_title_to });
        }
      });

      const filteredNodes = Array.from(parsedNodes).map((id) => ({ id }));
      const filteredLinks = parsedLinks.filter(
        (link) => parsedNodes.has(link.source) && parsedNodes.has(link.target)
      );

      nodes.set(filteredNodes);
      links.set(filteredLinks);
    } catch (error) {
      console.error("Failed to fetch wiki data:", error);
    }
  }

  onMount(() => {
    loadGraphData();
  });

  function updateGraph() {
    const width = window.innerWidth;
    const height = window.innerHeight;

    const svgElement = d3
      .select(svg)
      .attr("width", width)
      .attr("height", height);

    svgElement.selectAll("*").remove();

    svgElement
      .append("defs")
      .append("marker")
      .attr("id", "arrowhead")
      .attr("viewBox", "-0 -5 10 10")
      .attr("refX", 13)
      .attr("refY", 0)
      .attr("orient", "auto")
      .attr("markerWidth", 7)
      .attr("markerHeight", 10)
      .attr("xoverflow", "visible")
      .append("svg:path")
      .attr("d", "M 0,-5 L 10 ,0 L 0,5")
      .attr("fill", edgeColor)
      .style("stroke", "none");

    const allNodes = get(nodes);
    const allLinks = get(links);

    const currentNodes = allNodes.filter((node) => display.includes(node.id));
    const currentLinks = allLinks.filter(
      (link) => display.includes(link.source) && display.includes(link.target)
    );

    const simulation = d3
      .forceSimulation(currentNodes)
      .force(
        "link",
        d3
          .forceLink(currentLinks)
          .id((d) => d.id)
          .distance(100)
      )
      .force("charge", d3.forceManyBody().strength(-300))
      .force("center", d3.forceCenter(width / 2, height / 2));

    const link = svgElement
      .append("g")
      .attr("class", "links")
      .selectAll("path")
      .data(currentLinks)
      .enter()
      .append("path")
      .attr("class", "link")
      .attr("stroke", (d) =>
        $shortestPath.includes(d.source.id) &&
        $shortestPath.includes(d.target.id)
          ? "red"
          : edgeColor
      )
      .attr("stroke-opacity", 0.6)
      .attr("stroke-width", 2)
      .attr("marker-end", "url(#arrowhead)");

    const node = svgElement
      .append("g")
      .attr("class", "nodes")
      .selectAll("circle")
      .data(currentNodes)
      .enter()
      .append("circle")
      .attr("class", "node")
      .attr("r", 10)
      .attr("fill", (d) => ($shortestPath.includes(d.id) ? "red" : nodeColor))
      .attr("stroke", "#555")
      .attr("stroke-width", 1.5)
      .call(
        d3
          .drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended)
      )
      .on("mouseover", mouseover)
      .on("mouseout", mouseout)
      .on("click", click);

    const tooltip = d3
      .select("body")
      .append("div")
      .attr("class", "tooltip")
      .style("position", "absolute")
      .style("background", "#f9f9f9")
      .style("padding", "5px")
      .style("border", "1px solid #d3d3d3")
      .style("border-radius", "3px")
      .style("pointer-events", "none")
      .style("opacity", 0);

    node.append("title").text((d) => d.id);

    simulation.nodes(currentNodes).on("tick", ticked);

    simulation.force("link").links(currentLinks);

    function ticked() {
      link.attr("d", (d) => {
        const dx = d.target.x - d.source.x;
        const dy = d.target.y - d.source.y;
        const dr = Math.sqrt(dx * dx + dy * dy);
        if (
          currentLinks.filter(
            (link) => link.source === d.source && link.target === d.target
          ).length > 1
        ) {
          return `M${d.source.x},${d.source.y}A${dr},${dr} 0 0,1 ${d.target.x},${d.target.y}`;
        }
        return `M${d.source.x},${d.source.y}L${d.target.x},${d.target.y}`;
      });

      node.attr("cx", (d) => d.x).attr("cy", (d) => d.y);
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
      d3.select(this).classed("glow", true);
      tooltip.transition().duration(200).style("opacity", 0.9);
      tooltip
        .html(`${d.id}`)
        .style("left", event.pageX + 5 + "px")
        .style("top", event.pageY - 28 + "px");
    }

    function mouseout(event, d) {
      d3.select(this).classed("glow", false);
      tooltip.transition().duration(500).style("opacity", 0);
    }

    function click(event, d) {
      const nodeLinks = allLinks.filter((link) => link.source === d.id);
      selectedNodeLinks.set(nodeLinks);
      selectedNode.set(d);
      showLinkTable.set(true); // Show the link table when a node is clicked
    }
  }

  function handleNodeClick(nodeId) {
    if (!display.includes(nodeId)) {
      display = [...display, nodeId];
      selectedNodeLinks.update((links) =>
        links.filter((link) => link.target !== nodeId)
      );
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
        return { path, depthOrder };
      }
      const neighbors = get(links)
        .filter((link) => link.source.id === id)
        .map((link) => link.target.id);
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
    const nodesSelection = d3.selectAll(".node");
    const linksSelection = d3.selectAll(".link");

    for (let i = 0; i < path.length; i++) {
      const nodeId = path[i];
      nodesSelection
        .filter((d) => d.id === nodeId)
        .transition()
        .duration((2000 - animationDelay) / 2)
        .attr("fill", highlightColor);

      if (i > 0) {
        const sourceId = path[i - 1];
        linksSelection
          .filter((d) => d.source.id === sourceId && d.target.id === nodeId)
          .transition()
          .duration((2000 - animationDelay) / 2)
          .attr("stroke", highlightColor)
          .attr("stroke-width", 3);
      }

      await new Promise((resolve) => setTimeout(resolve, 2000 - animationDelay));
    }

    await new Promise((resolve) => setTimeout(resolve, (2000 - animationDelay) * 2));
    resetGraph();
    iterationTime.set(0);
  }

  function resetGraph() {
    const nodesSelection = d3.selectAll(".node");
    const linksSelection = d3.selectAll(".link");

    nodesSelection
      .transition()
      .duration((2000 - animationDelay) / 2)
      .attr("fill", nodeColor);
    linksSelection
      .transition()
      .duration((2000 - animationDelay) / 2)
      .attr("stroke", edgeColor)
      .attr("stroke-width", 2);
  }

  function handleBfsSubmit(event) {
    event.preventDefault();
    noPathFound.set(false);
    const startId = startNode.trim();
    const endId = endNode.trim();
    const { path, depthOrder } = bfs(startId, endId);
    bfsPath.set(path || []);
    pathFound.set(false);
    console.log(path);
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

<button class="menu-button" on:click={() => (showModal = true)}>
  Show Directions
</button>

<Modal bind:showModal>
  <h2 slot="header">Welcome to the Wikipedia Web</h2>
  <em>If you have knowledge, let others light their candle in it.</em><br />
  <em>-Margaret Fuller</em>
  <br /><br />
  All knowledge builds off of millenia of other knowledge. The golden ratio, 
  originally a mathematical concept discovered in 230 BC, has been used even 
  today in art, architecture, and design to create harmonious compositions. 
  Star charts from the ancient Egyptians and Babylonians formed the beginnings 
  of our knowledge of the stars, knowledge that today has taken us to faraway 
  galaxies. What if we could see how it's all connected?
  <br /><br />
  In this interactive web app, you can explore the vast amount of knowledge contained
  within Wikipedia, building a web of knowledge connected by hyperlinks. Explore
  your favorite topics and see how all knowledge is connected.
  <br /><br />
  <em>
    Click a node to see the articles linked to it on the right. Selecting an
    article from the list will add that node to the web. Use the pathfinding
    function on the bottom left to see how each topic connects to each other.
    The slider on the bottom right controls the speed of the pathfinding animation.
  </em>
</Modal>

<svg bind:this={svg}></svg>

{#if $showLinkTable && $selectedNodeLinks.length >= 0 && $selectedNode}
  <div class="link-table">
    <button class="close-button" on:click={() => showLinkTable.set(false)}>x</button>
    <h3>
      <a
        href={`https://en.wikipedia.org/wiki/${$selectedNode.id}`}
        target="_blank"
        rel="noopener noreferrer">{$selectedNode.id}</a
      >
    </h3>
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
        <th>Article</th>
      </tr>
      {#each $bfsPath as node, index}
        <tr>
          <td>{index + 1}</td>
          <td>
            <a
              href={`https://en.wikipedia.org/wiki/${node}`}
              target="_blank"
              rel="noopener noreferrer">{node}</a
            >
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

    <label for="speed">Speed:</label>
      <input
        type="range"
        id="speed"
        min="100"
        max="1990"
        step="10"
        bind:value={animationDelay}
      />
    


    <button type="submit" class="menu-button">Find Shortest Path</button>
  </form>

  {#if $noPathFound}
    <p style="color: red;">No path found between the selected Wiki.</p>
  {/if}

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
  background: #f9f9f9;
  border: 1px solid #ccc;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  padding: 15px;
  border-radius: 5px;
  margin-top: 20px;
  font-family: Arial, sans-serif;
}

.link-table {
  position: absolute;
  top: 10px;
  right: 10px;
  background: #f9f9f9;
  border: 1px solid #ccc;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  padding: 15px;
  border-radius: 5px;
  margin-top: 20px;
  font-family: Arial, sans-serif;
  max-height: 70vh; /* Adjust as needed */
  overflow-y: auto;
}

.close-button {
  position: absolute;
  top: 5px;
  right: 5px;
  background: none;
  border: none;
  font-size: 16px;
  cursor: pointer;
}

.link-table table {
  width: 100%;
  border-collapse: collapse;
}

.link-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.link-table tr:nth-child(even) {
  background-color: #f2f2f2;
}

.link-table tr:hover {
  background-color: #ddd;
}

.link-table td {
  cursor: pointer;
}

.link-table a {
  color: #225324;
  text-decoration: none;
}

.link-table a:hover {
  text-decoration: underline;
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

  .bfs-form label,
  .bfs-form button {
    margin: 5px 0;
  }

  .path-table {
  position: absolute;
  top: 30px;
  left: 10px;
  background: #f9f9f9;
  border: 1px solid #ccc;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  padding: 15px;
  border-radius: 5px;
  margin-top: 20px;
  font-family: Arial, sans-serif;
}

.path-table table {
  width: 100%;
  border-collapse: collapse;
}

.path-table th, .path-table td {
  border: 1px solid #ddd;
  padding: 8px;
  text-align: left;
}

.path-table th {
  background-color: #4CAF50;
  color: white;
  font-weight: bold;
}

.path-table tr:nth-child(even) {
  background-color: #f2f2f2;
}

.path-table tr:hover {
  background-color: #ddd;
}

.path-table a {
  color: #4CAF50;
  text-decoration: none;
}

.path-table a:hover {
  text-decoration: underline;
}

  .menu-button {
    border: none;
    padding: 10px 10px;
    text-align: center;
    color: #02330f;
    text-decoration: none;
    display: inline-block;
    font-size: 16px;
    border-radius: 5px;
    transition-duration: 300ms;
  }

  .menu-button:hover {
    background-color: #498a5a;
    color: white;
    cursor: pointer;
  }
</style>
