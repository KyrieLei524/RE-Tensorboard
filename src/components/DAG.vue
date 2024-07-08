<script setup>
import * as d3 from "d3";
import dagreD3 from "dagre-d3";
import { onMounted } from "vue";
import nodes from "../data/nodes.json";

const digraphdHandle = () => {
  // 初始化 graph
  const g = new dagreD3.graphlib.Graph().setGraph({});

  // 设置默认边标签
  g.setDefaultEdgeLabel(function () {
    return {};
  });

  // 方向
  g.graph().rankdir = "BT";

  //处理json中的node信息
  var map = new Map();
    for (const node of nodes) {
        map.set(node.name, node);
    }

  // 添加节点
  for (const node of nodes) {
    console.log(node.layer_num);
    if(node.layer_num === 1){
        g.setNode(node.name, {
            label: node.name,
        });
        var output_edges = node.output_edges;
        for (const edge of output_edges) {
            var output = edge.output;
            var output_node = map.get(output);
            if(output_node.layer_num === 1){
                g.setEdge(node.name, output);
            }
        }
        

    }
  }
  
//   g.setNode("A", {
//     shape: "circle",
//     label: "A",
//     style: "fill: #afa",
//   });
//   g.setNode("B", { label: "label：B" });
//   g.setNode("C", { label: "CCC" });
//   g.setNode("D", { label: "D" });
//   g.setNode("E", { label: "E" });
//   g.setNode("F", { label: "F" });

//   // 绘制连接线
//   g.setEdge("A", "B", {
//     arrowhead: "vee",
//     arrowheadStyle: "fill: #f66",
//     style: "stroke: #f66;stroke-width: 1.5px;stroke-dasharray: 5, 5;",
//   });
//   g.setEdge("A", "C", {
//     arrowhead: "undirected",
//     curve: d3.curveBasis,
//   });
//   g.setEdge("B", "D");
//   g.setEdge("C", "E");
//   g.setEdge("D", "F");
//   g.setEdge("E", "F");


  const svg = d3.select("#graphSvg");

  var inner = svg.append("g");

  var zoom = d3.zoom().on("zoom", function () {
    inner.attr("transform", d3.event.transform);
  });
    svg.call(zoom);

  const render = new dagreD3.render();

  // Center the graph
  const svgGroup = svg.append("g");
  render(d3.select("svg g"), g);
  
  svg.attr("width", 1000);
  svg.attr("height", 600);








  
//   // 添加点击事件处理函数
//   d3.selectAll(".node").on("click", function (d) {
//     const nodeId = d3.select(this).select("text").text();
//     if (nodeId === "label：B") {
//       // 放大节点
//       d3.select(this).select("rect")
//         .transition()
//         .attr("width", 100)
//         .attr("height", 100)
//         .attr("x", -50)
//         .attr("y", -50);
        
//     }
//   });
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
.node rect {
  stroke: #999;
  fill: #fff;
  stroke-width: 1.5px;
}
.node circle {
  stroke: #999;
  /* fill: #fff; */
  stroke-width: 1.5px;
}
.edgePath path.path {
  stroke: #333;
  fill: none;
  stroke-width: 1.5px;
}
</style>
