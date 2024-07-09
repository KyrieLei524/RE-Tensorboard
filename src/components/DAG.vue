<template>
  <div class="graph">
    <svg id="graphSvg" width="800" height="600"></svg>
  </div>
  <div class="tooltip">
    
  </div>
</template>

<script setup>
import * as d3 from "d3";
import dagreD3 from "dagre-d3";
import { ref, onMounted } from "vue";
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
        label: node.name.split("/").pop(),
        rx: 5,
        ry: 5,
        labelStyle: "margin-right: 50px; font-weight: bold; font-size: 12px;",
      });

      // 加入边
      var output_edges = node.output_edges;

       // 加入边
       var output_edges = node.output_edges;
      //统计输出边中是否存在相同目标节点的情况 统计
      var output_map = new Map(); // key: output, value: count
      for (const edge of output_edges) {
        var output = edge.output;
        if (output === "NoOp") continue;
        var output_node = map.get(output);
        if(output_node.layer_num !== 1) continue;
        if (output_map.has(output)) {
          output_map.set(output, output_map.get(output) + 1);
        } else {
          output_map.set(output, 1);
        }
      }
      for(const [key, value] of output_map) {
        g.setEdge(node.name, key, {
          label: value,
          labelpos: "c",
        });
      }
    }
  }

  var svg = d3.select("#graphSvg");

  var inner = svg.append("g");

  var zoom = d3.zoom().on("zoom", function (event) {
    inner.attr("transform", d3.event.transform);
  });
  svg.call(zoom);

  const render = new dagreD3.render();

  // 渲染图形
  render(inner, g);

  svg.attr("width", 1000);
  svg.attr("height", 600);

  inner.selectAll("g.node")
    .on("mouseover", (event) => showTooltip(event))
    .on("mouseout", () => hideTooltip());
};

onMounted(() => {
  digraphdHandle();
});

const showTooltip = (event) => {
  console.log(event);
};

const hideTooltip = () => {
  console.log("hide");
};
</script>

<style>


.graph {
  /* background-color: antiquewhite; */
}

#graphSvg {
  /* background-color: #e5ebd3; */
}

.node rect,
.node circle,
.node ellipse,
.node polygon
{
  stroke: #333;
  fill: #fff;
  stroke-width: 1.5px;
  transition: all 0.3s ease-in-out;
}

.node:hover rect,
.node:hover circle,
.node:hover ellipse,
.node:hover polygon
{
  stroke: #333;
  fill: #f5f5f5;
  stroke-width: 1.5px;
  filter: url(#hover-shadow);
  transform: scale(1.1);
}

.edgePath path.path {
  stroke: #333;
  fill: none;
  stroke-width: 1.5px;
}

</style>
