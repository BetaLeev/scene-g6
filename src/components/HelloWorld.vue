<template>

  <div class="top">
    <div class="thumbnail-panel">
      <div v-for="(item, index) in savedGraphs" :key="index" class="thumbnail-item" @click="loadGraph(item.data)">
        <img :src="item.thumbnail" :alt="item.name" />
        <span>{{ item.name }}</span>
      </div>
    </div>
  </div>

  <div class="graph-container">
    <!-- 左侧组件库 -->
    <div class="component-palette">
      <h3>组件库</h3>
      <div v-for="comp in components" :key="comp.type" class="component-item" :style="{ backgroundColor: comp.color }"
        draggable="true" @dragstart="onDragStart(comp)">
        {{ comp.label }}
      </div>
    </div>

    <!-- 中间画布 -->
    <div class="canvas-area">
      <div id="container" ref="container" class="g6-container"> </div>
      <div id="graph-minimap" ref="GraphMinimap" class="minimap-test"></div>
    </div>
    <div v-if="contextMenu.visible" :style="{ left: contextMenu.x + 'px', top: contextMenu.y + 'px' }"
      class="context-menu">
      <div v-if="contextMenu.item?.getType?.() === 'combo'">
        <div class="menu-item" @click="resizeCombo(1.2)">扩大</div>
        <div class="menu-item" @click="resizeCombo(0.8)">缩小</div>
        <div class="menu-item" @click="deleteItem">删除</div>
      </div>
      <div v-else>
        <div class="menu-item" @click="deleteItem">删除</div>
      </div>
    </div>
    <!-- 右侧属性编辑区域 -->
    <div class="property-editor">
      <div v-if="selectedNode">
        <h3>属性编辑</h3>
        <div v-for="(prop, key) in Object.values(selectedNode.model.properties)" :key="key" class="form-item">
          <label>{{ prop.label }}</label>
          <template v-if="prop.type === 'textarea'">
            <textarea v-model="selectedNode.model.properties[prop]"></textarea>
          </template>
          <template v-else-if="prop.type === 'select'">
            <select v-model="selectedNode.model.properties[prop]">
              <option v-for="(option, i) in prop.options" :key="i" :value="option">
                {{ option }}
              </option>
            </select>
          </template>
        </div>
      </div>
      <div v-else>
        <div class="form-item">
          <label>场景名称</label>
          <input v-model="sceneForm.label" />
        </div>
        <div class="form-item">
          <label>描述</label>
          <textarea v-model="sceneForm.describe" />
        </div>
      </div>
      <button @click="saveGraph" class="save-btn">保存方案</button>
      <!-- <button @click="updateNode">保存修改</button> -->
    </div>

  </div>
</template>

<script>
import { onMounted, ref, reactive } from 'vue';
import {
  Graph,
  Util,
  registerEdge,
  // registerBehavio,
  Grid,
  ToolBar,
  Tooltip,
  Minimap,
  // registerNode,
} from '@antv/g6';

export default {
  name: 'G6Graph',
  setup() {
    const contextMenu = reactive({
      visible: false,
      x: 0,
      y: 0,
      node: null
    });

    const deleteItem = () => {
      if (contextMenu.item) {
        graph.removeItem(contextMenu.item);
        contextMenu.visible = false;
        contextMenu.item = null; // 新增这行清除引用
      }
    };

    const resizeCombo = (scale) => {
      if (contextMenu.item) {
        const currentSize = contextMenu.item.getModel().size || [180, 100];
        const newSize = [currentSize[0] * scale, currentSize[1] * scale];

        graph.updateItem(contextMenu.item, {
          size: newSize,
          labelCfg: {
            offset: [0, newSize[1] / 2 + 5]
          }
        });
        graph.layout();
        contextMenu.visible = false;
      }
    };
    // 在setup()中添加响应式数据和保存方法
    let savedGraphs = reactive([]);

    const saveGraph = () => {
      const data = graph.save();
      const thumbnail = graph.toDataURL();
      savedGraphs.push({
        name: sceneForm.label,  // 使用场景名称作为方案名称
        data,
        thumbnail,
        timestamp: Date.now()
      });
    };

    // 添加方案加载方法
    // 修改loadGraph方法中的布局配置
    const loadGraph = (graphData) => {
      graph.clear();
      graph.read(graphData);

      // 使用dagre布局实现垂直排列（关键修改）
      graph.updateLayout({
        type: 'dagre',        // 明确指定使用dagre布局
        rankdir: 'TB',        // 垂直方向（Top to Bottom）
        nodesep: 20,          // 节点间距
        ranksep: 20,          // 层级间距
        controlPoints: true   // 保留连线控制点
      });

      graph.render();
      graph.zoom(0.5);
      graph.fitView();
    };

    const components = reactive([
      {
        type: 'rect',
        category: '输入',
        label: '输入组件',
        color: '#5B8FF9',
        anchorPoints: [
          [1, 0.5],
        ],
        linkPoints: {
          right: true,
          left: false,
          size: 10,
          fill: '#fff',
          stroke: '#5B8FF9'
        },
        properties: {
          description: {
            type: 'textarea',
            label: '描述'
          }
        },
      },
      {
        type: 'rect',
        category: '空间',
        label: '空间组件',
        color: '#E8684A',
        anchorPoints: [
          [0, 0.5], // 目标连接点（左侧）
          [1, 0.5], // 源连接点（右侧）
        ],
        linkPoints: {
          right: true,  // 右侧显示连接点（源）
          left: true,   // 左侧显示连接点（目标）
          size: 10,
          fill: '#fff',
          stroke: '#E8684A'
        },
        properties: {
          environment: {
            type: 'select',
            label: '选择环境',
            options: ['Windows', 'Linux', 'MacOS']
          },
        },

      },
      {
        type: 'rect',
        category: '环境',
        label: '环境组件',
        color: '#F6BD16',
        isCombo: true, // 标记为Combo类型
        properties: {
          system: {
            type: 'select',
            label: '系统',
            options: ['Windows', 'Linux', 'MacOS']
          },
          version: {
            type: 'select',
            label: '版本',
            options: {
              Windows: ['2012', '2016', '2019'],
              Linux: ['CentOS 7', 'Ubuntu 20.04'],
              MacOS: ['Ventura', 'Monterey']
            }
          },
          model: {
            type: 'select',
            label: '型号',
            options: ['标准型', '高可用型']
          }
        },
      },
      {
        type: 'rect',
        category: '输出',
        label: '输出组件',
        color: '#5AD8A6',
        anchorPoints: [
          [0, 0.5],
        ],
        linkPoints: {
          right: false,
          left: true,
          size: 10,
          fill: '#fff',
          stroke: '#5AD8A6'
        },
        properties: {
          outputType: {
            type: 'select',
            label: '输出类型',
            options: ['文件输出', '数据库输出', 'API输出']
          },
        },
      }
    ]);
    const toolbar = new ToolBar({
      position: { x: 10, y: 10 },

    });

    const tooltip = new Tooltip({
      itemTypes: ['node', 'edge'],
      getContent: (e) => {
        const outDiv = document.createElement('div');
        outDiv.style.width = 'fit-content';
        outDiv.style.padding = '0px 0px 20px 0px';
        outDiv.innerHTML = `
          <h4>${e.item.getModel().label} </h4>
        <ul>
          <li>Label: ${e.item.getModel().properties}</li>
        </ul>`;
        return outDiv;
      },
    })
    const container = ref(null);
    const grid = new Grid();
    const minimap = new Minimap({
      className: "minimap-test",
    });
    let graph = reactive(null);
    let sceneForm = reactive({
      label: '',
      describe: ''
    });

    const onDragStart = (comp) => {
      event.dataTransfer.setData('component', JSON.stringify(comp));
    };

    const selectedNode = ref(null);

    const updateNode = () => {
      if (selectedNode.value) {
        graph.updateItem(selectedNode.value.id, {
          label: selectedNode.value.label,
          x: selectedNode.value.x,
          y: selectedNode.value.y
        });
      }
    };

    onMounted(() => {
      // 修改graph初始化配置
      graph = new Graph({
        // 设置为true，启用 redo & undo 栈功能
        enabledStack: true,
        container: container.value,
        fitView: true,
        layout: {
          type: 'grid',
        },
        nodeStateStyles: {
          hover: {
            lineWidth: 1,
            fill: '#d3adf7',
          }
        },
        combo: {
          shouldUpdate: (item) => {
            const model = item.getModel();
            // 只有空间组件可以进入combo
            return model.category === '空间';
          }
        },
        modes: {
          default: [
            {
              type: 'drag-canvas',
            },
            {
              type: 'drag-node',
            },
            {
              type: 'drag-combo',
            },
            {
              type: 'create-edge',
              trigger: 'click',
              key: 'shift',
              shouldBegin: e => {
                // 获取源节点模型
                const sourceNode = graph.findById(e.item._cfg.id);
                if (!sourceNode) return false;
                const sourceModel = sourceNode.getModel();

                // 禁止从Combo创建连线
                if (sourceModel?.isCombo) return false;

                // 获取所有边
                const edges = graph.getEdges();
                const sourceEdges = sourceNode.getEdges();
                // 如果是输入组件，检查是否已有出线
                if (sourceModel?.category === '输入') {
                  return sourceEdges.length <= 2;
                }

                if (sourceModel?.category === '空间') {
                  if (sourceEdges.length > 2) { return false }
                  return !edges.some(edge => {
                    const edgeModel = edge.getModel();
                    return edgeModel.source === e.source;
                  });
                }

                if (sourceModel?.category === '输出') {
                  return false
                }
                return true;
              },
              shouldEnd: (e, self) => {
                // 移除调试语句并获取正确目标节点
                const targetNode =
                  graph.findById(e.item._cfg.id);

                if (!targetNode) return false;
                const targetModel = targetNode.getModel();

                const sourceNode = graph.findById(self.source);
                const sourceModel = sourceNode.getModel();

                // 禁止输入组件连接输出组件
                if (sourceModel?.category === '输入' && targetModel?.category === '输出') {
                  return false;
                }

                // 原有禁止规则保持不变
                return !(
                  targetModel?.category === '输入' ||
                  targetModel?.category === '环境' ||
                  targetModel?.isCombo
                );
              },
              edgeConfig: {
                type: 'cubic-vertical',
                sourceAnchor: 1,  // 指定源锚点索引
                targetAnchor: 0,  // 指定目标锚点索引
                style: {
                  stroke: '#f00',
                  lineWidth: 2,
                  endArrow: {
                    path: 'M 0,0 L 8,4 L 8,-4 Z',
                    fill: '#ff0',
                    d: 3,
                  }
                },

              },
            }
          ]
        },
        plugins: [
          toolbar, tooltip, grid, minimap
        ],
      });

      // 修改拖放事件处理
      container.value.addEventListener('drop', (e) => {
        e.preventDefault();
        const comp = JSON.parse(e.dataTransfer.getData('component'));
        const point = graph.getPointByClient(e.clientX, e.clientY);

        if (comp.isCombo) {
          // 添加Combo节点
          graph.addItem('combo', {
            id: `combo-${Date.now()}`,
            label: comp.label,
            labelCfg: {
              position: 'bottom',
              style: {
                fill: '#f'
              }
            },
            category: comp.category,
            type: 'rect',
            x: point.x,
            y: point.y,
            style: {
              fill: comp.color,
              stroke: comp.color,
              opacity: 0.6
            },
            size: [180, 100],
            isCombo: true,
            properties: comp.properties,
            shouldAccept: (child) => {
              const childModel = child.getModel();
              return childModel.category === '空间';
            }
            // 明确标记环境组件类型
          });
        } else {
          // 添加普通节点
          // 在添加节点的代码部分修改样式
          graph.addItem('node', {
            id: `node-${Date.now()}`,
            label: comp.label,
            labelCfg: {
              style: {
                fill: '#fff'
              }
            },
            category: comp.category,
            type: comp.type,
            x: point.x,
            y: point.y,
            singleEdge: true,
            style: {
              fill: comp.color,
              stroke: comp.color,
              // 添加label样式
              labelCfg: {
                style: {
                  fill: '#fff' // 设置文字颜色为白色
                }
              }
            },
            // 限制只能从连接桩连接
            anchorPoints: comp.anchorPoints,
            linkPoints: comp.linkPoints,
            isCombo: false,
            properties: comp.properties,
          });
        }
      });

      graph.on('aftercreateedge', (e) => {
        console.log(e, 'e')
        const edges = graph.save().edges;
      });

      graph.on('node:click', (evt, self) => {
        const node = evt.item;
        const model = node.getModel();
        console.log(node, 'node')
        console.log(self, 'self')
        selectedNode.value = { node, model };
        // 激活该节点的 hover 状态
        // graph.setItemState(node, 'hover', true);
        // graph.setItemState(node, 'active', true);
        // graph.setItemState(node, 'running', true);
      });

      // 监听鼠标进入节点事件
      graph.on('node:mouseenter', (evt) => {
        const node = evt.item;
        // 显示连接桩（新增逻辑）
        console.log(node, 'node')
        graph.setItemState(node, 'hover', true);

        // graph.updateItem(node, {
        //     anchorPoints: []
        // });
      });

      // 监听鼠标离开节点事件
      graph.on('node:mouseleave', (evt) => {
        const node = evt.item;
        // 隐藏连接桩（新增逻辑）
        // graph.updateItem(node, {
        //     linkPoints: {
        //         ...node.getModel().linkPoints,
        //         visible: false // 设为不可见
        //     }
        // });
        // 关闭该节点的 hover 状态
        graph.setItemState(node, 'hover', false);
        // graph.setItemState(node, 'active', false);
        // graph.setItemState(node, 'running', false);
      });

      graph.on('combo:mouseenter', (evt) => {
        const { item } = evt;
        // graph.setItemState(item, 'active', true);
      });

      graph.on('combo:mouseleave', (evt) => {
        const { item } = evt;
        // graph.setItemState(item, 'active', false);
      });
      graph.on('combo:click', (evt) => {
        const { item } = evt;
        const currentSize = item.getModel().size || [180, 100];
        const newSize = [currentSize[0] * 1.2, currentSize[1] * 1.2];

        // 限制最大缩放尺寸
        if (newSize[0] < 300) {
          graph.updateItem(item, {
            size: newSize,
            labelCfg: {
              position: 'bottom',
              offset: [0, newSize[1] / 2 + 5]
            }
          });
          graph.layout();
        }
      });

      graph.on('canvas:click', (evt) => {
        selectedNode.value = null; // 清空选中节点状态，触发右侧显示场景表单
        graph.getCombos().forEach((combo) => {
          graph.clearItemStates(combo);
        });
        contextMenu.visible = false; // 保持原有隐藏菜单逻辑
      });

      // 添加节点右键点击事件
      graph.on('node:contextmenu', (evt) => {
        evt.preventDefault();
        const point = graph.getPointByClient(evt.clientX, evt.clientY);
        contextMenu.node = evt.item;
        contextMenu.x = point.x + 260;
        contextMenu.y = point.y + 160;
        contextMenu.visible = true;
      });

      graph.on('combo:contextmenu', (evt) => {
        evt.preventDefault();
        const point = graph.getPointByClient(evt.clientX, evt.clientY);
        contextMenu.item = evt.item;
        contextMenu.x = point.x + 310;
        contextMenu.y = point.y + 160;
        contextMenu.visible = true;
      });

      // 新增连线右键事件
      graph.on('edge:contextmenu', (evt) => {
        evt.preventDefault();
        const point = graph.getPointByClient(evt.clientX, evt.clientY);
        contextMenu.item = evt.item;
        contextMenu.x = point.x + 260;
        contextMenu.y = point.y + 160;
        contextMenu.visible = true;
      });

      // 点击其他地方隐藏菜单
      graph.on('canvas:click', () => {
        contextMenu.visible = false;
      });

      graph.render();
    });

    return {
      resizeCombo,
      sceneForm,
      loadGraph,
      saveGraph,
      contextMenu,
      deleteItem,
      savedGraphs,
      components,
      onDragStart,
      container,
      selectedNode,
      updateNode
    };
  }
}
</script>

<style>
.top {
  height: 120px;
  margin-bottom: 20px;
}

.thumbnail-item {
  margin: 20px;
  width: 120px;
  height: 100px;
  cursor: pointer;
  border: 1px solid #ddd;
  padding: 4px;
}

.thumbnail-item img {
  width: 100%;
  height: 80px;
  object-fit: cover;
}

.thumbnail-item span {
  display: block;
  text-align: center;
  font-size: 12px;
}
</style>

<style>
.minimap-test {
  position: absolute;
  right: 12px;
  /* 直接贴边 */
  bottom: 30px;
  /* 直接贴边 */

  border: 1px solid #e2e2e2;
  z-index: 999;
  border-radius: 4px 0 0 0;
  /* 可选：左上角添加圆角 */
  box-shadow: -2px -2px 5px rgba(0, 0, 0, 0.1);
  /* 可选：添加阴影突出显示 */
}
</style>
<style scoped>
.context-menu {
  position: absolute;
  background: white;
  border: 1px solid #ddd;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
  z-index: 1000;
}

.menu-item {
  padding: 8px 16px;
  cursor: pointer;
}

.menu-item:hover {
  background: #f5f5f5;
}

.g6-component-tooltip {
  background-color: rgba(255, 255, 255, 0.8);
  padding: 0px 10px 24px 10px;
  box-shadow: rgb(174, 174, 174) 0px 0px 10px;
}

.graph-container {
  display: flex;
  height: 90vh;
  width: 100%;
  border: 1px solid #e8e8e8;
}

.component-palette {
  width: 200px;
  /* 修改为200px */
  padding: 15px;
  border-right: 1px solid #e8e8e8;
  background: #f5f5f5;
  overflow-y: auto;
}

.canvas-area {
  flex: 1;
  position: relative;
  display: flex;
  flex-direction: column;
  min-width: 0;
  /* 添加此属性防止flex布局溢出 */
}

.property-editor {
  width: 200px;
  /* 修改为200px */
  padding: 15px;
  border-left: 1px solid #e8e8e8;
  background: #f5f5f5;
  overflow-y: auto;
}

.component-item {
  margin: 10px 0;
  padding: 10px;
  border-radius: 4px;
  cursor: move;
  color: white;
  text-align: center;
}

.g6-container {
  flex: 1;
  /* height: 100%; */
  position: relative;
}

.canvas-area {
  flex: 1;
  position: relative;
  /* 确保绝对定位基于此容器 */
  display: flex;
  flex-direction: column;
  min-width: 0;
  /* 添加此属性防止flex布局溢出 */
}

.property-editor {
  width: 240px;
  /* 适当加宽提升内容显示空间 */
  padding: 20px;
  /* 增加内边距 */
  border-left: 1px solid #e8e8e8;
  background: #f8f9fa;
  /* 更柔和的背景色 */
  overflow-y: auto;
}

.property-editor h3 {
  margin: 0 0 20px 0;
  /* 调整标题间距 */
  color: #2c3e50;
  font-size: 16px;
  font-weight: 500;
}

.form-item {
  margin-bottom: 18px;
  /* 增大表单项间距 */
}

.form-item label {
  display: block;
  text-align: left;
  margin-bottom: 8px;
  /* 标签与输入框间距 */
  color: #495057;
  font-size: 14px;
  font-weight: 500;
}

/* 统一输入框样式 */
.form-item input,
.form-item textarea,
.form-item select {
  width: 100%;
  padding: 8px 12px;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  box-sizing: border-box;
  font-size: 14px;
  background: #fff;
}

.form-item textarea {
  height: 90px;
  /* 增大文本域高度 */
  resize: vertical;
  /* 允许垂直调整大小 */
}

/* 优化按钮组样式 */
.property-editor button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: opacity 0.2s;
}

.save-btn {
  background: #5B8FF9;
  color: white;
  margin-right: 10px;
  /* 按钮间间距 */
}

.update-btn {
  background: #5AD8A6;
  /* 与输出组件颜色呼应 */
  color: white;
}

.property-editor button:hover {
  opacity: 0.9;
  /* 悬停淡化效果 */
}
</style>
