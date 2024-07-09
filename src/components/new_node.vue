<template>
    <div style="border: 1px solid #ccc; padding: 20px; width: 600px">
        <svg class="dagre" width="600" height="600">
            <g class="container"></g>
        </svg>
        <div ref="tooltip" class="tooltip">
            <div>节点ID：{{ currentNode.id }}</div>
            <div>节点名称：{{ currentNode.nodeName }}</div>
        </div>
    </div>
</template>

<script>
import dagreD3 from 'dagre-d3';
import * as d3 from 'd3';

export default {
    name: 'dagre',
    data() {
        return {
            currentNode: {
                id: null,
                nodeName: '',
            },
            nodes: [
                { id: 0, nodeName: '节点0' },
                { id: 1, nodeName: '节点1' },
                { id: 2, nodeName: '节点2' },
                { id: 3, nodeName: '节点3' },
                { id: 4, nodeName: '节点4' },

            ],
            edges: [
                { start: 1, end: 0 },
                { start: 2, end: 1 },
                { start: 3, end: 2 },
                { start: 4, end: 3 },
            ],
        };
    },
    mounted() {
        this.draw();
    },
    methods: {
        draw() {
            const g = new dagreD3.graphlib.Graph().setGraph({
                rankdir: 'BT',
            }).setDefaultEdgeLabel(() => ({}));

            this.nodes.forEach(node => {
                g.setNode(node.id, {
                    id: node.id,
                    label: node.nodeName,
                    shape: 'rect',
                    style: 'fill:#61b2e4;stroke:#fff',
                    labelStyle: 'fill: #fff;font-weight:bold',
                    rx: 5,
                    ry: 5,
                    paddingBottom: 15,
                    paddingLeft: 20,
                    paddingRight: 20,
                    paddingTop: 15,
                });
            });

            if (this.nodes.length > 1) {
                this.edges.forEach(edge => {
                    g.setEdge(edge.start, edge.end, {
                        style: 'stroke: #0fb2cc; fill: none; stroke-width: 2px',
                        arrowheadStyle: 'fill: #0fb2cc;stroke: #0fb2cc;',
                        arrowhead: 'normal',
                    })
                });
            }

            const container = d3.select('svg.dagre').select('g.container');

            const svg = d3.select('svg.dagre');
            var zoom = d3.zoom().on("zoom", function () {
                container.attr("transform", d3.event.transform);
            });
            svg.call(zoom);

            const render = new dagreD3.render();
            render(container, g);

            // 添加鼠标事件处理器
            container.selectAll("g.node")
                .on("mouseover", (event, d) => this.showTooltip(event, d, g.node(d)))
                .on("mouseout", () => this.hideTooltip());
        },

        showTooltip(event, nodeId, nodeData) {
            console.log(event);
            console.log(nodeData.x);
            this.currentNode = {
                id: nodeId,
                nodeName: nodeData.label,
            };

            const tooltip = this.$refs.tooltip;
            tooltip.style.display = 'block';
            
            tooltip.style.left = `${nodeData.x}px`;
            tooltip.style.top = `${nodeData.y}px`;
        },
        hideTooltip() {
            const tooltip = this.$refs.tooltip;
            tooltip.style.display = 'none';
        }
    },
};
</script>

<style scoped>
.tooltip {
    position: absolute;
    font-size: 12px;
    background-color: white;
    border-radius: 3px;
    box-shadow: rgb(174, 174, 174) 0px 0px 10px;
    cursor: pointer;
    display: none;
    padding: 10px;
    pointer-events: none;
}

.tooltip>div {
    padding: 10px;
}
</style>
