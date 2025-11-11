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
const TITLE_TAG_GAP = 8;
const TAG_HORIZONTAL_PADDING = 12;
const TAG_VERTICAL_PADDING = 6;
const TAG_MIN_WIDTH = 36;

const COLLAPSED_SIZE: [number, number] = [288, 72];
const EXPANDED_SIZE: [number, number] = [288, 180];
const BASE_VERTICAL_GAP = 10;
const ROOT_VERTICAL_GAP = 10;
const EXPANDED_VERTICAL_GAP = 12;
const BASE_HORIZONTAL_GAP = 50;
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
    onOpenChange(open: boolean) {
      if (open) {
        const clickTooltip = graph.getPluginInstance(CLICK_TOOLTIP_KEY) as any;
        clickTooltip?.hide?.();
      }
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

        const camera = graph.getCanvas().getCamera();
        const zoom = camera.getZoom();
        const position = camera.getPosition();
        const focus = graph.getViewportCenter();

        if (collapsed) {
          graph.expandElement(this.id);
        } else {
          graph.collapseElement(this.id);
        }
        graph.once(GraphEvent.AFTER_LAYOUT, () => {
          if (graph.destroyed) return;
          camera.setZoom(zoom);
          camera.setPosition(position.x, position.y);
          graph.focusViewport(focus);
        });
        graph.layout();
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

    return {
      backgroundFill: '#FFF1F0',         // 浅红底
      backgroundStroke: '#FF4D4F',       // 红色边框
      backgroundRadius: 4,
      backgroundWidth,
      backgroundHeight,
      x,
      y,
      text: this.data.tag,
      fontSize: tagFontSize,
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

    // this.drawDecryptShape(attributes, container);
    // this.drawDetailContent(attributes, container);
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

  // 创建一个变量来存储 graph 引用，用于布局函数中访问
  let graphInstance: Graph | null = null;

  // 创建一个映射来存储节点的父节点信息
  const parentMap = new Map<string, string>();

  // 递归遍历数据，建立父子关系映射
  const buildParentMap = (node: any, parentId?: string) => {
    if (parentId) {
      parentMap.set(node.id, parentId);
    }
    if (node.children) {
      node.children.forEach((child: any) => {
        buildParentMap(child, node.id);
      });
    }
  };
  buildParentMap(data);

  const graph = new Graph({
    container: 'container',
    data: treeToGraphData(data, {
      getNodeData: (datum, depth) => {
        if (!datum.style) datum.style = {};
        datum.style.size = [...COLLAPSED_SIZE];
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
        endLabelText: '100%',
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
      getVGap: (node?: any) => {
        // 如果没有节点数据或 graph 实例，返回默认间距
        if (!node || !graphInstance) return BASE_VERTICAL_GAP;

        try {
          // 获取当前节点的 ID（支持多种格式）
          let nodeId: string | undefined;
          if (typeof node === 'string') {
            nodeId = node;
          } else if (node.id) {
            nodeId = node.id;
          } else if (node.data?.id) {
            nodeId = node.data.id;
          }

          if (!nodeId) return BASE_VERTICAL_GAP;

          // 通过 parentMap 查找父节点
          const parentId = parentMap.get(nodeId);

          if (parentId) {
            // 根节点（id 为 'root'）默认是展开的，所以根节点的子节点之间应该有更大的间距
            if (parentId === 'root') {
              return ROOT_VERTICAL_GAP;
            }

            // 如果有父节点，检查父节点是否展开了子节点
            try {
              const parentData = graphInstance?.getNodeData(parentId);
              if (parentData) {
                const parentCollapsed = parentData.style?.collapsed ?? true;
                // 如果父节点展开了子节点（collapsed 为 false），返回更大的间距
                // 这样父节点的所有子节点之间都会有更大的间距
                return parentCollapsed ? BASE_VERTICAL_GAP : EXPANDED_VERTICAL_GAP;
              }
            } catch (e) {
              // 如果获取父节点数据失败，返回默认间距
            }
          }

          // 如果没有父节点（根节点），检查当前节点本身是否有子节点且展开
          try {
            const nodeData = graphInstance?.getNodeData(nodeId);
            if (nodeData) {
              const children = nodeData?.children;
              if (children && children.length > 0) {
                const collapsed = nodeData?.style?.collapsed ?? true;
                return collapsed ? BASE_VERTICAL_GAP : EXPANDED_VERTICAL_GAP;
              }
            }
          } catch (e) {
            // 如果获取节点数据失败，继续使用默认值
          }

          return BASE_VERTICAL_GAP;
        } catch (e) {
          // 如果获取数据失败，返回默认间距
          return BASE_VERTICAL_GAP;
        }
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

  graph.once(GraphEvent.AFTER_RENDER, () => {
    graph.fitView();
  });

  graph.on('tooltip:show', (evt: any) => {
    if (!evt) return;
    const sourceEvent = evt?.originalEvent ?? evt?.event ?? evt;
    if (isCollapseTarget(sourceEvent)) {
      evt?.preventDefault?.();
      const clickTooltip = graph.getPluginInstance(CLICK_TOOLTIP_KEY) as any;
      const hoverTooltip = graph.getPluginInstance(HOVER_TOOLTIP_KEY) as any;
      clickTooltip?.hide?.();
      hoverTooltip?.hide?.();
    }
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
  font-weight: 400;
  color: #000000;
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
