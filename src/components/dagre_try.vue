<template>
  <div>
    <svg ref="svg" :width="width" :height="height" style="background-color: aliceblue;"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as d3 from 'd3';
import dagre from 'dagre';

const width = 1000;
const height = 1000;
const svg = ref(null);

// 定义节点的固定大小 目前全部都是矩形
const nodeWidth = 100;
const nodeHeight = 100;

// 创建一个图的层次信息 图之间有很多关系

const g = new dagre.graphlib.Graph();


var A1 = {
  label: 'A1',
  children: [],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [],
  x: 0,
  y: 0
};

var A2 = {
  label: 'A2',
  children: [],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [],
  x: 0,
  y: 0
};
var A = {
  label: 'A',
  children: [A1, A2
  ],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [{
    source: 'A2',
    target: 'A1',
  }],
  x: 0,
  y: 0
};
var B1 = {
  label: 'B1',
  children: [],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [],
  x: 0,
  y: 0
};
var B2 = {
  label: 'B2',
  children: [],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [],
  x: 0,
  y: 0
};

var B = {
  label: 'B',
  children: [B1, B2
  ],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [{
    source: 'B1',
    target: 'B2',
  }],
  x: 0,
  y: 0

};
var C = {
  label: 'C',
  children: [
  ],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [],
  x: 0,
  y: 0
};

var root = {
  label: 'root',
  children: [A, B, C
  ],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [{
    source: 'A',
    target: 'B',
  }, {
    source: 'C',
    target: 'B',
  }, {
    source: 'A',
    target: 'C',

  }],
  x: 0,
  y: 0
};

const map = new Map();
map.set('A', A);
map.set('B', B);
map.set('C', C);
map.set('A1', A1);
map.set('A2', A2);
map.set('root', root);
map.set('B1', B1);
map.set('B2', B2);

function compute_node(node) {

  if (node.children.length == 0 || !node.expanded) {
    node.width = nodeWidth;
    node.height = nodeHeight;
    return;
  }
  var nodes = node.children; // 一个存子节点的数组，但是没有节点的大小信息，只有label
  //递归更新所有子节点的信息
  for (var i = 0; i < nodes.length; i++) {
    var n = nodes[i];
    console.log(n);
    if (n.expanded) {
      compute_node(n); //如果这个节点是展开的，那么就递归更新这个节点的信息
    } else { //如果这个节点是折叠的，那么就直接用固定的大小赋值
      n.width = nodeWidth;
      n.height = nodeHeight;
    }
  }
  //至此, 已经算出所有子节点的大小信息，使用dagre进行布局
  const g = new dagre.graphlib.Graph();
  g.setGraph({});
  g.setDefaultEdgeLabel(() => ({}));
  // Add nodes to the graph
  for (var i = 0; i < nodes.length; i++) {
    var n = nodes[i];
    g.setNode(n.label, { label: n.label, width: n.width, height: n.height });
  }
  console.log(node.edges)
  // Add edges to the graph
  for (var i = 0; i < node.edges.length; i++) {
    var e = node.edges[i];
    g.setEdge(e.source, e.target);
  }
  dagre.layout(g);

  //布局完毕 将这个节点的布局信息存储到node中
  node.dagreGraph = g;
  //更新这个节点的大小信息
  var graph = g.graph();
  node.width = graph.width * 1.1;
  node.height = graph.height * 1.1;
}


function draw_root(root, svgGroup) {
  if (root.children.length == 0 || !root.expanded) {
    return;
  }
  //先画这个节点的图
  var g = root.dagreGraph;
  // Create an SVG group

  // Draw edges
  // console.log(root.label);
  // console.log(g.edges());


  g.edges().forEach(edge => {
    const points = g.edge(edge).points;
    const lineGenerator = d3.line()
      .x(d => d.x)
      .y(d => d.y);

    svgGroup.append("path")
      .attr("d", lineGenerator(points))
      .attr("class", "edgePath")
      .attr("stroke", "#333")
      .attr("stroke-width", 1.5)
      .attr("fill", "none");
  });

  // Draw nodes
  g.nodes().forEach(node => {
    const n = g.node(node);
    console.log(node)
    const nodeGroup = svgGroup.append("g")
      .attr("transform", `translate(${n.x - n.width / 2 + root.x}, ${n.y - n.height / 2 + root.y})`);

    // 画完图以后就应该更新刚刚画的这个节点的位置信息 是绝对位置 因为每次画的时候已经是绝对位置了
    const node_obj = map.get(node);
    node_obj.x = n.x - n.width / 2 + root.x;
    node_obj.y = n.y - n.height / 2 + root.y;

    nodeGroup.append("rect")
      .attr("width", n.width)
      .attr("height", n.height)
      .attr("stroke", "#333")
      .attr("fill", "#fff");

    nodeGroup.append("text")
      .attr("x", n.width / 2)
      .attr("y", n.height / 2)
      .attr("dy", ".35em")
      .attr("text-anchor", "middle")
      .text(n.label);
  });
  //至此 首先画出上层的节点 然后递归的画出每个节点的子图

  //递归画每个子节点的图
  for (var i = 0; i < root.children.length; i++) {
    var n = root.children[i];
    if (n.expanded) {
      draw_root(n, svgGroup);
    }
  }

}



const renderGraph = () => {
  // Create a new directed graph

  // Set an object for the graph label
  g.setGraph({});

  // Default to assigning a new object as a label for each new edge.
  g.setDefaultEdgeLabel(() => ({}));

  // Add nodes to the graph
  g.setNode("A", { label: "A", width: nodeWidth, height: nodeHeight });
  g.setNode("B", { label: "B", width: nodeWidth, height: nodeHeight });
  g.setNode("C", { label: "C", width: nodeWidth, height: nodeHeight });

  // Add edges to the graph
  g.setEdge("A", "B");
  g.setEdge("B", "C");
  g.setEdge("C", "A");

  // Layout the graph
  dagre.layout(g);

  // Create an SVG group
  const svgEl = d3.select(svg.value);

  // Create a container for zoom and pan
  const zoomGroup = svgEl.append("g");

  // Create a group for the graph elements
  const svgGroup = zoomGroup.append("g");

  // Draw edges
  g.edges().forEach(edge => {
    const points = g.edge(edge).points;
    const lineGenerator = d3.line()
      .x(d => d.x)
      .y(d => d.y);

    svgGroup.append("path")
      .attr("d", lineGenerator(points))
      .attr("class", "edgePath")
      .attr("stroke", "#333")
      .attr("stroke-width", 1.5)
      .attr("fill", "none");
  });

  // Draw nodes
  g.nodes().forEach(node => {
    const n = g.node(node);
    const nodeGroup = svgGroup.append("g")
      .attr("transform", `translate(${n.x - n.width / 2}, ${n.y - n.height / 2})`);

    nodeGroup.append("rect")
      .attr("width", n.width)
      .attr("height", n.height)
      .attr("stroke", "#333")
      .attr("fill", "#fff");

    nodeGroup.append("text")
      .attr("x", n.width / 2)
      .attr("y", n.height / 2)
      .attr("dy", ".35em")
      .attr("text-anchor", "middle")
      .text(n.label);
  });

  // // Center the graph
  // const xCenterOffset = (svgEl.attr("width") - g.graph().width) / 2;
  // svgGroup.attr("transform", `translate(${xCenterOffset}, 20)`);
  // svgEl.attr("height", g.graph().height + 40);

  // Add zoom behavior
  const zoom = d3.zoom()
    .scaleExtent([0.5, 2])
    .on("zoom", (event) => {
      svgGroup.attr("transform", d3.event.transform);
    });

  svgEl.call(zoom);

};

onMounted(() => {
  // renderGraph();

  const svgEl = d3.select(svg.value);
  const zoomGroup = svgEl.append("g");
  const svgGroup = zoomGroup.append("g");

  compute_node(root);
  console.log(root);
  draw_root(root, svgGroup);
  const zoom = d3.zoom().on("zoom", function (event) {
    svgGroup.attr("transform", d3.event.transform);
  });
  svgEl.call(zoom);

});
</script>




<style scoped>
.edgePath {
  stroke: #333;
  fill: none;
  stroke-width: 1.5px;
}

.node rect {
  stroke: #333;
  fill: #fff;
  stroke-width: 1.5px;
}
</style>
