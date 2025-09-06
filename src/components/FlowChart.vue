<template>
  <div class="flow-chart-container">
    <div class="left-panel">
      <h3>流程图定义</h3>
      
      <div class="input-section">
        <h4>节点定义</h4>
        <div class="input-rules">
          <p class="rule-tip">支持编号和非编号节点，按行排列</p>
          <p class="rule-example">示例：1.A  2.B  3.C  4.D  E  5.F</p>
        </div>
        <textarea
          v-model="nodeText"
          placeholder="请输入节点定义，每行一个节点\n1.A\n2.B\n3.C\n4.D\nE\n5.F"
          @input="parseTextToGraph"
          class="node-textarea"
        ></textarea>
      </div>
      
      <div class="input-section">
        <h4>连接定义</h4>
        <div class="input-rules">
          <p class="rule-tip">每行定义一条连接路径，支持多次绘制和连接注释</p>
          <p class="rule-example">示例：1->2->3  2->|注释|4->5  3->5</p>
        </div>
        <textarea
          v-model="connectionText"
          placeholder="请输入连接定义，每行一条路径\n1->2->3\n2->|注释|4->5\n3->5"
          @input="parseTextToGraph"
          class="connection-textarea"
        ></textarea>
      </div>
      
      <button class="reset-button" @click="resetToExample">重置为示例</button>
    </div>
    <div class="right-panel">
      <div class="panel-header">
        <h3>流程图展示</h3>
        <button @click="exportImage" class="export-btn" title="导出为图片">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>
            <polyline points="7,10 12,15 17,10"></polyline>
            <line x1="12" y1="15" x2="12" y2="3"></line>
          </svg>
          导出图片
        </button>
      </div>
      <div class="graph-area">
        <div id="graph-container" ref="graphContainer"></div>
        <!-- Loading 遮罩层 -->
        <div v-if="isLoading" class="loading-overlay">
          <div class="loading-spinner">
            <div class="spinner"></div>
            <p class="loading-text">正在更新流程图...</p>
          </div>
        </div>
      </div>
    </div>
    
    <!-- 自定义编辑对话框 -->
    <div v-if="showEditDialog" class="edit-dialog-overlay" @click="cancelEdit">
      <div class="edit-dialog" @click.stop>
        <h4>编辑节点文本</h4>
        <textarea 
          v-model="editingText" 
          class="edit-textarea"
          placeholder="请输入节点文本，支持多行"
          @keydown.ctrl.enter="confirmEdit"
          ref="editTextarea"
        ></textarea>
        <div class="edit-dialog-buttons">
          <button @click="confirmEdit" class="confirm-btn">确认</button>
          <button @click="cancelEdit" class="cancel-btn">取消</button>
        </div>
        <p class="edit-tip">提示：Ctrl+Enter 快速确认</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick, watch } from 'vue';
import { Graph, Shape } from '@antv/x6';
import { Export } from '@antv/x6-plugin-export';

const graphContainer = ref(null);
const nodeText = ref(`1.测试
2.面试
3.通过面试
4.拿到offer
签约
5.入职`);
const connectionText = ref(`1->2->3->|通过|4->6
3->5->6`);
const graph = ref(null);
const nodes = ref([]);
const edges = ref([]);
const isUpdatingFromGraph = ref(false);
const hasUserAdjustedLayout = ref(false);
const isLoading = ref(false);

// 编辑对话框相关变量
const showEditDialog = ref(false);
const editingText = ref('');
const editingNode = ref(null);
const editTextarea = ref(null);

// 初始化图表
onMounted(() => {
  nextTick(() => {
    initGraph();
    parseTextToGraph();
  });
});

// 防抖函数
let parseTimeout = null;

// 监听输入文本变化
watch([nodeText, connectionText], () => {
  if (!isUpdatingFromGraph.value) {
    // 清除之前的定时器
    if (parseTimeout) {
      clearTimeout(parseTimeout);
    }
    // 设置新的定时器，防抖300ms
    parseTimeout = setTimeout(() => {
      parseTextToGraph();
    }, 300);
  }
});

// 初始化图表
const initGraph = () => {
  graph.value = new Graph({
    container: graphContainer.value,
    grid: true,
    panning: {
      enabled: true,
    },
    selecting: {
      enabled: true,
      rubberband: false,
      movable: true,  // 允许移动节点
      showNodeSelectionBox: true,
    },
    mousewheel: {
      enabled: true,
      zoomAtMousePosition: true,
      modifiers: 'ctrl',
      minScale: 0.5,
      maxScale: 2,
    },
    connecting: {
      enabled: false,
    },
  });

  // 使用导出插件
  graph.value.use(new Export());

  // 添加双击编辑节点功能
  graph.value.on('node:dblclick', ({ node }) => {
    // 启用节点文本编辑
    node.attr('text/style/pointerEvents', 'auto');
    
    // 创建临时输入框进行编辑
    const currentText = node.attr('text/text');
    const cleanText = currentText.replace(/\n\n─────────\n\n/g, '\n').trim();
    
    // 使用自定义对话框进行编辑
    editingNode.value = node;
    editingText.value = cleanText;
    showEditDialog.value = true;
    
    // 等待DOM更新后聚焦到文本框
    nextTick(() => {
      if (editTextarea.value) {
        editTextarea.value.focus();
        editTextarea.value.select();
      }
    });
  });

};

// 根据节点文本获取颜色
const getNodeColor = (text) => {
  if (text.includes('开始')) return '#e6f7ff';
  if (text.includes('结束')) return '#fff2e8';
  if (text.includes('处理') || text.includes('数据')) return '#f6ffed';
  if (text.includes('展示') || text.includes('显示')) return '#fff1f0';
  return '#f5f5f5';
};

// 重置为示例
const resetToExample = () => {
  nodeText.value = `1.测试
2.面试
3.通过面试
4.拿到offer
签约
5.入职`;
  connectionText.value = `1->2->3->|通过|4->6
3->5->6`;
  hasUserAdjustedLayout.value = false;
};

// 解析文本并生成流程图
const parseTextToGraph = () => {
  if (!graph.value) return;
  
  // 启用loading状态
  isLoading.value = true;
  
  // 清除防抖定时器
  if (parseTimeout) {
    clearTimeout(parseTimeout);
    parseTimeout = null;
  }
  
  const nodeTextValue = nodeText.value.trim();
  const connectionTextValue = connectionText.value.trim();
  
  if (!nodeTextValue) {
    graph.value.clearCells();
    nodes.value = [];
    edges.value = [];
    graph.value.centerContent();
    isLoading.value = false;
    return;
  }

  // 保存现有节点位置和信息
  const existingPositions = new Map();
  const existingNodeTexts = new Map();
  const existingNodes = graph.value.getNodes();
  existingNodes.forEach(node => {
    const nodeId = node.id.replace('node-', '');
    const position = node.getPosition();
    const text = node.attr('text/text');
    existingPositions.set(nodeId, position);
    existingNodeTexts.set(nodeId, text);
  });
  
  const nodeMap = new Map();
  const connections = [];
  
  // 解析节点定义
  const nodeLines = nodeTextValue.split('\n').map(line => line.trim()).filter(line => line);
  let currentNodeId = null;
  let currentNodeText = '';
  let nodeIndex = 0;
  
  // 重置解析状态，确保没有残留
  
  nodeLines.forEach((line, index) => {
    // 检查是否有编号格式 "数字.文本"
    const numberedMatch = line.match(/^(\d+)\.(.+)/);
    if (numberedMatch) {
      // 如果之前有节点，先保存
      if (currentNodeId !== null && currentNodeText.trim()) {
        nodeMap.set(currentNodeId, { 
          id: currentNodeId, 
          text: currentNodeText.trim(), 
          type: 'process',
          index: nodeIndex
        });
        nodeIndex++;
      }
      
      // 开始新节点，完全重置状态
      [, currentNodeId, currentNodeText] = numberedMatch;
      currentNodeText = currentNodeText.trim();
    } else {
      // 非编号行，添加到当前节点的文本中（换行）
      if (currentNodeId !== null) {
        currentNodeText += '\n' + line;
      } else {
        // 如果没有前置编号节点，创建一个默认节点
        currentNodeId = String(nodeIndex + 1);
        currentNodeText = line.trim();
      }
    }
  });
  
  // 处理最后一个节点
  if (currentNodeId !== null && currentNodeText.trim()) {
    nodeMap.set(currentNodeId, { 
      id: currentNodeId, 
      text: currentNodeText.trim(), 
      type: 'process',
      index: nodeIndex
    });
  }
  
  // 解析连接定义
  if (connectionTextValue) {
    const connectionLines = connectionTextValue.split('\n').map(line => line.trim()).filter(line => line);
    connectionLines.forEach(line => {
      // 支持连接路径：1->2->3 或 2->|注释|4->5
      const parts = line.split('->');
      for (let i = 0; i < parts.length - 1; i++) {
        const source = parts[i].trim();
        const target = parts[i + 1].trim();
        
        // 检查target是否包含注释语法 |注释|
        const labelMatch = target.match(/^\|(.+?)\|(.+)$/);
        if (labelMatch) {
          // 有注释的情况：2->|注释|4
          const label = labelMatch[1].trim();
          const actualTarget = labelMatch[2].trim();
          connections.push({ source, target: actualTarget, label });
        } else {
          // 普通连接
          connections.push({ source, target });
        }
      }
    });
  }
  
  // 基于节点顺序的布局算法
  const layoutNodes = () => {
    const positions = new Map();
    
    // 如果节点有index信息，使用顺序布局
    const hasIndexInfo = Array.from(nodeMap.values()).some(node => 
      node.hasOwnProperty('index')
    );
    
    if (hasIndexInfo) {
      // 使用节点index进行垂直布局
      const xSpacing = 200;
      const ySpacing = 120;
      const startX = 100;
      const startY = 100;
      
      nodeMap.forEach((nodeData, nodeId) => {
        const index = nodeData.index || 0;
        const x = startX;
        const y = startY + index * ySpacing;
        positions.set(nodeId, { x, y });
      });
    } else {
      // 回退到基于连接关系的布局
      const visited = new Set();
      const levels = new Map();
      
      // 构建连接关系图
      const adjacencyList = new Map();
      const inDegree = new Map();
      
      // 初始化
      nodeMap.forEach((_, nodeId) => {
        adjacencyList.set(nodeId, []);
        inDegree.set(nodeId, 0);
      });
      
      // 构建邻接表和入度
      connections.forEach(conn => {
        if (nodeMap.has(conn.source) && nodeMap.has(conn.target)) {
          adjacencyList.get(conn.source).push(conn.target);
          inDegree.set(conn.target, inDegree.get(conn.target) + 1);
        }
      });
      
      // 拓扑排序分层
      const queue = [];
      inDegree.forEach((degree, nodeId) => {
        if (degree === 0) {
          queue.push(nodeId);
          levels.set(nodeId, 0);
        }
      });
      
      while (queue.length > 0) {
        const current = queue.shift();
        const currentLevel = levels.get(current);
        
        adjacencyList.get(current).forEach(neighbor => {
          const newDegree = inDegree.get(neighbor) - 1;
          inDegree.set(neighbor, newDegree);
          
          if (newDegree === 0) {
            queue.push(neighbor);
            levels.set(neighbor, currentLevel + 1);
          }
        });
      }
      
      // 如果还有未分层的节点（可能存在环），给它们分配层级
      nodeMap.forEach((_, nodeId) => {
        if (!levels.has(nodeId)) {
          levels.set(nodeId, 0);
        }
      });
      
      // 按层级布局
      const levelGroups = new Map();
      levels.forEach((level, nodeId) => {
        if (!levelGroups.has(level)) {
          levelGroups.set(level, []);
        }
        levelGroups.get(level).push(nodeId);
      });
      
      const xSpacing = 200;
      const ySpacing = 120;
      const startX = 100;
      const startY = 100;
      
      levelGroups.forEach((nodeIds, level) => {
        nodeIds.forEach((nodeId, index) => {
          const x = startX + level * xSpacing;
          const y = startY + index * ySpacing;
          positions.set(nodeId, { x, y });
        });
      });
    }
    
    return positions;
  };
  
  const nodePositions = layoutNodes();
  
  // 检查是否需要完全重建图表
  const currentNodeIds = new Set(nodeMap.keys());
  const existingNodeIds = new Set(existingPositions.keys());
  
  // 获取现有连接信息
  const existingConnections = [];
  const existingEdges = graph.value.getEdges();
  existingEdges.forEach(edge => {
    const sourceNode = edge.getSourceNode();
    const targetNode = edge.getTargetNode();
    if (sourceNode && targetNode) {
      const sourceId = sourceNode.id.replace('node-', '');
      const targetId = targetNode.id.replace('node-', '');
      existingConnections.push({ source: sourceId, target: targetId });
    }
  });
  
  // 检查连接是否发生变化
  const connectionsChanged = existingConnections.length !== connections.length ||
    !existingConnections.every(existing => 
      connections.some(current => 
        current.source === existing.source && current.target === existing.target
      )
    ) ||
    !connections.every(current => 
      existingConnections.some(existing => 
        existing.source === current.source && existing.target === current.target
      )
    );
  
  const needsRebuild = currentNodeIds.size !== existingNodeIds.size || 
    ![...currentNodeIds].every(id => existingNodeIds.has(id)) ||
    ![...existingNodeIds].every(id => currentNodeIds.has(id)) ||
    connectionsChanged;
  
  if (needsRebuild) {
    // 需要重建：彻底清理图表和DOM
    graph.value.clearCells();
    graph.value.dispose();
    
    // 清理容器DOM
    if (graphContainer.value) {
      graphContainer.value.innerHTML = '';
    }
    
    // 重新初始化图表
    initGraph();
    
    nodes.value = [];
    edges.value = [];
  } else {
    // 仅更新内容：检查文本变化并更新
    let hasTextChanges = false;
    nodeMap.forEach((nodeData, nodeId) => {
      const existingText = existingNodeTexts.get(nodeId);
      const newText = nodeData.text.trim().replace(/\n/g, '\n\n─────────\n\n');
      if (existingText !== newText) {
        hasTextChanges = true;
        const node = graph.value.getNodes().find(n => n.id === `node-${nodeId}`);
        if (node) {
          node.attr('text/text', newText);
        }
      }
    });
    
    if (!hasTextChanges) {
      isLoading.value = false;
      return; // 没有变化，直接返回
    }
  }
  
  // 清理不存在节点的位置缓存
  const validNodeIds = new Set(nodeMap.keys());
  const cleanedPositions = new Map();
  existingPositions.forEach((position, nodeId) => {
    if (validNodeIds.has(nodeId)) {
      cleanedPositions.set(nodeId, position);
    }
  });
  
  // 只在重建模式下创建节点
  if (needsRebuild) {
    // 创建节点
    nodeMap.forEach((nodeData, nodeId) => {
    // 优先使用保存的位置，如果没有则使用计算的位置
    const position = cleanedPositions.get(nodeId) || nodePositions.get(nodeId) || { x: 100, y: 100 };
    
    const node = graph.value.addNode({
      id: `node-${nodeId}`,
      x: position.x,
      y: position.y,
      width: 160,
      height: 100,
      attrs: {
        body: {
          fill: getNodeColor(nodeData.text),
          stroke: '#d9d9d9',
          strokeWidth: 1,
          rx: 8,
          ry: 8,
        },
        text: {
          text: nodeData.text.trim().replace(/\n/g, '\n\n─────────\n\n'),
          fill: '#333',
          fontSize: 12,
          textWrap: {
            width: 150,
            height: 90,
            breakWord: true,
            ellipsis: true,
          },
          style: {
            lineHeight: '1.8em',
          },
        },
      },
      ports: {
        groups: {
          top: {
            position: 'top',
            attrs: {
              circle: {
                r: 4,
                magnet: false,  // 禁用磁性连接
                stroke: '#5F95FF',
                strokeWidth: 1,
                fill: '#fff',
              },
            },
          },
          bottom: {
            position: 'bottom',
            attrs: {
              circle: {
                r: 4,
                magnet: false,  // 禁用磁性连接
                stroke: '#5F95FF',
                strokeWidth: 1,
                fill: '#fff',
              },
            },
          },
        },
        items: [
          { id: 'port-top', group: 'top' },
          { id: 'port-bottom', group: 'bottom' },
        ],
      },
    });
    
    nodes.value.push(node);
  });
  
  // 创建连接
  connections.forEach(conn => {
    const sourceNode = nodes.value.find(n => n.id === `node-${conn.source}`);
    const targetNode = nodes.value.find(n => n.id === `node-${conn.target}`);
    
    if (sourceNode && targetNode) {
      // 使用端口进行连接，但端口已设置为不可交互
      const sourcePort = 'port-bottom';
      const targetPort = 'port-top';
      
      // 构建边的属性
      const edgeAttrs = {
        line: {
          stroke: '#5F95FF',
          strokeWidth: 2,
          targetMarker: {
            name: 'classic',
            size: 8,
          },
        },
      };
      
      // 如果有标签，添加标签样式
      if (conn.label) {
        edgeAttrs.text = {
          text: conn.label,
          fill: '#333',
          fontSize: 12,
          textAnchor: 'middle',
          textVerticalAnchor: 'middle',
          fontWeight: 'bold',
        };
      }
      
      const edge = graph.value.addEdge({
        source: { cell: sourceNode.id, port: sourcePort },
        target: { cell: targetNode.id, port: targetPort },
        attrs: edgeAttrs,
        router: { name: 'manhattan' },
        connector: { name: 'rounded', args: { radius: 8 } },
        // 如果有标签，设置标签位置
        labels: conn.label ? [{
          attrs: {
            text: {
              text: conn.label,
              fill: '#333',
              fontSize: 12,
              fontWeight: 'bold',
            },
            rect: {
              fill: '#ffffff',
              stroke: '#5F95FF',
              strokeWidth: 1,
              rx: 4,
              ry: 4,
            },
          },
          position: {
            distance: 0.5, // 标签位置在边的中点
          },
        }] : [],
      });
      
      edges.value.push(edge);
    }
  });
  
    if (!hasUserAdjustedLayout.value) {
      setTimeout(() => {
        graph.value.centerContent();
        // 图表更新完成，关闭loading状态
        isLoading.value = false;
      }, 100);
    } else {
      // 如果不需要居中，直接关闭loading
      isLoading.value = false;
    }
  } else {
    // 轻量更新模式完成，关闭loading
    isLoading.value = false;
  } // 结束重建模式的 if 语句
};

// 只更新节点文本，不影响连接定义
const updateNodeTextOnly = () => {
  if (!graph.value) return;
  
  // 获取当前图中的所有节点
  const currentNodes = graph.value.getNodes();
  
  if (currentNodes.length === 0) {
    isUpdatingFromGraph.value = true;
    nodeText.value = '';
    isUpdatingFromGraph.value = false;
    return;
  }
  
  // 只更新节点文本，保持连接文本不变
  isUpdatingFromGraph.value = true;
  const nodeTexts = currentNodes.map((node, index) => {
    const text = node.attr('text/text');
    // 清理格式化的分隔线，与双击编辑时保持一致
    const cleanText = text.replace(/\n\n─────────\n\n/g, '\n').trim();
    return `${index + 1}.${cleanText}`;
  });
  nodeText.value = nodeTexts.join('\n');
  isUpdatingFromGraph.value = false;
};

// 从图表更新输入文本
const updateInputTextFromGraph = () => {
  if (!graph.value) return;
  
  // 获取当前图中的所有节点和边
  const currentNodes = graph.value.getNodes();
  const currentEdges = graph.value.getEdges();
  
  if (currentNodes.length === 0) {
    isUpdatingFromGraph.value = true;
    nodeText.value = '';
    connectionText.value = '';
    isUpdatingFromGraph.value = false;
    return;
  }
  
  // 更新节点文本
  isUpdatingFromGraph.value = true;
  const nodeTexts = currentNodes.map((node, index) => {
    const text = node.attr('text/text');
    return `${index + 1}.${text}`;
  });
  nodeText.value = nodeTexts.join('\n');
  
  // 如果没有边，清空连接文本
  if (currentEdges.length === 0) {
    connectionText.value = '';
    isUpdatingFromGraph.value = false;
    return;
  }
  
  // 构建连接关系图
  const nodeMap = new Map();
  const inDegree = new Map();
  const outEdges = new Map();
  
  // 初始化节点
  currentNodes.forEach(node => {
    const nodeId = node.id;
    nodeMap.set(nodeId, node);
    inDegree.set(nodeId, 0);
    outEdges.set(nodeId, []);
  });
  
  // 构建边的关系
  currentEdges.forEach(edge => {
    const sourceId = edge.getSourceNode()?.id;
    const targetId = edge.getTargetNode()?.id;
    
    if (sourceId && targetId) {
      outEdges.get(sourceId).push(targetId);
      inDegree.set(targetId, inDegree.get(targetId) + 1);
    }
  });
  
  // 找到起始节点（入度为0的节点）
  const startNodes = [];
  inDegree.forEach((degree, nodeId) => {
    if (degree === 0) {
      startNodes.push(nodeId);
    }
  });
  
  // 构建路径
  const result = [];
  
  const dfs = (nodeId, path, visited) => {
    // 检查是否为开头节点（入度为0）或结尾节点（出度为0）
    const isStartNode = inDegree.get(nodeId) === 0;
    const isEndNode = (outEdges.get(nodeId) || []).length === 0;
    
    // 如果不是开头或结尾节点，且已访问过，则返回（防止环路）
    if (!isStartNode && !isEndNode && visited.has(nodeId)) return;
    
    const newVisited = new Set(visited);
    // 只有非开头非结尾节点才加入visited集合
    if (!isStartNode && !isEndNode) {
      newVisited.add(nodeId);
    }
    const newPath = [...path, nodeMap.get(nodeId).attr('text/text')];
    
    const neighbors = outEdges.get(nodeId) || [];
    if (neighbors.length === 0) {
      // 叶子节点，记录路径
      result.push(newPath);
    } else {
      neighbors.forEach(neighborId => {
        dfs(neighborId, newPath, newVisited);
      });
    }
  };
  
  // 从每个起始节点开始遍历
  if (startNodes.length > 0) {
    startNodes.forEach(startId => {
      dfs(startId, [], new Set());
    });
  } else {
    // 如果没有起始节点（可能有环），从第一个节点开始
    dfs(currentNodes[0].id, [], new Set());
  }
  
  // 选择最长的路径作为结果
  let longestPath = [];
  result.forEach(path => {
    if (path.length > longestPath.length) {
      longestPath = path;
    }
  });
  
  // 构建连接文本
  const connectionPaths = [];
  const nodeIdMap = new Map();
  
  // 创建节点ID映射
  currentNodes.forEach((node, index) => {
    nodeIdMap.set(node.id, index + 1);
  });
  
  // 构建连接关系
  const edgeMap = new Map();
  currentEdges.forEach(edge => {
    const sourceId = nodeIdMap.get(edge.getSourceNode()?.id);
    const targetId = nodeIdMap.get(edge.getTargetNode()?.id);
    if (sourceId && targetId) {
      if (!edgeMap.has(sourceId)) {
        edgeMap.set(sourceId, []);
      }
      edgeMap.get(sourceId).push(targetId);
    }
  });
  
  // 生成连接路径
  edgeMap.forEach((targets, source) => {
    targets.forEach(target => {
      connectionPaths.push(`${source}->${target}`);
    });
  });
  
  connectionText.value = connectionPaths.join('\n');
  isUpdatingFromGraph.value = false;
 };

// 确认编辑
const confirmEdit = () => {
  if (editingNode.value && editingText.value.trim() !== '') {
    // 更新节点文本
    const formattedText = editingText.value.trim().replace(/\n/g, '\n\n─────────\n\n');
    editingNode.value.attr('text/text', formattedText);
    
    // 只更新节点文本，不影响连接定义
    updateNodeTextOnly();
  }
  
  // 关闭对话框
  showEditDialog.value = false;
  editingNode.value = null;
  editingText.value = '';
};

// 取消编辑
  const cancelEdit = () => {
    showEditDialog.value = false;
    editingNode.value = null;
    editingText.value = '';
  };

// 导出图片
const exportImage = () => {
  if (!graph.value) return;
  
  // 使用导出插件导出为PNG格式
  graph.value.exportPNG(`流程图_${new Date().toISOString().slice(0, 19).replace(/:/g, '-')}.png`, {
    backgroundColor: '#ffffff',
    padding: 20,
    quality: 1
  });
 };


</script>

<style scoped>
.flow-chart-container {
  display: flex;
  height: 100vh;
  gap: 0;
  margin: 0;
  padding: 0;
  font-family: Arial, sans-serif;
  box-sizing: border-box;
}

.left-panel {
  width: 350px;
  min-width: 350px;
  display: flex;
  flex-direction: column;
  border-right: 1px solid #d9d9d9;
  padding: 20px;
  background-color: #f9f9f9;
  box-sizing: border-box;
  overflow-y: auto;
}

.right-panel {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 20px;
  box-sizing: border-box;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.export-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 12px;
  background-color: #5F95FF;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 13px;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 2px 4px rgba(95, 149, 255, 0.2);
}

.export-btn:hover {
  background-color: #4080ff;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(95, 149, 255, 0.3);
}

.export-btn:active {
  transform: translateY(0);
  box-shadow: 0 2px 4px rgba(95, 149, 255, 0.2);
}

.export-btn svg {
  flex-shrink: 0;
}

h3 {
  margin: 0 0 20px 0;
  color: #333;
  font-size: 16px;
  font-weight: 600;
}

h4 {
  margin: 0 0 10px 0;
  color: #333;
  font-size: 14px;
  font-weight: 600;
}

.input-section {
  margin-bottom: 20px;
}

.input-section:last-of-type {
  margin-bottom: 15px;
}

textarea {
  width: 100%;
  padding: 12px;
  border: 1px solid #d9d9d9;
  border-radius: 6px;
  resize: vertical;
  font-size: 13px;
  line-height: 1.4;
  font-family: 'Courier New', monospace;
  box-sizing: border-box;
}

.node-textarea {
  height: 120px;
  min-height: 80px;
}

.connection-textarea {
  height: 80px;
  min-height: 60px;
}

textarea:focus {
  outline: none;
  border-color: #5F95FF;
  box-shadow: 0 0 0 2px rgba(95, 149, 255, 0.2);
}

.reset-button {
  width: 100%;
  margin-top: 15px;
  padding: 10px 15px;
  background-color: #5F95FF;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.2s;
  box-sizing: border-box;
}

.reset-button:hover {
  background-color: #4080ff;
}

.reset-button:active {
  background-color: #3d6fd9;
}

.input-rules {
  background-color: #f8f9fa;
  border: 1px solid #e9ecef;
  border-radius: 6px;
  padding: 10px;
  margin-bottom: 10px;
  font-size: 12px;
}

.rule-tip {
  margin: 0 0 5px 0;
  color: #666;
  font-size: 11px;
  line-height: 1.3;
}

.rule-example {
  margin: 0;
  padding: 6px 8px;
  background-color: #fff;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  font-family: 'Courier New', monospace;
  font-size: 11px;
  color: #495057;
  line-height: 1.2;
}

.graph-area {
  flex: 1;
  border: 1px solid #d9d9d9;
  border-radius: 6px;
  overflow: hidden;
  position: relative;
  background-color: #fafafa;
  min-height: 0;
}

#graph-container {
  width: 100%;
  height: 100%;
  background-color: #fff;
}

/* Loading 遮罩层样式 */
.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(255, 255, 255, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  backdrop-filter: blur(2px);
}

.loading-spinner {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.spinner {
  width: 40px;
  height: 40px;
  border: 3px solid #f3f3f3;
  border-top: 3px solid #5F95FF;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.loading-text {
  margin: 0;
  color: #666;
  font-size: 14px;
  font-weight: 500;
}

/* 编辑对话框样式 */
.edit-dialog-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.edit-dialog {
  background: white;
  border-radius: 8px;
  padding: 24px;
  width: 400px;
  max-width: 90vw;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);
}

.edit-dialog h4 {
  margin: 0 0 16px 0;
  color: #333;
  font-size: 16px;
  font-weight: 600;
}

.edit-textarea {
  width: 100%;
  min-height: 120px;
  padding: 12px;
  border: 1px solid #d9d9d9;
  border-radius: 4px;
  font-size: 14px;
  font-family: inherit;
  resize: vertical;
  box-sizing: border-box;
  margin-bottom: 16px;
}

.edit-textarea:focus {
  outline: none;
  border-color: #5F95FF;
  box-shadow: 0 0 0 2px rgba(95, 149, 255, 0.2);
}

.edit-dialog-buttons {
  display: flex;
  gap: 12px;
  justify-content: flex-end;
  margin-bottom: 8px;
}

.confirm-btn, .cancel-btn {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
}

.confirm-btn {
  background-color: #5F95FF;
  color: white;
}

.confirm-btn:hover {
  background-color: #4080ff;
}

.cancel-btn {
  background-color: #f5f5f5;
  color: #666;
}

.cancel-btn:hover {
  background-color: #e8e8e8;
}

.edit-tip {
  margin: 0;
  font-size: 12px;
  color: #999;
  text-align: center;
}
</style>