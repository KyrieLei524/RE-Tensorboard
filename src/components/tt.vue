<template>
  <div class="node-info-panel" v-if="selectedNode">
    <h3>{{ selectedNode.raw_info['name'] }}</h3>
    <p><strong>Operation:</strong> {{ selectedNode.raw_info['op'] }}</p>
    <div class="attributes-section">
      <h4>Attributes ({{ Object.keys(selectedNode.raw_info['attr']).length }})</h4>
      <div v-for="(value, key) in selectedNode.raw_info['attr']" :key="key" class="attribute">
        <p><strong>{{ key }}:</strong> {{ value }}</p>
      </div>
    </div>
    <div class="inputs-section" v-if="selectedNode">
      <h4>Inputs ({{ selectedNode.input_num }})</h4>
      <div v-for="input in selectedNode.raw_info['input_edges']" :key="input" class="input">
        {{ input['input'] }}
      </div>
    </div>
    <div class="outputs-section" v-if="selectedNode">
      <h4>Outputs ({{ selectedNode.output_num }})</h4>
      <div v-for="output in selectedNode.raw_info['output_edges']" :key="output" class="output">
        {{ output['output'] }}
      </div>
    </div>
  </div>
  <div>
    <svg ref="svg" :width="width" :height="height"></svg>
  </div>

</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import * as d3 from 'd3';
import dagre from 'dagre';
import node_data from "../../../../nodes.json";
import { componentToSlot } from 'element-plus/es/components/table-v2/src/utils.mjs';
const width = 1200;
const height = 800;
const svg = ref(null);


// 定义节点的固定大小 目前全部都是矩形
const nodeWidth = 100;
const nodeHeight = 100;

// 创建一个图的层次信息 图之间有很多关系

const g = new dagre.graphlib.Graph();

//开始处理node_data 
//首先根据每个node的json完整信息建立存图所有的数据节点信息 先不存图的子节点信息 先把所有的节点建立起来 并存到map里面再说
var my_nodes = new Map();

var root = {
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
    attr: ""
  },
  input_num: 0,
  output_num: 0
};
my_nodes.set('root', root);


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
    raw_info: element,
    input_num: element.input_edges.length,
    output_num: element.output_edges.length
  };
  // console.log(element.input_edges.length);
  if (node.parent === null || element.layer_num === 1) {
    node.parent = 'root';
  }
  my_nodes.set(element.name, node);
});
//现在开始处理children的信息
//首先把root的children信息处理一下
node_data.forEach(element => {
  if (element.layer_num === 1) {
    root.children.push(my_nodes.get(element.name));
  }
});

//然后处理每个节点的children信息
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

//现在已经处理了所有node的父子关系，包括root节点 但是还没有添加边的信息
//现在开始处理边的信息 对于每一个节点要统计他的子图的边的信息

//建立边的映射信息 要记录所有的边 后期还需要记录边的points信息 暂时一个obj存储边
var my_edges = new Map();
const edge_example = {
  source: 'A',
  target: 'B',
  info_array: [] //这里主要是为了处理多边的情况 直接把原始json文件里面 source和target的 ouptput_edges放进来就行
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
  })
});

//现在已经处理了所有的边的信息，接下来要把边的信息添加到node里面去
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
  //判断这条边的起始点和终止点是不是拥有相同的parent
  if (my_nodes.get(source).parent === my_nodes.get(target).parent) {
    //如果父节点相同 说明他们存在于同一个子图当中 那么就把这条边添加到子图的边里面去
    const parent_name = my_nodes.get(source).parent;
    my_nodes.get(parent_name).edges.push(edge_obj);
  }
});



var map = my_nodes;



var selectedNode = ref(root);

function compute_node(node) {
  if (node.children.length === 0 || !node.expanded) {
    node.width = nodeWidth;
    node.height = nodeHeight;
    return;
  }
  //递归更新所有子节点的信息
  node.children.forEach(childNode => {
    if (childNode.expanded) {
      compute_node(childNode);//如果这个节点是展开的，那么就递归更新这个节点的信息
    } else { //如果这个节点是折叠的，那么就直接用固定的大小赋值
      childNode.width = nodeWidth;
      childNode.height = nodeHeight;
    }
  });
  //至此, 已经算出所有子节点的大小信息，使用dagre进行布局
  const g = new dagre.graphlib.Graph();
  g.setGraph({});
  g.setDefaultEdgeLabel(() => ({}));
  // 方向
  g.graph().rankdir = "BT";

  // Add nodes to the graph
  node.children.forEach(childNode => {
    g.setNode(childNode.label, { label: childNode.label, width: childNode.width, height: childNode.height });
  });

  node.edges.forEach(e => {
    g.setEdge(e.source, e.target);
  });
  dagre.layout(g);

  //布局完毕 将这个节点的布局信息存储到node中
  node.dagreGraph = g;
  //更新这个节点的大小信息
  var graph = g.graph();
  node.width = graph.width * 1.5;
  node.height = graph.height * 1.5;
}
function getNodeShape(node) {
  if (node.raw_info.node_type === 0) {
    return 'circle';
  } else if (node.raw_info.node_type === 1) {
    return 'ellipse';
  } else {
    return 'rect';
  }
}

function draw_root(root, svgGroup) {
  if (root.children.length === 0 || !root.expanded) {
    return;
  }
  //先画这个节点的图
  var g = root.dagreGraph;

  // 定义箭头标记
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

  // 绘制边并添加箭头
  g.edges().forEach(edge => {
    const points = g.edge(edge).points;
    const lineGenerator = d3.line()
      .x(d => d.x + root.x)
      .y(d => d.y + root.y)
      .curve(d3.curveBasis); // 使用曲线生成器

    svgGroup.append("path")
      .attr("d", lineGenerator(points))
      .attr("class", "edgePath")
      .attr("stroke", "#333")
      .attr("stroke-width", 1.5)
      .attr("fill", "none")
      .attr("marker-end", "url(#arrowhead)"); // 添加箭头
  });

  g.nodes().forEach(nodeLabel => {
    const n = g.node(nodeLabel);
    const node_obj = my_nodes.get(nodeLabel);
    const shape = getNodeShape(node_obj);

    const nodeGroup = svgGroup.append("g")
      .attr("id", nodeLabel.replace(/\//g, "-")) //建立id,方便后续再次寻找，注意这里名字不能带反斜杠，所以需要替换
      .attr("transform", `translate(${n.x - n.width / 2 + root.x}, ${n.y - n.height / 2 + root.y})`);
    // .attr("class", "node");

    node_obj.x = n.x - n.width / 2 + root.x;
    node_obj.y = n.y - n.height / 2 + root.y;

    if (shape === 'circle') {
      nodeGroup.append("circle")
        .attr("cx", n.width / 2)
        .attr("cy", n.height / 2)
        .attr("r", Math.min(n.width, n.height) / 2)
        .attr("stroke", "#333")
        .attr("fill", "#fff")
        .on("dblclick", function (event) {
          console.log("dbclick");
          if (node_obj.children.length === 0) {
            return;
          }
          node_obj.expanded = !node_obj.expanded;
          compute_node(root);
          svgGroup.selectAll("*").remove();
          draw_root(root, svgGroup);
        })
        .on("click", function (event) {
          if (selectedNode.value.label === node_obj.label) {
            console.log("same node");
            selectedNode.value = root;
            return;
          }
          selectedNode.value = node_obj;
        })
        .on("mouseover", function (event) {
          //以中心为基准，放大1.1倍
          d3.select(this).attr("fill", "#f5f5f5");

        })
        .on("mouseout", function (event) {
          d3.select(this).attr("fill", "#fff");
        });
    } else if (shape === 'ellipse') {
      nodeGroup.append("ellipse")
        .attr("cx", n.width / 2)
        .attr("cy", n.height / 2)
        .attr("rx", n.width / 2)
        .attr("ry", n.height / 2)
        .attr("stroke", "#333")
        .attr("fill", "#fff")
        .on("dblclick", function (event) {
          console.log("dbclick");
          if (node_obj.children.length === 0) {
            return;
          }
          node_obj.expanded = !node_obj.expanded;
          compute_node(root);
          svgGroup.selectAll("*").remove();
          draw_root(root, svgGroup);
        })
        .on("click", function (event) {
          if (selectedNode.value.label === node_obj.label) {
            console.log("same node");
            selectedNode.value = root;
            return;
          }
          selectedNode.value = node_obj;
        })
        .on("mouseover", function (event) {
          //以中心为基准，放大1.1倍
          d3.select(this).attr("fill", "#f5f5f5");

        })
        .on("mouseout", function (event) {
          d3.select(this).attr("fill", "#fff");
        });
    } else {
      nodeGroup.append("rect")
        .attr("width", n.width)
        .attr("height", n.height)
        .attr("rx", 10)
        .attr("ry", 10)
        .attr("stroke", "#333")
        .attr("fill", "#fff")
        .on("dblclick", function (event) {
          console.log("dbclick");
          if (node_obj.children.length === 0) {
            return;
          }
          node_obj.expanded = !node_obj.expanded;
          compute_node(root);
          svgGroup.selectAll("*").remove();
          draw_root(root, svgGroup);
        })
        .on("click", function (event) {
          if (selectedNode.value.label === node_obj.label) {
            console.log("same node");
            selectedNode.value = root;
            return;
          }
          selectedNode.value = node_obj;
        })
        .on("mouseover", function (event) {
          //以中心为基准，放大1.1倍
          d3.select(this).attr("fill", "#f5f5f5");

        })
        .on("mouseout", function (event) {
          d3.select(this).attr("fill", "#fff");
        });
    }

    nodeGroup.append("text")
      .attr("x", n.width / 2)
      .attr("y", n.height / 2)
      .attr("dy", ".35em")
      .attr("text-anchor", "middle")
      .text(nodeLabel.split('/').pop()).on("dblclick", function (event) {
        console.log("dbclick");
        if (node_obj.children.length === 0) {
          return;
        }
        node_obj.expanded = !node_obj.expanded;
        compute_node(root);
        svgGroup.selectAll("*").remove();
        draw_root(root, svgGroup);
      });

    //添加点击事件

  });
  //至此 首先画出上层的节点 然后递归的画出每个节点的子图

  //递归画每个子节点的图
  root.children.forEach(child => {
    draw_root(child, svgGroup);
  });
}

watch(selectedNode, (newVal, oldVal) => {

  if (oldVal === newVal) {
    selectedNode.value = null;
    return;
  }
  //更新新选中节点的样式
  if (newVal.raw_info['name']) {
    const id = newVal.raw_info['name'].toString().replace(/\//g, "-");
    console.log(id);

    const node = d3.select(`#${id}`);
    node.select("rect").attr("stroke", "#ff0000").attr("fill", "#f5f5f5");
  }

  //更新旧选中节点的样式
  if (oldVal.raw_info['name']) {
    const old_id = oldVal.raw_info['name'].toString().replace(/\//g, "-");
    const old_node = d3.select(`#${old_id}`);
    old_node.select("rect").attr("stroke", "#333").attr("fill", "#fff");
  }

});


onMounted(() => {

  const svgEl = d3.select(svg.value);
  const zoomGroup = svgEl.append("g");
  const svgGroup = zoomGroup.append("g");
  //所有内容都画在svgGroup上
  compute_node(root);
  draw_root(root, svgGroup);

  const zoom = d3.zoom().on("zoom", function (event) {
    svgGroup.attr("transform", d3.event.transform);
  });
  svgEl.call(zoom).on("dblclick.zoom", null);

});
</script>




<style scoped>
.edgePath {
  stroke: #333;
  fill: none;
  stroke-width: 1.5px;
}

.node-info-panel {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 400px;
  padding: 10px;
  border: 1px solid #ccc;
  background-color: white;
}

.attributes-section,
.inputs-section,
.outputs-section {
  margin-top: 10px;
}

.attribute,
.input,
.output {
  padding: 5px;
  background-color: #ffffff;
  border-radius: 3px;
  margin-top: 5px;
}

.node rect,
.node circle,
.node ellipse,
.node polygon {
  stroke: #333;
  fill: #fff;
  stroke-width: 1.5px;
  transition: all 0.3s ease-in-out;
  cursor: pointer;
  /* Adds cursor pointer on hover */
}

.node rect:hover,
.node circle:hover,
.node ellipse:hover,
.node polygon:hover {
  fill: #b3d9ff;
  /* Lighter blue on hover */
}

.node rect.active,
.node circle.active,
.node ellipse.active,
.node polygon.active {
  fill: #4d94ff;
  /* Darker blue on click */
}
</style>