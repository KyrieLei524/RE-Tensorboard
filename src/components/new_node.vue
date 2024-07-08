<script setup>
import * as d3 from "d3";
import dagreD3 from "dagre-d3";
import { onMounted } from "vue";

const yangshi = { label: "F", width: 90, width: 100 }

const digraphdHandle = () => {
    const g = new dagreD3.graphlib.Graph().setGraph({});
    g.setDefaultEdgeLabel(function () {
        return {};
    });
    //方向
    g.graph().rankdir = "LR";
    //绘制节点
    g.setNode("A", {
        shape: "circle",
        label: "A",
        style: "fill: #afa",
    });
    g.setNode("B", { label: "label：B" });
    g.setNode("C", { label: "CCC" });
    g.setNode("D", { label: "D" });
    g.setNode("E", { label: "E" });
    g.setNode("F", yangshi);
    //绘制连接线
    g.setEdge("A", "B", {
        arrowhead: "vee",
        arrowheadStyle: "fill: #f66",
        style: "stroke: #f66;stroke-width: 1.5px;stroke-dasharray: 5, 5;",
    });
    g.setEdge("A", "C", {
        arrowhead: "undirected",
        curve: d3.curveBasis,
    });
    g.setEdge("B", "D");
    g.setEdge("C", "E");
    g.setEdge("D", "F");
    g.setEdge("E", "F");
    
    var svg = d3.select("#graphSvg");
    var inner = svg.append("g");

    var zoom = d3.zoom().on("zoom", function () {
        inner.attr("transform", d3.event.transform);
    });
    svg.call(zoom);

    const render = new dagreD3.render();

    // 渲染图形 这一步是真正画图的地方
    render(inner, g);

    // 添加点击事件处理函数
    d3.selectAll(".node").on("click", function (d) {
        const nodeId = d3.select(this).select("text").text();
        if (nodeId === "F") {
            yangshi.width = 10 + yangshi.width;
            // 先清空当前的d3图 重新绘制
            render(inner, g);

        }
    });


    svg.attr("width", 1000);
    svg.attr("height", 600);
};
onMounted(() => {
    digraphdHandle();
});
</script>

<template>
    <div class="graph"><svg id="graphSvg" width="800" height="600"></svg></div>
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