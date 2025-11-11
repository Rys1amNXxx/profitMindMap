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

const GREY_COLOR = '#CED4D9';
const TITLE_TAG_GAP = 8;
const TAG_HORIZONTAL_PADDING = 12;
const TAG_VERTICAL_PADDING = 6;
const TAG_MIN_WIDTH = 36;

const COLLAPSED_SIZE: [number, number] = [288, 72];
const EXPANDED_SIZE: [number, number] = [288, 180];
const BASE_HORIZONTAL_GAP = 35;
const COLLAPSE_TARGET_NAME = 'collapse-button';
const HOVER_TOOLTIP_KEY = 'node-hover-tooltip';
const CLICK_TOOLTIP_KEY = 'node-click-tooltip';

const escapeHtml = (raw: string) =>
  raw
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');

const formatDetailToHtml = (detail: string) =>
  detail
    .split(/\n+/)
    .map((line) => line.trim())
    .filter(Boolean)
    .map((line) => `<p>${escapeHtml(line)}</p>`)
    .join('');

const getItemData = (item: any) => {
  if (!item) return undefined;
  return item.data ?? item.model?.data ?? item.model;
};

const isCollapseTarget = (event: any) => {
  const checkShape = (shape: any): boolean => {
    if (!shape) return false;
    const namesToCheck = [
      shape?.name,
      shape?.cfg?.name,
      shape?.attributes?.name,
      typeof shape.get === 'function' ? shape.get('name') : undefined,
      typeof shape.getAttribute === 'function' ? shape.getAttribute('name') : undefined,
    ];
    if (namesToCheck.some((name) => name === COLLAPSE_TARGET_NAME || name === 'collapse')) return true;
    return checkShape(shape.parent);
  };

  const target = event?.target ?? event?.shape ?? event?.detail?.shape;
  if (checkShape(target)) return true;

  const originalTarget = event?.originalEvent?.target;
  if (checkShape(originalTarget)) return true;

  const domEventTarget = event?.originalEvent?.originalEvent?.target;
  if (checkShape(domEventTarget)) return true;

  const key = event?.key ?? event?.name;
  if (key === COLLAPSE_TARGET_NAME || key === 'collapse') return true;

  return false;
};

const getDetailFromItems = (items: any[] | undefined, graph?: Graph) => {
  if (!items || items.length === 0) return undefined;
  const item = items[0];
  const data = getItemData(item);
  if (data?.detail) return data.detail;
  const id = item?.id ?? item?.data?.id ?? item?.model?.id;
  if (graph && id) {
    try {
      const nodeData = graph.getNodeData(id);
      return nodeData?.detail;
    } catch (e) {
      return undefined;
    }
  }
  return undefined;
};

const getEventDetail = (event: any, graph?: Graph) => {
  if (isCollapseTarget(event)) return undefined;
  const item = event?.item ?? event?.target ?? event?.items?.[0];
  const data = getItemData(item);
  if (data?.detail) return data.detail;

  const id =
    item?.id ??
    item?.data?.id ??
    item?.model?.id ??
    event?.id ??
    event?.target?.id ??
    event?.currentTarget;

  if (graph && id) {
    try {
      const nodeData = graph.getNodeData(id);
      return nodeData?.detail;
    } catch (e) {
      return undefined;
    }
  }
  return undefined;
};

function createHoverTooltipPlugin(this: Graph) {
  const graph = this;
  return {
    key: HOVER_TOOLTIP_KEY,
    type: 'tooltip' as const,
    trigger: 'hover' as const,
    itemTypes: ['node'],
    enable(event: any) {
      if (isCollapseTarget(event)) return false;
      const detail = getEventDetail(event, graph);
      if (!detail) return false;
      const clickTooltip = graph.getPluginInstance(CLICK_TOOLTIP_KEY) as any;
      const currentTarget = event?.item?.id ?? event?.target?.id ?? event?.currentTarget;
      if (clickTooltip && clickTooltip.currentTarget === currentTarget) {
        return false;
      }
      return true;
    },
    getContent: (evt: any) => {
      if (isCollapseTarget(evt)) return '';
      return `<div class="node-tooltip-hint">点击查看详细信息</div>`;
    },
  };
}

function createClickTooltipPlugin(this: Graph) {
  const graph = this;
  return {
    key: CLICK_TOOLTIP_KEY,
    type: 'tooltip' as const,
    trigger: 'click' as const,
    itemTypes: ['node'],
    enable(event: any) {
      if (isCollapseTarget(event)) return false;
      return Boolean(getEventDetail(event, graph));
    },
    getContent: (evt: any, items: any[]) => {
      if (isCollapseTarget(evt)) return '';
      const detail = getDetailFromItems(items, graph);
      if (!detail) return '';
      return `
        <div class="node-detail-tooltip">
          <div class="node-detail-tooltip__title">节点描述</div>
          <div class="node-detail-tooltip__content">${formatDetailToHtml(detail)}</div>
        </div>
      `;
    },
    onOpenChange(open: boolean) {
      if (open) {
        const hoverTooltip = graph.getPluginInstance(HOVER_TOOLTIP_KEY) as any;
        hoverTooltip?.hide?.();
      }
    },
  };
}

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
      x: -width / 2 + 10,
      y: -height / 2 + 20,
      text: this.data.name,
      fontSize: 14,
      opacity: 0.85,
      fill: '#000',
      cursor: 'pointer',
      fontWeight: 600,
      wordWrap: false,
      wordWrapWidth: width - 20,
      maxLines: 1,
      textOverflow: 'clip',
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
        fontSize: 14,
        fill: 'gray',
        opacity: 0.85,
      };
    }

    // 未展开：保持原来的位置在卡片底部
    return {
      x: -width / 2 + 8,
      y: height / 2 - 8,
      text: this.data.label,
      fontSize: 14,
      fill: 'gray',
      opacity: 0.85,
    };
  }

  drawPriceShape(attributes, container) {
    const priceStyle = this.getPriceStyle(attributes);
    this.upsert('price', GText, priceStyle, container);
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
      name: COLLAPSE_TARGET_NAME,
      tooltip: false,
    };
  }

  drawCollapseShape(attributes, container) {
    const collapseStyle = this.getCollapseStyle(attributes);
    const btn = this.upsert('collapse', Badge, collapseStyle, container);

    if (btn && !Reflect.has(btn, '__bind__')) {
      Reflect.set(btn, '__bind__', true);
      const stopPointerPropagation = (e: any) => {
        e?.stopPropagation?.();
        e?.stopImmediatePropagation?.();
      };

      btn.addEventListener(CommonEvent.POINTERENTER, stopPointerPropagation);
      btn.addEventListener(CommonEvent.POINTERMOVE, stopPointerPropagation);
      btn.addEventListener(CommonEvent.POINTERLEAVE, stopPointerPropagation);
      btn.addEventListener(CommonEvent.MOUSEENTER, stopPointerPropagation);
      btn.addEventListener(CommonEvent.MOUSEMOVE, stopPointerPropagation);
      btn.addEventListener(CommonEvent.MOUSELEAVE, stopPointerPropagation);
      btn.addEventListener(CommonEvent.CLICK, (event: any) => {
        stopPointerPropagation(event);
        const { collapsed } = this.attributes;
        const graph = this.context.graph;
        if (collapsed) {
          graph.expandElement(this.id);
        } else {
          graph.collapseElement(this.id);
        }
      });
    }
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

  getLabelTextShape() {
    if (!this.shapeMap) return undefined;
    const shapes = Object.values(this.shapeMap) as any[];
    return shapes.find(
      (shape) => shape instanceof GText && shape?.attributes?.text === this.data.name,
    );
  }

  getLabelBounds() {
    const labelShape: any = this.getLabelTextShape();
    if (labelShape?.getLocalBounds) {
      return labelShape.getLocalBounds();
    }
    return undefined;
  }

  estimateTextWidth(text: string | number | undefined, fontSize: number) {
    if (!text) return 0;
    const str = String(text);
    return Array.from(str).reduce((sum, char) => {
      const isFullWidth = char.charCodeAt(0) > 255;
      const ratio = isFullWidth ? 1 : 0.6;
      return sum + fontSize * ratio;
    }, 0);
  }

  getTagStyle(attributes) {
    // 没有 tag 就不画
    if (!this.data.tag) return false;

    const labelStyle: any = this.getLabelStyle(attributes);
    const labelBounds = this.getLabelBounds();
    const labelRight = labelBounds ? labelBounds.max[0] : labelStyle.x + this.estimateTextWidth(labelStyle.text, labelStyle.fontSize ?? 12);
    const x = labelRight + TITLE_TAG_GAP;
    const y = labelStyle.y;          // 与标题同一行
    const tagFontSize = 12;
    const tagTextWidth = this.estimateTextWidth(this.data.tag, tagFontSize);
    const backgroundWidth = Math.max(TAG_MIN_WIDTH, tagTextWidth + TAG_HORIZONTAL_PADDING);
    const backgroundHeight = tagFontSize + TAG_VERTICAL_PADDING;

    const tagText = this.data.tag as string;
    const isCoreCashCow = tagText === '核心现金牛';

    return {
      backgroundFill: isCoreCashCow ? '#FFF1F0' : '#7684961F',
      backgroundStroke: isCoreCashCow ? '#FF4D4F' : '#7684961F',
      backgroundRadius: 4,
      backgroundWidth,
      backgroundHeight,
      x,
      y,
      text: tagText,
      fontSize: tagFontSize,
      fill: isCoreCashCow ? '#FF4D4F' : '#768496',
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
    this.drawCollapseShape(attributes, container);
  }
}

register(ExtensionCategory.NODE, 'tree-node', TreeNode);

// 自定义边：在边的终点显示标签（百分比）
class LabelEdge extends Polyline {
  private layoutListener?: () => void;

  private ensureLayoutListener() {
    if (this.layoutListener) return;
    const graph = this.context?.graph;
    if (!graph?.on) return;
    this.layoutListener = () => {
      this.drawEndLabel(this.parsedAttributes, this);
    };
    graph.on(GraphEvent.AFTER_LAYOUT, this.layoutListener);
  }

  render(attributes: any = this.parsedAttributes, container: any = this) {
    this.ensureLayoutListener();
    super.render(attributes, container);
    // 只在终点（子节点侧）显示百分比
    this.drawEndLabel(attributes, container);
  }

  update(attributes: any = this.parsedAttributes, container: any = this) {
    super.update(attributes);
    this.drawEndLabel(this.parsedAttributes, container);
  }

  destroy(): void {
    const graph = this.context?.graph;
    if (this.layoutListener && graph?.off) {
      graph.off(GraphEvent.AFTER_LAYOUT, this.layoutListener);
      this.layoutListener = undefined;
    }
    super.destroy();
  }

  drawEndLabel(attributes: any, container: any) {
    const targetContainer = container ?? this;
    const style = subStyleProps(attributes, 'endLabel');
    const text = style?.text;

    // 如果没有文本，则不显示标签
    if (!text) {
      this.upsert('label-end', GText, false, targetContainer);
      return;
    }

    // 获取边的路径点
    const points = this.getPoints(attributes);
    if (!points || points.length === 0) {
      this.upsert('label-end', GText, false, targetContainer);
      return;
    }

    // 获取终点坐标（连接到目标节点的端点）
    const endpoints = this.getEndpoints(attributes);
    const [x, y] = endpoints[1]; // 终点

    // 标签样式：在终点左侧显示，向上偏移
    const fontStyle = {
      x,
      y,
      dy: -10,           // 向上偏移10px
      dx: -15,           // 向左偏移15px，避免与节点重叠
      fontSize: 12,
      fill: '#666',
      textBaseline: 'middle' as const,
      textAlign: 'right' as const,  // 右对齐，因为标签在节点左侧
      text,
    };

    this.upsert('label-end', GText, { ...fontStyle, ...style }, targetContainer);
  }
}

register(ExtensionCategory.EDGE, 'label-edge', LabelEdge);

const graphRef = ref<Graph | null>(null);

const initGraph = async () => {
  // const response = await fetch('https://assets.antv.antgroup.com/g6/decision-tree.json');
  // const data = await response.json();

  // 创建一个变量来存储 graph 引用，用于布局函数中访问
  let graphInstance: Graph | null = null;

  const nodeHasChildrenMap = new Map<string, boolean>();

  const buildNodeMaps = (node: any) => {
    const hasChildren = Array.isArray(node.children) && node.children.length > 0;
    nodeHasChildrenMap.set(node.id, hasChildren);
    if (hasChildren) {
      node.children.forEach((child: any) => buildNodeMaps(child));
    }
  };
  buildNodeMaps(data);

  const graph = new Graph({
    container: 'container',
    data: treeToGraphData(data, {
      getNodeData: (datum, depth) => {
        if (!datum.style) datum.style = {};
        datum.style.size = [...COLLAPSED_SIZE];
        // 只在第4层及以后才默认折叠，让前三层都能正常显示
        datum.style.collapsed = depth >= 5;
        if (typeof datum.style.expanded === 'undefined') {
          datum.style.expanded = false;
        }
        // datum.style.size = COLLAPSED_SIZE;
        if (!datum.children) return datum;
        const { children, ...restDatum } = datum;
        return { ...restDatum, children: children.map((child) => child.id) };
      },
      getEdgeData: (source, target) => {
        // 如果目标节点有 rate 数据，则在边线终点显示百分比
        const rate = target?.rate;
        if (rate !== undefined && rate !== null) {
          const percentage = (rate * 100).toFixed(1) + '%';
          return {
            source: source.id,
            target: target.id,
            style: {
              labelText: percentage,
            }
          };
        }
        return {
          source: source.id,
          target: target.id,
        };
      },
    }),
    node: {
      type: 'tree-node',
      style: {
        size: [...COLLAPSED_SIZE],
        ports: [{ placement: 'left' }, { placement: 'right' }],
        radius: 4,
      },
    },
    edge: {
      type: 'label-edge',
      style: {
        stroke: '#CED4D9',
        lineWidth: 1,
        router: {
          type: 'orth',
        },
        endArrow: true,
        endArrowType: 'circle',
        radius: 8,
        // labelText: '100%',
      },
    },
    layout: {
      type: 'mindmap',
      direction: 'LR',
      getHeight: (node?: any) => {
        if (!node || !graphInstance) return COLLAPSED_SIZE[1];
        const nodeId = typeof node === 'string' ? node : node.id ?? node.data?.id;
        if (!nodeId) return COLLAPSED_SIZE[1];
        try {
          const nodeData = graphInstance.getNodeData(nodeId);
          const expanded = nodeData?.style?.expanded ?? false;
          return (expanded ? EXPANDED_SIZE : COLLAPSED_SIZE)[1];
        } catch (e) {
          return COLLAPSED_SIZE[1];
        }
      },
      getWidth: (node?: any) => {
        if (!node || !graphInstance) return COLLAPSED_SIZE[0];
        const nodeId = typeof node === 'string' ? node : node.id ?? node.data?.id;
        if (!nodeId) return COLLAPSED_SIZE[0];
        try {
          const nodeData = graphInstance.getNodeData(nodeId);
          const expanded = nodeData?.style?.expanded ?? false;
          return (expanded ? EXPANDED_SIZE : COLLAPSED_SIZE)[0];
        } catch (e) {
          return COLLAPSED_SIZE[0];
        }
      },
      getVGap: () => {
        // 使用固定的垂直间距
        return 20;
      },
      getHGap: () => BASE_HORIZONTAL_GAP,
    },
    plugins: [
      function () {
        return createClickTooltipPlugin.call(this);
      },
      function () {
        return createHoverTooltipPlugin.call(this);
      },
    ],
    behaviors: ['zoom-canvas', 'drag-canvas'],
  });

  // 保存 graph 实例引用
  graphInstance = graph;

  const hideClickTooltip = () => {
    const clickTooltip = graph.getPluginInstance(CLICK_TOOLTIP_KEY) as any;
    clickTooltip?.hide?.();
  };

  const hideHoverTooltip = () => {
    const hoverTooltip = graph.getPluginInstance(HOVER_TOOLTIP_KEY) as any;
    hoverTooltip?.hide?.();
  };

  graph.once(GraphEvent.AFTER_RENDER, () => {
    graph.fitView();
  });

  graph.on('tooltip:show', (evt: any) => {
    if (!evt) return;
    const sourceEvent = evt?.originalEvent ?? evt?.event ?? evt;
    if (isCollapseTarget(sourceEvent)) {
      evt?.preventDefault?.();
      hideClickTooltip();
      hideHoverTooltip();
    }
  });

  graph.on('canvas:click', () => {
    hideClickTooltip();
    hideHoverTooltip();
  });

  // 监听节点展开/折叠后的布局更新事件，重新渲染边以更新标签位置
  graph.on(GraphEvent.AFTER_LAYOUT, () => {
    // 布局完成后，强制更新所有边的渲染
    // 使用 draw 方法重新绘制图形
    graph.draw();
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

:deep(.node-tooltip-hint) {
  /* padding: 6px 12px; */
  background: transparent;
  border: none;
  border-radius: 14px;
  font-weight: 600;
  color: #4A5465;
  font-size: 14px;
  line-height: 1.4;
  /* box-shadow: 0 8px 16px rgba(22, 119, 255, 0.15); */
  white-space: nowrap;
}

:deep(.node-detail-tooltip) {
  /* max-width: 320px; */
  /* padding: 14px 16px; */
  background: #fff;
  border-radius: 14px;
  /* box-shadow: 0 12px 32px rgba(31, 41, 79, 0.18); */
  border: none;
  color: #4A5465;
  font-size: 14px;
  line-height: 1.6;
  white-space: normal;
  word-break: break-word;
  font-weight: 400;
}

:deep(.node-detail-tooltip__title) {
  font-weight: 600;
  color: #768496;
  margin-bottom: 8px;
  font-size: 14px;
}

:deep(.node-detail-tooltip__content p) {
  margin: 0 0 6px;
}

:deep(.node-detail-tooltip__content p:last-child) {
  margin-bottom: 0;
}
</style>
