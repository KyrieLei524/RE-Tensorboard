<template>
    <div>
      <svg ref="svg" :width="width" :height="height"></svg>
    </div>
  </template>
  
  <script>
  import * as d3 from 'd3';
  import dagre from 'dagre';
  
  export default {
    name: 'GraphVisualizer',
    data() {
      return {
        width: 960,
        height: 600,
      };
    },
    methods: {
      renderGraph() {
        // Create a new directed graph
        var g = new dagre.graphlib.Graph();
  
        // Set an object for the graph label
        g.setGraph({});
  
        // Default to assigning a new object as a label for each new edge.
        g.setDefaultEdgeLabel(function() { return {}; });
  
        // Add nodes to the graph
        g.setNode("kspacey",    { label: "Kevin Spacey",  width: 144, height: 100 });
        g.setNode("swilliams",  { label: "Saul Williams", width: 160, height: 100 });
        g.setNode("bpitt",      { label: "Brad Pitt",     width: 108, height: 100 });
        g.setNode("hford",      { label: "Harrison Ford", width: 168, height: 100 });
        g.setNode("lwilson",    { label: "Luke Wilson",   width: 144, height: 100 });
        g.setNode("kbacon",     { label: "Kevin Bacon",   width: 121, height: 100 });
  
        // Add edges to the graph
        g.setEdge("kspacey",   "swilliams");
        g.setEdge("swilliams", "kbacon");
        g.setEdge("bpitt",     "kbacon");
        g.setEdge("hford",     "lwilson");
        g.setEdge("lwilson",   "kbacon");
        g.setEdge("lwilson",   "kspacey");
  
        // Layout the graph
        dagre.layout(g);
  
        // Create an SVG group
        const svg = d3.select(this.$refs.svg);
        const svgGroup = svg.append("g");
  
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
  
        // Center the graph
        const xCenterOffset = (svg.attr("width") - g.graph().width) / 2;
        svgGroup.attr("transform", `translate(${xCenterOffset}, 20)`);
        svg.attr("height", g.graph().height + 40);
      }
    },
    mounted() {
      this.renderGraph();
    }
  };
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
  