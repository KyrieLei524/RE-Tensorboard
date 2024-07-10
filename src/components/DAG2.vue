<template>
  <div class="graph">
    <svg id="graphSvg" width="800" height="600"></svg>
  </div>
  <div class="tooltip"></div>
</template>

<script setup>
import * as d3 from "d3";
import dagreD3 from "dagre-d3";
import { ref, onMounted } from "vue";
import nodes from "C:/Users/qzh18/Desktop/RE-Tensorboard-main/src/data/nodes.json";

if (!nodes || nodes.length === 0) {
  console.error("Failed to load nodes JSON");
}

const digraphdHandle = () => {
  console.log("Initializing graph rendering");

  const g = new dagreD3.graphlib.Graph().setGraph({});
  g.setDefaultEdgeLabel(() => { return {}; });
  g.graph().rankdir = "BT";

  const map = new Map();
  for (const node of nodes) {
    map.set(node.name, node);
  }

  function updateArrowMarker(svg, color) {
    const defs = svg.select("defs");
    const markerId = `arrow-${color.replace(/[^a-zA-Z0-9]/g, "-")}`;
    const existingMarker = defs.select(`#${markerId}`);

    if (!existingMarker.empty()) {
      return `url(#${markerId})`;
    }

    const marker = defs.append("marker")
      .attr("id", markerId)
      .attr("viewBox", "0 0 10 10")
      .attr("refX", "9")
      .attr("refY", "5")
      .attr("markerUnits", "strokeWidth")
      .attr("markerWidth", "8")
      .attr("markerHeight", "6")
      .attr("orient", "auto-start-reverse");

    marker.append("path")
      .attr("d", "M 0 0 L 10 5 L 0 10 z")
      .attr("fill", color);

    return `url(#${markerId})`;
  }

  function addNodesAndEdges(g, nodes, map, svg) {
    console.log("Adding nodes and edges");

    nodes.forEach((node) => {
      if (node.layer_num === 1 && node.name !== "NoOp") {
        let shape = 'rect';
        if (node.node_type === 0) shape = 'circle';
        else if (node.node_type === 1) shape = 'ellipse';
        else if (node.node_type === 3) shape = 'polygon';

        g.setNode(node.name, {
          shape: shape,
          label: node.name.split("/").pop(),
          rx: 5,
          ry: 5,
          labelStyle: "margin-right: 50px; font-weight: bold; font-size: 12px;",
        });

        const output_edges = node.output_edges;
        const output_map = new Map();

        output_edges.forEach((edge) => {
          var output = edge.output;
          if (output === "NoOp") return;
          var output_node = map.get(output);
          if (output_node.layer_num !== 1) return;
          output_map.set(output, (output_map.get(output) || 0) + 1);
        });

        const max_value = Math.max(...output_map.values());

        output_map.forEach((value, key) => {
          const color = d3.interpolateLab("lightgray", "black")(value / max_value);
          const arrowMarkerUrl = updateArrowMarker(svg, color);
          g.setEdge(node.name, key, {
            label: value,
            labelpos: "c",
            style: `stroke-width: ${1 + value}px; stroke: ${color}; fill: none; marker-end: ${arrowMarkerUrl};`
          });
        });
      }
    });
  }

  const svg = d3.select("#graphSvg");
  const inner = svg.append("g");

  const zoom = d3.zoom().on("zoom", function (event) {
    inner.attr("transform", event.transform);
  });

  svg.call(zoom);
  const render = new dagreD3.render();

  addNodesAndEdges(g, nodes, map, svg);
  render(inner, g);

  svg.attr("width", 1000);
  svg.attr("height", 600);

  inner.selectAll("g.node")
    .on("mouseover", showTooltip)
    .on("mouseout", hideTooltip)
    .on("click", (event, d) => handleNodeClick(event, d, g, map, render, inner, svg));
};

onMounted(() => {
  console.log("Component mounted");
  digraphdHandle();
});

const showTooltip = (event) => {
  console.log("Tooltip on:", event);
};

const hideTooltip = () => {
  console.log("Tooltip off");
};

const handleNodeClick = (event, d, g, map, render, inner, svg) => {
  console.log("Node clicked:", d);
  const node = map.get(d);

  if (node && node.children && node.children.length > 0) {
    const childNodes = node.children.map(name => map.get(name));
    addNodesAndEdges(g, childNodes, map, svg);
    render(inner, g);
  }
};

</script>

<style scoped>
.graph {}

#graphSvg {}

.node circle, .node polygon {
  stroke: #333;
  fill: #fff;
  stroke-width: 1.5px;
  transition: all 0.3s ease-in-out;
}

.node rect {
  stroke: #333;
  fill: #ffbaba;
  stroke-width: 1.5px;
  transition: all 0.3s ease-in-out;
}

.node ellipse {
  stroke: #333;
  fill: #f2ecb5;
  stroke-width: 1.5px;
  transition: all 0.3s ease-in-out;
}

.node:hover rect {
  stroke: #333;
  fill: #e98c8c;
  stroke-width: 1.5px;
  transform: scale(1.1);
}

.node:hover ellipse {
  stroke: #333;
  fill: #fdfaa0;
  stroke-width: 1.5px;
  transform: scale(1.1);
}

.node:hover circle, .node:hover polygon {
  stroke: #333;
  fill: #f5f5f5;
  stroke-width: 1.5px;
  transform: scale(1.1);
}

.edgePath {
  stroke: #5d5d5d;
  fill: none;
  stroke-width: 2.5px;
}
</style>