<script setup>
import * as d3 from "d3";
import dagreD3 from "dagre-d3";
import { onMounted } from "vue";
import nodes from "../../../../nodes.json";

const digraphdHandle = () => {
  // 初始化 graph
  const g = new dagreD3.graphlib.Graph().setGraph({});

  // 设置默认边标签
  g.setDefaultEdgeLabel(function () {
    return {};
  });

  // 方向
  g.graph().rankdir = "BT";

  // 处理json中的node信息
  var map = new Map();
  for (const node of nodes) {
    map.set(node.name, node);
  }

  // 添加节点
  for (const node of nodes) {
    console.log(node.layer_num);

    if (node.layer_num === 1 && node.name !== "NoOp") {
      var shape = 'rect';

      // 加入点 处理Const 节点和 NoOp节点
      if (node.node_type === 0) {
        shape = 'circle';
      } else if (node.node_type === 1) {
        shape = 'ellipse';
      } else if (node.node_type === 2) {
        shape = 'rect';
      } else if (node.node_type === 3) {
        shape = 'polygon';
      }

      g.setNode(node.name, {
        shape: shape,
        label: node.name,
      });

      // 加入边
      var output_edges = node.output_edges;
      for (const edge of output_edges) {
        var output = edge.output;
        if (output === "NoOp") continue;
        var output_node = map.get(output);
        if (output_node.layer_num === 1) {
          g.setEdge(node.name, output);
        }
      }
    }
  }

  // g.nodes().forEach(function (v) {
  //   var node = g.node(v);
  //   console.log(node);
  // });

  var svg = d3.select("#graphSvg");

  var inner = svg.append("g");

  var zoom = d3.zoom().on("zoom", function () {
    inner.attr("transform", d3.event.transform);
  });
  svg.call(zoom);

  const render = new dagreD3.render();

  // 渲染图形
  render(inner, g);


  svg.attr("width", 1000);
  svg.attr("height", 600);
};

onMounted(() => {
  digraphdHandle();
});
</script>

<template>
  <div class="graph">
    <svg id="graphSvg" width="800" height="600"></svg>
  </div>
</template>

<style>
.graph {
  background-color: antiquewhite;
}

#graphSvg {
  background-color: #e5ebd3;
}

.node rect,
.node circle,
.node ellipse,
.node polygon {
  stroke: #333;
  fill: #fff;
  stroke-width: 1.5px;
}

.edgePath path.path {
  stroke: #333;
  fill: none;
  stroke-width: 1.5px;
}
</style>
