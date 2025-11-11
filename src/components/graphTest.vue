<template>
  <div class="insight-shell">
    <section class="insight-panel">
      <header class="insight-header">
        <div>
          <p class="insight-label">盈利拆解</p>
          <h2 class="insight-title">利润结构分析</h2>
        </div>
        <div class="insight-toolbar">
          <button type="button" @click="handleRefresh" aria-label="刷新图谱">
            <span>↻</span>
          </button>
          <button type="button" @click="() => handleZoom(1.15)" aria-label="放大图谱">
            <span>+</span>
          </button>
          <button type="button" @click="() => handleZoom(0.85)" aria-label="缩小图谱">
            <span>-</span>
          </button>
        </div>
      </header>

      <div class="insight-canvas">
        <div id="container"></div>
      </div>

      <footer class="insight-footer">
        <p>* 数据来自年报、研报和互动易，仅用于演示。</p>
      </footer>
    </section>
  </div>
</template>

<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue';
import { Rect as GRect, Text as GText } from '@antv/g';
import {
  Badge,
  CommonEvent,
  ExtensionCategory,
  Graph,
  GraphEvent,
  iconfont,
  Label,
  Polyline,
  Rect,
  register,
  subStyleProps,
  treeToGraphData,
} from '@antv/g6';

import data from './config.json';

const style = document.createElement('style');
style.innerHTML = `@import url('${iconfont.css}');`;
document.head.appendChild(style);

const COLORS = {
  B: '#1783FF',
  R: '#F46649',
  Y: '#DB9D0D',
  G: '#60C42D',
  DI: '#A7A7A7',
};
const GREY_COLOR = '#CED4D9';

const COLLAPSED_SIZE = [202, 60];
const EXPANDED_SIZE = [202, 180];

class TreeNode extends Rect {
  get data() {
    return this.context.model.getNodeLikeDatum(this.id);
  }

  get childrenData() {
    return this.context.model.getChildrenData(this.id);
  }

  getSize(attributes) {
    const expanded = attributes.expanded ?? false;
    return expanded ? EXPANDED_SIZE : COLLAPSED_SIZE;
  }

  getLabelStyle(attributes) {
    const [width, height] = this.getSize(attributes);
    return {
      x: -width / 2 + 8,
      y: -height / 2 + 16,
      text: this.data.name,
      fontSize: 12,
      opacity: 0.85,
      fill: '#000',
      cursor: 'pointer',
      fontWeight: 600,
    };
  }

  getPriceStyle(attributes) {
    const [width, height] = this.getSize(attributes);

    // 展开状态：把营收&占比挪到灰色容器上方
    if (attributes.expanded) {
      const rectStyle: any = this.getDetailContainerRectStyle(attributes);
      // rectStyle.y 是灰容器的 top，我们往上挪一点
      const y = rectStyle ? rectStyle.y : -height / 2 + 36;

      return {
        x: -width / 2 + 8,
        y,
        text: this.data.label,
        fontSize: 10,
        fill: 'gray',
        opacity: 0.85,
      };
    }

    // 未展开：保持原来的位置在卡片底部
    return {
      x: -width / 2 + 8,
      y: height / 2 - 8,
      text: this.data.label,
      fontSize: 10,
      fill: 'gray',
      opacity: 0.85,
    };
  }

  drawPriceShape(attributes, container) {
    const priceStyle = this.getPriceStyle(attributes);
    this.upsert('price', GText, priceStyle, container);
  }

  getCurrencyStyle(attributes) {
    const [, height] = this.getSize(attributes);
    return {
      x: this.shapeMap['price'].getLocalBounds().max[0] + 4,
      y: height / 2 - 8,
      text: this.data.currency,
      fontSize: 12,
      fill: '#000',
      opacity: 0.75,
    };
  }

  drawCurrencyShape(attributes, container) {
    const currencyStyle = this.getCurrencyStyle(attributes);
    this.upsert('currency', GText, currencyStyle, container);
  }

  getPercentStyle(attributes) {
    const [width, height] = this.getSize(attributes);
    return {
      x: width / 2 - 4,
      y: height / 2 - 8,
      text: `${((Number(this.data.variableValue) || 0) * 100).toFixed(2)}%`,
      fontSize: 8,
      textAlign: 'right',
      fill: COLORS[this.data.status],
    };
  }

  drawPercentShape(attributes, container) {
    const percentStyle = this.getPercentStyle(attributes);
    this.upsert('percent', GText, percentStyle, container);
  }

  getTriangleStyle(attributes) {
    const percentMinX = this.shapeMap['percent'].getLocalBounds().min[0];
    const [, height] = this.getSize(attributes);
    return {
      fill: COLORS[this.data.status],
      x: this.data.variableUp ? percentMinX - 18 : percentMinX,
      y: height / 2 - 16,
      fontFamily: 'iconfont',
      fontSize: 10,
      text: '\ue62d',
      transform: this.data.variableUp ? [] : [['rotate', 180]],
    };
  }

  drawTriangleShape(attributes, container) {
    const triangleStyle = this.getTriangleStyle(attributes);
    this.upsert('triangle', Label, triangleStyle, container);
  }

  getVariableStyle(attributes) {
    const [, height] = this.getSize(attributes);
    return {
      fill: '#000',
      fontSize: 12,
      opacity: 0.45,
      text: this.data.variableName,
      textAlign: 'right',
      x: this.shapeMap['triangle'].getLocalBounds().min[0] - 4,
      y: height / 2 - 8,
    };
  }

  drawVariableShape(attributes, container) {
    const variableStyle = this.getVariableStyle(attributes);
    this.upsert('variable', GText, variableStyle, container);
  }

  getCollapseStyle(attributes) {
    if (this.childrenData.length === 0) return false;
    const { collapsed } = attributes;
    const [width, height] = this.getSize(attributes);
    return {
      backgroundFill: '#fff',
      backgroundHeight: 16,
      backgroundLineWidth: 1,
      backgroundRadius: 0,
      backgroundStroke: GREY_COLOR,
      backgroundWidth: 16,
      cursor: 'pointer',
      fill: GREY_COLOR,
      fontSize: 16,
      text: collapsed ? '+' : '-',
      textAlign: 'center',
      textBaseline: 'middle',
      x: width / 2 + 10,
      y: 0,
    };
  }

  drawCollapseShape(attributes, container) {
    const collapseStyle = this.getCollapseStyle(attributes);
    const btn = this.upsert('collapse', Badge, collapseStyle, container);

    if (btn && !Reflect.has(btn, '__bind__')) {
      Reflect.set(btn, '__bind__', true);
      btn.addEventListener(CommonEvent.CLICK, () => {
        const { collapsed } = this.attributes;
        const graph = this.context.graph;
        if (collapsed) graph.expandElement(this.id);
        else graph.collapseElement(this.id);
      });
    }
  }

  getProcessBarStyle(attributes) {
    const { rate, status } = this.data;
    const { radius } = attributes;
    const color = COLORS[status];
    const percent = `${Number(rate) * 100}%`;
    const [width, height] = this.getSize(attributes);
    return {
      x: -width / 2,
      y: height / 2 - 4,
      width: width,
      height: 4,
      radius: [0, 0, radius, radius],
      fill: `linear-gradient(to right, ${color} ${percent}, ${GREY_COLOR} ${percent})`,
    };
  }

  drawProcessBarShape(attributes, container) {
    const processBarStyle = this.getProcessBarStyle(attributes);
    this.upsert('process-bar', GRect, processBarStyle, container);
  }

  getKeyStyle(attributes) {
    const keyStyle = super.getKeyStyle(attributes);
    return {
      ...keyStyle,
      fill: '#fff',
      lineWidth: 1,
      stroke: GREY_COLOR,
    };
  }

  getDecryptStyle(attributes) {
    const [width] = this.getSize(attributes);
    const [, height] = this.getSize(attributes);
    const expanded = attributes.expanded ?? false;
    console.log('decrypt label expanded =', attributes.expanded);
    return {
      x: -width / 2 + 150,  // 根据设计微调横坐标
      y: -height / 2 + 22,
      text: expanded ? '收起' : '解密',
      fontSize: 12,
      fill: '#1677ff',
      cursor: 'pointer',
    };
  }

  drawDecryptShape(attributes, container) {
    const decryptStyle = this.getDecryptStyle(attributes);
    const textShape = this.upsert('decrypt', GText, decryptStyle, container);

    if (!textShape) return;

    if (!Reflect.has(textShape, '__bind__')) {
      Reflect.set(textShape, '__bind__', true);
      textShape.addEventListener(CommonEvent.CLICK, (e: any) => {
        e.stopPropagation();
        console.log('clicked decrypt', this.id);
        const graph = this.context.graph;

        // 1. 先拿到当前这个节点的完整数据
        const nodeData: any = graph.getNodeData(this.id);
        const prevExpanded = nodeData.style?.expanded ?? false;
        const nextExpanded = !prevExpanded;

        // 2. 展开 / 收起 时使用不同的 size
        // const nextSize = nextExpanded ? EXPANDED_SIZE : COLLAPSED_SIZE;

        // 3. 更新这个节点的数据（一定要用 updateNodeData）
        graph.updateNodeData([
          {
            id: this.id,
            style: {
              ...nodeData.style,
              expanded: nextExpanded,
              // size: nextSize,
            },
          },
        ]);

        // 4. 重新布局，让兄弟节点跟着挪开
        graph.layout();
      });
    }
  }

  getDetailContainerRectStyle(attributes) {
    if (!attributes.expanded) return false;

    const [width, height] = this.getSize(attributes);
    const padding = 10;
    const containerWidth = width - padding * 2;
    const containerHeight = height - 60; // 上面留 60 给标题/标签

    return {
      x: -width / 2 + padding,
      y: -height / 2 + 40,
      width: containerWidth,
      height: containerHeight,
      radius: 6,
      fill: '#F7F8FA',
    };
  }

  getDetailTextStyle(attributes) {
    if (!attributes.expanded) return false;
    const rectStyle: any = this.getDetailContainerRectStyle(attributes);
    if (!rectStyle) return false;

    return {
      x: rectStyle.x + 8,
      y: rectStyle.y + 16,
      text: this.data.detail || '',
      fontSize: 12,
      fill: '#4E5969',
      opacity: 0.9,
      lineHeight: 18,
      wordWrap: true,
      wordWrapWidth: rectStyle.width - 16, // 控制换行宽度
      textAlign: 'left',
      textBaseline: 'top',
    };
  }

  drawDetailContent(attributes, container) {
    if (attributes.expanded) {
      // 展开：画灰底容器 + 文本
      const rectStyle = this.getDetailContainerRectStyle(attributes);
      if (rectStyle) {
        this.upsert('detail-container', GRect, rectStyle, container);
      }

      const textStyle = this.getDetailTextStyle(attributes);
      if (textStyle) {
        this.upsert('detail-text', GText, textStyle, container);
      }
    } else {
      // 收起：把之前画过的详情相关 shape 清理掉
      const rect = this.shapeMap['detail-container'];
      if (rect) {
        rect.destroy();                     // 从画布中移除
        delete this.shapeMap['detail-container'];
      }

      const text = this.shapeMap['detail-text'];
      if (text) {
        text.destroy();
        delete this.shapeMap['detail-text'];
      }
    }
  }

  getTagStyle(attributes) {
    // 没有 tag 就不画
    if (!this.data.tag) return false;

    const [width, height] = this.getSize(attributes);

    // 根据设计微调位置：标题右侧一点
    const x = -width / 2 + 85;          // 标题起点是 -width/2+8，往右挪一点
    const y = -height / 2 + 16;          // 和标题同一行

    return {
      backgroundFill: '#FFF1F0',         // 浅红底
      backgroundStroke: '#FF4D4F',       // 红色边框
      backgroundRadius: 4,
      backgroundWidth: 60,               // 可以固定一个宽度 或者用 textLength 算
      backgroundHeight: 15,
      x,
      y,
      text: this.data.tag,
      fontSize: 10,
      fill: '#FF4D4F'
    };
  }

  // 画 tag
  drawTagShape(attributes, container) {
    const tagStyle = this.getTagStyle(attributes);
    if (!tagStyle) return;

    this.upsert('tag', Badge, tagStyle, container);
  }

  render(attributes = this.parsedAttributes, container) {
    super.render(attributes, container);
    this.drawTagShape(attributes, container);
    this.drawPriceShape(attributes, container);
    // this.drawCurrencyShape(attributes, container);
    // this.drawPercentShape(attributes, container);
    // this.drawTriangleShape(attributes, container);
    // this.drawVariableShape(attributes, container);
    // this.drawProcessBarShape(attributes, container);
    this.drawCollapseShape(attributes, container);

    this.drawDecryptShape(attributes, container);
    this.drawDetailContent(attributes, container);
  }
}

register(ExtensionCategory.NODE, 'tree-node', TreeNode);

// 自定义边：在边的两端显示标签（百分比）
class LabelEdge extends Polyline {
  render(attributes: any = this.parsedAttributes, container?: any) {
    super.render(attributes, container);
    this.drawEndLabel(attributes, container, 'start');
    this.drawEndLabel(attributes, container, 'end');
  }

  drawEndLabel(attributes: any, container: any, type: 'start' | 'end') {
    const key = type === 'start' ? 'startLabel' : 'endLabel';
    const points = this.getPoints(attributes);
    if (!points || points.length === 0) return;

    // 获取起点或终点坐标
    const point = type === 'start' ? points[0] : points[points.length - 1];
    if (!point) return;

    const [x, y] = this.getEndpoints(attributes)[type === 'start' ? 0 : 1];;

    const fontStyle = {
      x,
      y,
      dy: type === 'start' ? -10 : -10, // 向上偏移
      dx: type === 'start' ? 15 : -15,  // 起点向右，终点向左
      fontSize: 12,
      fill: '#666',
      textBaseline: 'middle' as const,
      textAlign: (type === 'start' ? 'left' : 'right') as const,
    };

    const style = subStyleProps(attributes, key);
    const text = style?.text;
    this.upsert(`label-${type}`, GText, text ? { ...fontStyle, ...style, text } : false, container);
  }
}

register(ExtensionCategory.EDGE, 'label-edge', LabelEdge);

const graphRef = ref<Graph | null>(null);

const initGraph = async () => {
  // const response = await fetch('https://assets.antv.antgroup.com/g6/decision-tree.json');
  // const data = await response.json();
  const graph = new Graph({
    container: 'container',
    data: treeToGraphData(data, {
      getNodeData: (datum, depth) => {
        if (!datum.style) datum.style = {};
        datum.style.collapsed = depth >= 2;
        if (typeof datum.style.expanded === 'undefined') {
          datum.style.expanded = false;
        } 
        // datum.style.size = COLLAPSED_SIZE;
        if (!datum.children) return datum;
        const { children, ...restDatum } = datum;
        return { ...restDatum, children: children.map((child) => child.id) };
      },
    }),
    node: {
      type: 'tree-node',
      style: {
        size: [245, 60],
        ports: [{ placement: 'left' }, { placement: 'right' }],
        radius: 4,
      },
    },
    edge: {
      type: 'label-edge',
      style: {
        router: {
          type: 'orth',
        },
        endArrow: true,
        endArrowType: 'circle',
        radius: 8,
      },
    },
    layout: {
      type: 'mindmap',
      direction: 'LR',
      getHeight: () => 32,
      getWidth: () => 32,
      getVGap: () => 20,
      getHGap: () => 125,
    },
    behaviors: ['zoom-canvas', 'drag-canvas'],
  });

  graph.once(GraphEvent.AFTER_RENDER, () => {
    graph.fitView();
  });

  graph.render();
  graphRef.value = graph;
};

const handleRefresh = () => {
  if (graphRef.value) {
    graphRef.value.destroy();
    graphRef.value = null;
  }
  void initGraph();
};

const handleZoom = (ratio: number) => {
  const graph = graphRef.value;
  if (!graph) return;
  void graph.zoomBy(ratio, undefined, graph.getViewportCenter());
};

onMounted(() => {
  void initGraph();
});

onBeforeUnmount(() => {
  graphRef.value?.destroy();
  graphRef.value = null;
});

</script>

<style scoped>
#container {
  width: 100%;
  height: 620px;
  border-radius: 16px;
}

.insight-shell {
  padding: 32px;
  background: linear-gradient(180deg, #edf2ff 0%, #dae4ff 100%);
  border-radius: 32px;
  border: 1px solid #b6c8ff;
  min-height: 100vh;
  box-sizing: border-box;
}

.insight-panel {
  background: #ffffff;
  border-radius: 28px;
  padding: 24px 28px 18px;
  box-shadow: 0 20px 45px rgba(80, 107, 166, 0.18);
  border: 1px solid #d6e0ff;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.insight-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 16px;
}

.insight-label {
  margin: 0;
  font-size: 14px;
  color: #6d7c9c;
  letter-spacing: 0.2em;
  text-transform: uppercase;
}

.insight-title {
  margin: 4px 0 0;
  font-size: 24px;
  color: #1c2353;
  font-weight: 600;
}

.insight-toolbar {
  display: inline-flex;
  gap: 10px;
}

.insight-toolbar button {
  width: 40px;
  height: 40px;
  border-radius: 12px;
  border: 1px solid #d4dcfb;
  background: #f6f8ff;
  color: #5261a7;
  font-size: 18px;
  cursor: pointer;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.6);
  transition: all 0.2s ease;
}

.insight-toolbar button:hover {
  background: #e6edff;
  color: #2e48c7;
}

.insight-toolbar span {
  display: inline-block;
  transform: translateY(-1px);
}

.insight-canvas {
  padding: 16px;
  border-radius: 20px;
  background: #f4f6ff;
  border: 1px solid #d8e0ff;
  box-shadow: inset 0 4px 30px rgba(88, 108, 158, 0.12);
}

.insight-canvas #container {
  background: #fbfcff;
  background-image: radial-gradient(#dbe1f5 1px, transparent 1px);
  background-size: 26px 26px;
  border-radius: 18px;
}

.insight-footer {
  font-size: 12px;
  color: #8a90ad;
  text-align: right;
}
</style>
