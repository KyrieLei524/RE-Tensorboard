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
      <div v-for="input in selectedNode.raw_info['raw_input_nodes']" :key="input" class="input">
        {{ input }}
      </div>
    </div>
    <div class="outputs-section" v-if="selectedNode">
      <h4>Outputs ({{ selectedNode.output_num }})</h4>
      <div v-for="output in selectedNode.raw_info['raw_output_nodes']" :key="output" class="output">
        {{ output }}
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
  output_num: 0,
  all_children_num: 0
};
my_nodes.set('root', root);

//针对每个节点创建对象 
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
    input_num: element.raw_input_nodes.length,
    output_num: element.raw_output_nodes.length,
    all_children_num: 0,
    const_parent: null, //记录子常量节点的合并常量节点
    const_children: [] //记录合并常量节点的子节点 这个里面的子常量节点的对象
  };
  if (!node.parent || element.layer_num === 1) {
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

//处理完节点信息以后 从每个节点的children里面出发 去合并当前子图 指向同一个节点的常量节点
function merge_const_node(root) {
  var bei_merged_nodes = [];
  let new_merged_const_node_arr = [];
  //对当前root节点的所有children里面出发 找到指向相同节点的常量节点 然后 新建一个合并性节点
  root.children.forEach(element => {
    if (element.raw_info['node_type'] === 0 && element.raw_info['output_edges'].length === 1 && element.raw_info['input_edges'].length === 0) {
      bei_merged_nodes.push(element);
      //这是一个常量节点 并且在它父节点的子图
      const const_to_node_name = element.raw_info['output_edges'][0].output;
      const merged_const_node_name = 'const_to_' + const_to_node_name;
      if (my_nodes.has(merged_const_node_name)) {
        my_nodes.get(merged_const_node_name).const_children.push(element);
        element.const_parent = my_nodes.get(merged_const_node_name);
      } else {
        var new_merged_const_node = {
          label: merged_const_node_name,
          children: [],
          expanded: false,
          dagreGraph: null,
          width: 0,
          height: 0,
          edges: [],
          x: 0,
          y: 0,
          parent: element.parent,
          raw_info: {
            "node_type": 0,
            "op": "Const_merge",
            "name": merged_const_node_name,
            "attr": {},
            "raw_input_nodes": [],
            "raw_output_nodes": [element.name]
          },
          input_num: 0,
          output_num: 0,
          all_children_num: 0,
          const_parent: null, //记录子常量节点的合并常量节点
          const_children: [] //记录合并常量节点的子节点 这个里面的子常量节点的对象
        };
        my_nodes.set(merged_const_node_name, new_merged_const_node);
        new_merged_const_node.const_children.push(element);
        element.const_parent = new_merged_const_node;
      }
      if (!new_merged_const_node_arr.includes(my_nodes.get(merged_const_node_name))) {
        new_merged_const_node_arr.push(my_nodes.get(merged_const_node_name));
      }
    }
  });
  //合并完毕 把root子节点中删除这些常量节点
  root.children = root.children.filter(child => !bei_merged_nodes.includes(child));

  //把新建的合并常量节点添加到root的children里面去
  new_merged_const_node_arr.forEach(element => {
    root.children.push(element);
  });

  root.children.forEach(element => {
    if (element.raw_info === null || element.raw_info['node_type'] !== 2) {
      return;
    }
    merge_const_node(element);
  });

};
merge_const_node(root);
//对每个合并常量节点 填补其arrt 里面放子节点的信息
my_nodes.forEach((value, key) => {
  if (value.raw_info['op'] === 'Const_merge') {
    value.const_children.forEach(child => {
      value.raw_info['attr'][child.label] = child.raw_info['attr'];
    });
  }
});
var max_children = 0;
//更新计算每个节点的复杂度 就是节点的all_children_num 用于计算节点的高度
function compute_all_children_num(node) {
  if (node.children.length === 0) {
    node.all_children_num = 1;
    return;
  }
  var sum = 1;
  node.children.forEach(child => {
    compute_all_children_num(child);
    sum += child.all_children_num;
  });
  node.all_children_num = sum;
  if (sum > max_children) {
    max_children = sum;
  }
}

compute_all_children_num(root);

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

var wait_add_edges = []
//现在已经处理了所有的边的信息，接下来要把边的信息添加到node里面去
my_edges.forEach((value, key) => {
  const source = value.source;
  const target = value.target;
  if (source === 'NoOp' || target === 'NoOp') {
    return;
  }
  const edge_obj = {
    source: source,
    target: target,
    info_array: value.info_array,
    edge_num: 0,
    edge_complexity_log_sum: 0
  };
  //判断这条边的起始点和终止点是不是拥有相同的parent
  if (my_nodes.get(source).parent === my_nodes.get(target).parent) {
    const parent_name = my_nodes.get(source).parent;
    //如果父节点相同 说明他们存在于同一个子图当中 那么就把这条边添加到子图的边里面去

    if (my_nodes.get(source).raw_info['node_type'] === 0 && my_nodes.get(source).raw_info['output_edges'].length === 1 && my_nodes.get(source).raw_info['input_edges'].length === 0) {
      //起点是个const节点 那么就要找到这个const节点的合并节点 把边加进去

      const parent_const_node = my_nodes.get(source).const_parent;

      edge_obj.source = parent_const_node.label;
      my_nodes.get(parent_name).edges.push(edge_obj);
      // my_edges.set(edge_obj.source+'->'+edge_obj.target, edge_obj);
      wait_add_edges.push(edge_obj);

    } else {
      my_nodes.get(parent_name).edges.push(edge_obj);
    }

  }
});

console.log(wait_add_edges)

wait_add_edges.forEach(element => {
  if (my_edges.has(element.source + '->' + element.target)) {
    my_edges.get(element.source + '->' + element.target).info_array = my_edges.get(element.source + '->' + element.target).info_array.concat(element.info_array);
  } else {
    my_edges.set(element.source + '->' + element.target, element);
  }
})

//算一遍每个边的数量和复杂度

var edge_num_max = 0;
var edge_complexity_log_sum_max = 0;
my_edges.forEach((value, key) => {
  var edge_num = value.info_array.length;
  var edge_complexity_log_sum = 0; // 初始化对数和
  value.info_array.forEach(element => {
    var info = element.info;
    if (info === null || info.dim === undefined || info.dim.length === 0 || info.dim.size === undefined) {
      edge_complexity_log_sum += Math.log(1 + 1); // 对每个常数项取对数
    } else {
      var log_com = 0;
      info.dim.forEach(e => {
        log_com += Math.log(e.size); // 对每个维度的大小取对数并累加
      })
      edge_complexity_log_sum += Math.log(1 + Math.exp(log_com)); // 累加对数
    }
  })
  value.edge_num = edge_num;
  value.edge_complexity_log_sum = edge_complexity_log_sum;
  if (edge_num > edge_num_max) {
    edge_num_max = edge_num;
  }
  if (edge_complexity_log_sum > edge_complexity_log_sum_max) {
    edge_complexity_log_sum_max = edge_complexity_log_sum;
  }
});


console.log(root)

var selectedNode = ref(root);

//根据节点的类型返回节点的大小
function get_node_size(node) {
  var width = 0;
  var height = 0;
  if (node.raw_info['node_type'] === 0) {
    width = 80;
    height = 80;
  } else if (node.raw_info['node_type'] === 1) {
    width = 100;
    height = 50;
  } else {
    const all_children_num = node.all_children_num;
    const max_height = 200;
    const min_height = 50;
    height = min_height + (max_height - min_height) * (all_children_num / max_children);
    width = 150;
  }
  return [width, height];
}

// console.log(root)

//计算每个图的dagre布局
function compute_node(node) {
  if (node.children.length === 0 || !node.expanded) {
    var [nodeWidth1, nodeHeight1] = get_node_size(node);
    node.width = nodeWidth1;
    node.height = nodeHeight1;
    return;
  }
  //递归更新所有子节点的信息
  node.children.forEach(childNode => {
    if (childNode.expanded) {
      compute_node(childNode);//如果这个节点是展开的，那么就递归更新这个节点的信息
    } else { //如果这个节点是折叠的，那么就直接用固定的大小赋值

      var [nodeWidth1, nodeHeight1] = get_node_size(childNode);
      childNode.width = nodeWidth1;
      childNode.height = nodeHeight1;
    }
  });
  //至此, 已经算出所有子节点的大小信息，使用dagre进行布局
  const g = new dagre.graphlib.Graph();
  g.setGraph({
    rankdir: 'BT',
    ranker: 'tight-tree',
    nodesep: 50,
    ranksep: 70,
  });
  g.setDefaultEdgeLabel(() => ({}));
  // 方向

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

//获取节点大小的函数
function getNodeShape(node) {
  if (node.raw_info.node_type === 0) {
    return 'circle';
  } else if (node.raw_info.node_type === 1) {
    return 'ellipse';
  } else {
    return 'rect';
  }
}

//根据节点的信息绘制子图
function draw_root(root, svgGroup) {
  if (root.children.length === 0 || !root.expanded) {
    return;
  }
  //先画这个节点的图
  var g = root.dagreGraph;

  g.edges().forEach(edge => {
    const my_edge_name = edge.v + '->' + edge.w;

    const my_edge = my_edges.get(my_edge_name);
    //根据edge_num确定边的粗细 根据edge_complexity_log_sum确定边的颜色
    const edge_num = my_edge.edge_num;
    const edge_complexity_log_sum = my_edge.edge_complexity_log_sum;
    const edge_width = 1.5 + 30 * (edge_num / edge_num_max);
    const scaleGray = d3.scaleLinear()
    .domain([0, edge_complexity_log_sum_max])
    .range(["#D3D3D3", "#696969"]); // 浅灰到深灰

const edge_color = scaleGray(edge_complexity_log_sum);

    const points = g.edge(edge).points;
    const lineGenerator = d3.line()
      .x(d => d.x + root.x)
      .y(d => d.y + root.y)
      .curve(d3.curveBasis); // 使用曲线生成器

    // 动态设置线条和箭头的颜色和粗细
    // const strokeColor = "#333";
    // const strokeWidth = 2;

    // 定义箭头标记（每次都可以更新箭头样式）
    svgGroup.append("defs")
      .append("marker")
      .attr("id", `arrowhead-${edge.v}-${edge.w}`)
      .attr("viewBox", "0 0 10 10")
      .attr("refX", 7) // 调整refX使箭头尖端与线条终点对齐
      .attr("refY", 5)
      .attr("markerWidth", 6)
      .attr("markerHeight", 6)
      .attr("orient", "auto")
      .append("path")
      .attr("d", "M 0 0 L 8 5 L 0 10 Q 4 5 0 0") // 修改后的路径，箭头角向后弯曲
      .attr("fill", edge_color);

    svgGroup.append("path")
      .attr("d", lineGenerator(points))
      .attr("class", "edgePath")
      .style("stroke", edge_color)
      .style("stroke-width", edge_width)
      .style("fill", "none")
      .attr("marker-end", `url(#arrowhead-${edge.v}-${edge.w})`); // 使用动态生成的箭头
  });




  function truncateString(str, length) {
    // 判断字符串的长度是否大于指定的长度
    if (str.length > length) {
      // 截取指定长度的字符串并加上省略号
      return str.substring(0, length) + '..';
    } else {
      // 返回原始字符串
      return str;
    }
  }

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

    //根据点的形状画出节点
    if (shape === 'circle') {
      nodeGroup.append("circle")
        .attr("cx", n.width / 2)
        .attr("cy", n.height / 2)
        .attr("r", Math.min(n.width, n.height) / 2)
        .attr("stroke", "#333")
        .attr("fill", "#fff")
        .style("cursor", "pointer")
        .style("stroke-width", 1)
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

      var name = ""
      if (node_obj.raw_info['op'] !== 'Const_merge') {
        name = node_obj.label.split('/').pop();
      } else {
        name = node_obj.const_children.length + " consts";
      }
      nodeGroup.append("text")
        .attr("x", n.width / 2)
        .attr("y", n.height / 2)
        .attr("dy", ".35em")
        .attr("text-anchor", "middle")
        .text(name).on("dblclick", function (event) {
          console.log("dbclick");
          if (node_obj.children.length === 0) {
            return;
          }
          node_obj.expanded = !node_obj.expanded;
          compute_node(root);
          svgGroup.selectAll("*").remove();
          draw_root(root, svgGroup);
        });

    } else if (shape === 'ellipse') {
      nodeGroup.append("ellipse")
        .attr("cx", n.width / 2)
        .attr("cy", n.height / 2)
        .attr("rx", n.width / 2)
        .attr("ry", n.height / 2)
        .attr("stroke", "#333")
        .attr("fill", "#fff")
        .style("cursor", "pointer")
        .style("stroke-width", 1)
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
          d3.select(this).attr("fill", "#f5f5f5");

        })
        .on("mouseout", function (event) {
          d3.select(this).attr("fill", "#fff");
        });

      nodeGroup.append("text")
        .attr("x", n.width / 2)
        .attr("y", n.height / 2)
        .attr("dy", ".35em")
        .attr("text-anchor", "middle")
        .text(truncateString(nodeLabel.split('/').pop(), 8)).on("dblclick", function (event) {
          console.log("dbclick");
          if (node_obj.children.length === 0) {
            return;
          }
          node_obj.expanded = !node_obj.expanded;
          compute_node(root);
          svgGroup.selectAll("*").remove();
          draw_root(root, svgGroup);
        });

    } else {
      nodeGroup.append("rect")
        .attr("width", n.width)
        .attr("height", n.height)
        .attr("rx", 10)
        .attr("ry", 10)
        .attr("stroke", "#333")
        .attr("fill", "#fff")
        .style("cursor", "pointer")
        .style("stroke-width", 2)
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

      nodeGroup.append("text")
        .attr("x", n.width / 2)
        .attr("y", n.height / 2)
        .attr("dy", ".35em")
        .attr("text-anchor", "middle")
        .style("font-size", "17px")
        .style("font-weight", "bold")
        .text(truncateString(nodeLabel.split('/').pop(), 12)).on("dblclick", function (event) {
          console.log("dbclick");
          if (node_obj.children.length === 0) {
            return;
          }
          node_obj.expanded = !node_obj.expanded;
          compute_node(root);
          svgGroup.selectAll("*").remove();
          draw_root(root, svgGroup);
        });

    }

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
    const node = d3.select(`#${id}`);
    const shape = getNodeShape(newVal);
    if (shape === 'circle') {
      node.select("circle").attr("stroke", "#ff0000").attr("fill", "#f5f5f5").style("stroke-width", 3);
    } else if (shape === 'ellipse') {
      node.select("ellipse").attr("stroke", "#ff0000").attr("fill", "#f5f5f5").style("stroke-width", 3);
    } else {
      node.select("rect").attr("stroke", "#ff0000").attr("fill", "#f5f5f5").style("stroke-width", 4);
    }
  }
  //更新旧选中节点的样式
  if (oldVal.raw_info['name']) {
    const old_id = oldVal.raw_info['name'].toString().replace(/\//g, "-");
    const old_node = d3.select(`#${old_id}`);
    const shape = getNodeShape(oldVal);
    if (shape === 'circle') {
      old_node.select("circle").attr("stroke", "#333").attr("fill", "#fff").style("stroke-width", 1);
    } else if (shape === 'ellipse') {
      old_node.select("ellipse").attr("stroke", "#333").attr("fill", "#fff").style("stroke-width", 1);
    } else {
      old_node.select("rect").attr("stroke", "#333").attr("fill", "#fff").style("stroke-width", 2);
    }
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
  width: 300px;
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