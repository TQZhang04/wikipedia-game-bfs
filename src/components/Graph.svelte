<script>
  import { onMount } from "svelte";
  import * as d3 from "d3";

  let nodes = [{ id: 1 }, { id: 2 }, { id: 3 }, { id: 4 }, { id: 5 }];

  let edges = [
    { source: 1, target: 2, weight: 1 },
    { source: 1, target: 3, weight: 4 },
    { source: 2, target: 3, weight: 2 },
    { source: 2, target: 4, weight: 5 },
    { source: 3, target: 5, weight: 3 },
    { source: 4, target: 5, weight: 1 },
  ];

  let selectedNode = null;
  let simulation;

  function runDijkstra(startNode) {
    console.log("start", startNode);

    let distances = new Map(nodes.map((node) => [node.id, Infinity]));
    distances.set(startNode, 0);
    console.log("dist", distances);

    let visited = new Set();
    let priorityQueue = new Set(nodes);

    let steps = [];
    while (priorityQueue.size > 0) {
      console.log("PQ", priorityQueue);
      console.log("Steps", steps);

      let currentNode = [...priorityQueue].reduce((a, b) =>
        distances.get(a.id) < distances.get(b.id) ? a : b
      );
      priorityQueue.delete(currentNode);
      visited.add(currentNode);
      console.log("curr", currentNode);

      steps.push({
        node: currentNode,
        distances: new Map(distances),
        visited: new Set(visited),
      });

      // edges
      //   .filter(
      //     (edge) => edge.source === currentNode && !visited.has(edge.target)
      //   )
      //   .forEach((edge) => {
      //     let alt = distances.get(currentNode) + edge.weight;
      //     console.log("alt", alt);
      //     if (alt < distances.get(edge.target)) {
      //       console.log("updated")
      //       distances.set(edge.target, alt);
      //       // priorityQueue.add(edge.target);
      //     }
      //   });

      for (let edge of edges) {
        if (edge.source === currentNode) {
          console.log(distances.get(currentNode.id));
          console.log(edge.weight);
          let alt = distances.get(currentNode.id) + edge.weight;
          console.log("alt", alt);
          if (alt < distances.get(edge.target.id)) {
            console.log("updated");
            distances.set(edge.target.id, alt);
            // priorityQueue.add(edge.target);
          }
        } else if (edge.target === currentNode) {
          console.log(distances.get(currentNode.id));
          console.log(edge.weight);
          let alt = distances.get(currentNode.id) + edge.weight;
          console.log("alt", alt);
          if (alt < distances.get(edge.source.id)) {
            console.log("updated");
            distances.set(edge.source.id, alt);
            // priorityQueue.add(edge.target);
          }
        }
      }

      // edges
      //   .filter(
      //     (edge) => edge.target === currentNode && !visited.has(edge.source)
      //   )
      //   .forEach((edge) => {
      //     let alt = distances.get(currentNode) + edge.weight;
      //     if (alt < distances.get(edge.source)) {
      //       distances.set(edge.source, alt);
      //       priorityQueue.add(edge.source);
      //     }
      //   });
    }
    console.log(steps);
    animateSteps(steps);
  }

  function animateSteps(steps) {
    let i = 0;

    function nextStep() {
      if (i < steps.length) {
        const { node, distances, visited } = steps[i];
        updateGraph(node, distances, visited);
        i++;
        setTimeout(nextStep, 1000); // Adjust the speed of the animation here
      }
    }

    nextStep();
  }

  function updateGraph(currentNode, distances, visited) {
    const svg = d3.select("svg");

    svg
      .selectAll(".nodes circle")
      .data(nodes, (d) => d.id)
      .attr("fill", (d) => (visited.has(d.id) ? "green" : "blue"))
      .attr("stroke", (d) => (d.id === currentNode ? "red" : "none"));

    svg
      .selectAll(".links line")
      .data(edges, (d) => `${d.source}-${d.target}`)
      .attr("stroke", (d) =>
        visited.has(d.source) && visited.has(d.target) ? "green" : "#999"
      )
      .attr("stroke-width", 2);

    svg
      .selectAll(".distances text")
      .data(nodes, (d) => d.id)
      .text((d) =>
        distances.get(d.id) === Infinity ? "∞" : distances.get(d.id)
      );
  }

  onMount(() => {
    const svg = d3.select("svg");
    const width = +svg.attr("width");
    const height = +svg.attr("height");

    simulation = d3
      .forceSimulation(nodes)
      .force(
        "link",
        d3
          .forceLink(edges)
          .id((d) => d.id)
          .distance(100)
      )
      .force("charge", d3.forceManyBody().strength(-400))
      .force("center", d3.forceCenter(width / 2, height / 2));

    const link = svg
      .append("g")
      .attr("class", "links")
      .selectAll("line")
      .data(edges)
      .enter()
      .append("line")
      .attr("stroke-width", 2)
      .attr("stroke", "#999");

    const node = svg
      .append("g")
      .attr("class", "nodes")
      .selectAll("circle")
      .data(nodes)
      .enter()
      .append("circle")
      .attr("r", 10)
      .attr("fill", "blue")
      .call(
        d3
          .drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended)
      )
      .on("click", function (event, d) {
        selectedNode = d.id;
        runDijkstra(selectedNode);
      });

    svg
      .append("g")
      .attr("class", "labels")
      .selectAll("text")
      .data(edges)
      .enter()
      .append("text")
      .attr("text-anchor", "middle")
      .attr("dx", (d) => (d.source.x + d.target.x) / 2)
      .attr("dy", (d) => (d.source.y + d.target.y) / 2)
      .text((d) => d.weight);

    svg
      .append("g")
      .attr("class", "distances")
      .selectAll("text")
      .data(nodes)
      .enter()
      .append("text")
      .attr("text-anchor", "middle")
      .attr("x", (d) => d.x)
      .attr("y", (d) => d.y - 15)
      .text((d) => "∞");

    simulation.on("tick", () => {
      link
        .attr("x1", (d) => d.source.x)
        .attr("y1", (d) => d.source.y)
        .attr("x2", (d) => d.target.x)
        .attr("y2", (d) => d.target.y);

      node.attr("cx", (d) => d.x).attr("cy", (d) => d.y);

      svg
        .selectAll(".labels text")
        .attr("dx", (d) => (d.source.x + d.target.x) / 2)
        .attr("dy", (d) => (d.source.y + d.target.y) / 2);

      svg
        .selectAll(".distances text")
        .attr("x", (d) => d.x)
        .attr("y", (d) => d.y - 15);
    });

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

<svg width="800" height="600"></svg>

<style>
  .links line {
    stroke: #999;
    stroke-opacity: 0.6;
  }

  .nodes circle {
    stroke: #fff;
    stroke-width: 1.5px;
  }

  .labels text,
  .distances text {
    fill: #000;
    font-size: 12px;
    pointer-events: none;
  }
</style>
