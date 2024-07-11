<template>
  <div>
    <div class="info-panel" v-if="selectedNode">
      <h3>Node Information</h3>
      <p><strong>attr:</strong></p>
      <div v-for="(value, key) in selectedNode.raw_info.attr" :key="key" style="margin-left: 20px;">
        <p><strong>{{ key }}:</strong> {{ value }}</p>
      </div>
      <p><strong>input:</strong></p>
    </div>

  </div>

  <div>
    <svg ref="svg" :width="width" :height="height" style="background-color: aliceblue;"></svg>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import * as d3 from 'd3';
import dagre from 'dagre';
import node_data from "../../../../nodes.json";

const width = 800;
const height = 800;
const svg = ref(null);

// 定义节点的固定大小 目前全部都是矩形
const nodeWidth = 100;
const nodeHeight = 100;

// 创建一个图的层次信息 图之间有很多关系
const g = new dagre.graphlib.Graph();

// 开始处理node_data 
// 首先根据每个node的json完整信息建立存图所有的数据节点信息 先不存图的子节点信息 先把所有的节点建立起来 并存到map里面再说
var my_nodes = new Map();

var root_node = {
  label: 'root',
  children: [],
  expanded: true,
  dagreGraph: null,
  width: 0,
  height: 0,
  edges: [],
  x: 0,
  y: 0,
  parent: null,
  raw_info: {
    attr: null
  }
};
my_nodes.set('root', root_node);

node_data.forEach(element => {
  var node = {
    label: element.name,
    children: [],
    expanded: false,
    dagreGraph: null,
    width: 0,
    height: 0,
    edges: [],
    x: 0,
    y: 0,
    parent: element.parent,
    raw_info: element
  };
  if (node.parent === null || element.layer_num === 1) {
    node.parent = 'root';
  }
  my_nodes.set(element.name, node);
});

// 处理children的信息
node_data.forEach(element => {
  if (element.layer_num === 1) {
    root_node.children.push(my_nodes.get(element.name));
  }
});

node_data.forEach(element => {
  var children = element.children;
  if (children === null) {
    children = [];
    element.children = children;
  }
  children.forEach(child => {
    var node = my_nodes.get(element.name);
    node.children.push(my_nodes.get(child));
  });
});

// 处理边的信息
var my_edges = new Map();
const edge_example = {
  source: 'A',
  target: 'B',
  info_array: []
};

node_data.forEach(element => {
  var output_edges = element.output_edges;
  output_edges.forEach(edge => {
    const input_name = edge.input;
    const output_name = edge.output;
    const edge_key = input_name + '->' + output_name;
    if (!my_edges.has(edge_key)) {
      const edge_obj = {
        source: input_name,
        target: output_name,
        info_array: [edge]
      };
      my_edges.set(edge_key, edge_obj);
    } else {
      my_edges.get(edge_key).info_array.push(edge);
    }
  });
});

my_edges.forEach((value, key) => {
  const source = value.source;
  const target = value.target;
  if (source === 'NoOp' || target === 'NoOp') {
    return;
  }
  const edge_obj = {
    source: source,
    target: target
  };
  if (my_nodes.get(source).parent === my_nodes.get(target).parent) {
    const parent_name = my_nodes.get(source).parent;
    my_nodes.get(parent_name).edges.push(edge_obj);
  }
});

var map = my_nodes;

console.log(root_node);

var selectedNode = ref(root_node);

function compute_node(node) {
  if (node.children.length == 0 || !node.expanded) {
    node.width = nodeWidth;
    node.height = nodeHeight;
    return;
  }
  var nodes = node.children;
  for (var i = 0; i < nodes.length; i++) {
    var n = nodes[i];
    if (n.expanded) {
      compute_node(n);
    } else {
      n.width = nodeWidth;
      n.height = nodeHeight;
    }
  }
  const g = new dagre.graphlib.Graph();
  g.setGraph({});
  g.setDefaultEdgeLabel(() => ({}));
  g.graph().rankdir = "BT";
  for (var i = 0; i < nodes.length; i++) {
    var n = nodes[i];
    g.setNode(n.label, { label: n.label, width: n.width, height: n.height });
  }
  for (var i = 0; i < node.edges.length; i++) {
    var e = node.edges[i];
    g.setEdge(e.source, e.target);
  }
  dagre.layout(g);
  node.dagreGraph = g;
  var graph = g.graph();
  node.width = graph.width * 1.5;
  node.height = graph.height * 1.5;
}

function draw_root(root, svgGroup) {
  if (root.children.length == 0 || !root.expanded) {
    return;
  }
  var g = root.dagreGraph;

  svgGroup.append("defs")
    .append("marker")
    .attr("id", "arrowhead")
    .attr("viewBox", "0 0 10 10")
    .attr("refX", 10)
    .attr("refY", 5)
    .attr("markerWidth", 6)
    .attr("markerHeight", 6)
    .attr("orient", "auto")
    .append("path")
    .attr("d", "M 0 0 L 10 5 L 0 10 z")
    .attr("fill", "#333");

  g.edges().forEach(edge => {
    const points = g.edge(edge).points;
    const lineGenerator = d3.line()
      .x(d => d.x + root.x)
      .y(d => d.y + root.y)
      .curve(d3.curveBasis);

    svgGroup.append("path")
      .attr("d", lineGenerator(points))
      .attr("class", "edgePath")
      .attr("stroke", "#333")
      .attr("stroke-width", 1.5)
      .attr("fill", "none")
      .attr("marker-end", "url(#arrowhead)");
  });

  g.nodes().forEach(node => {
    const n = g.node(node);
    const nodeGroup = svgGroup.append("g")
      .attr("transform", `translate(${n.x - n.width / 2 + root.x}, ${n.y - n.height / 2 + root.y})`);
    const node_obj = map.get(node);
    node_obj.x = n.x - n.width / 2 + root.x;
    node_obj.y = n.y - n.height / 2 + root.y;

    nodeGroup.append("rect")
      .attr("width", n.width)
      .attr("height", n.height)
      .attr("stroke", "#333")
      .attr("fill", "#fff")
      .on("dblclick", function (event) {
        if (node_obj.children.length === 0) {
          return;
        }
        node_obj.expanded = !node_obj.expanded;
        compute_node(root);
        svgGroup.selectAll("*").remove();
        draw_root(root, svgGroup);
      })
      .on("click", function (event) {
        selectedNode.value = node_obj;  // 正确地更新ref对象
        console.log(selectedNode.value);
      });

    nodeGroup.append("text")
      .attr("x", n.width / 2)
      .attr("y", n.height / 2)
      .attr("dy", ".35em")
      .attr("text-anchor", "middle")
      .text(n.label.split('/').pop())
      .on("dblclick", function (event) {
        if (node_obj.children.length === 0) {
          return;
        }
        node_obj.expanded = !node_obj.expanded;
        compute_node(root);
        svgGroup.selectAll("*").remove();
        draw_root(root, svgGroup);
      });
  });

  for (var i = 0; i < root.children.length; i++) {
    var n = root.children[i];
    if (n.expanded) {
      draw_root(n, svgGroup);
    }
  }
}

const renderGraph = () => {
  g.setGraph({});
  g.setDefaultEdgeLabel(() => ({}));
  g.graph().rankdir = "BT";
  g.setNode("A", { label: "A", width: nodeWidth, height: nodeHeight });
  g.setNode("B", { label: "B", width: nodeWidth, height: nodeHeight });
  g.setNode("C", { label: "C", width: nodeWidth, height: nodeHeight });
  g.setEdge("A", "B");
  g.setEdge("B", "C");

  dagre.layout(g);

  const svgElement = d3.select(svg.value);
  const svgGroup = svgElement.append("g");

  compute_node(root_node);
  draw_root(root_node, svgGroup);
};

onMounted(() => {
  renderGraph();
});
</script>
