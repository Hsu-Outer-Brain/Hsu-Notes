# Streamlit - 知乎
![](https://pic3.zhimg.com/v2-cc90016284c98155c32798716b80b62f_720w.jpg?source=3af55fa1)

嗨，今天给大家分享一款可以在 Streamlit 绘制媲美专业软件的脑图、思维导图绘制方法。  
公众号的朋友们有福了，从此绘制脑图，不用再到处找软件了，也可以发布在服务器上供其他朋友或同事们使用了。

### **今天的实现效果**

**今天的实现代码**

1、streamlit 程序部分

```python3
import streamlit as st
import streamlit.components.v1 as components
import random
st.set_page_config(page_title="可编辑脑图", layout="wide")
p = open("naotu.html", encoding="utf-8")
components.html(p.read(), height=1000, width=1800, scrolling=True)
choose = st.sidebar.selectbox("是否要进行截图保存", ("是", "否"))
if choose == "是":
  import PIL.ImageGrab
  import time
  time.sleep(3)
  scr = PIL.ImageGrab.grab()
  scr.save("screen.png")
else:
  pass

```

2、naotu.html 部分

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Tutorial Demo</title>
</head>
<body>
  <div id="container" style="height: 1500;"></div>

<script src="https://gw.alipayobjects.com/os/lib/antv/g6/4.6.4/dist/g6.min.js"></script>
<script>

const {
  Util
} = G6;

const colorArr = [
  '#5B8FF9',
  '#5AD8A6',
  '#5D7092',
  '#F6BD16',
  '#6F5EF9',
  '#6DC8EC',
  '#D3EEF9',
  '#DECFEA',
  '#FFE0C7',
  '#1E9493',
  '#BBDEDE',
  '#FF99C3',
  '#FFE0ED',
  '#CDDDFD',
  '#CDF3E4',
  '#CED4DE',
  '#FCEBB9',
  '#D3CEFD',
  '#945FB9',
  '#FF9845',
];

const rawData = {
  label: '学校',
  id: '0',
  children: [{
      label: '幼儿园',
      id: '0-1',
      color: '#5AD8A6',
      children: [{
          label: '小班',
          id: '0-1-1',
        },
        {
          label: '中班',
          id: '0-1-2',
        },
      ],
    },
    {
      label: '小学',
      id: '0-2',
      color: '#F6BD16',
      children: [{
          label: '1年级',
          id: '0-2-1',
        },
        {
          label: '2年级',
          id: '0-2-2',
        },
      ],
    },
    {
      label: '初中',
      id: '0-3',
      color: '#269A99',
      children: [{
          label: '初一',
          id: '0-3-1',
        },
        {
          label: '初二',
          id: '0-3-2',
        },
      ],
    },
    {
      label: '高中',
      id: '0-4',
      color: '#269A99',
      children: [{
          label: '高一',
          id: '0-4-1',
        },
        {
          label: '高二',
          id: '0-4-2',
        },
      ],
    },
  ],
};

G6.registerNode(
  'dice-mind-map-root', {
    jsx: (cfg) => {
      const width = Util.getTextSize(cfg.label, 16)[0] + 24;
      const stroke = cfg.style.stroke || '#096dd9';
      const fill = cfg.style.fill;

      return `
      <group>
        <rect draggable="true" style={{width: ${width}, height: 42, stroke: ${stroke}, radius: 4}} keyshape>
          <text style={{ fontSize: 16, marginLeft: 12, marginTop: 12 }}>${cfg.label}</text>
          <text style={{ marginLeft: ${
            width - 16
          }, marginTop: -20, stroke: '#66ccff', fill: '#000', cursor: 'pointer', opacity: ${
        cfg.hover ? 0.75 : 0
      } }} action="add">+</text>
        </rect>
      </group>
    `;
    },
    getAnchorPoints() {
      return [
        [0, 0.5],
        [1, 0.5],
      ];
    },
  },
  'single-node',
);
G6.registerNode(
  'dice-mind-map-sub', {
    jsx: (cfg) => {
      const width = Util.getTextSize(cfg.label, 14)[0] + 24;
      const color = cfg.color || cfg.style.stroke;

      return `
      <group>
        <rect draggable="true" style={{width: ${width + 24}, height: 22}} keyshape>
          <text draggable="true" style={{ fontSize: 14, marginLeft: 12, marginTop: 6 }}>${
            cfg.label
          }</text>
          <text style={{ marginLeft: ${
            width - 8
          }, marginTop: -10, stroke: ${color}, fill: '#000', cursor: 'pointer', opacity: ${
        cfg.hover ? 0.75 : 0
      }, next: 'inline' }} action="add">+</text>
          <text style={{ marginLeft: ${
            width - 4
          }, marginTop: -10, stroke: ${color}, fill: '#000', cursor: 'pointer', opacity: ${
        cfg.hover ? 0.75 : 0
      }, next: 'inline' }} action="delete">-</text>
        </rect>
        <rect style={{ fill: ${color}, width: ${width + 24}, height: 2, x: 0, y: 22 }} />

      </group>
    `;
    },
    getAnchorPoints() {
      return [
        [0, 0.965],
        [1, 0.965],
      ];
    },
  },
  'single-node',
);
G6.registerNode(
  'dice-mind-map-leaf', {
    jsx: (cfg) => {
      const width = Util.getTextSize(cfg.label, 12)[0] + 24;
      const color = cfg.color || cfg.style.stroke;

      return `
      <group>
        <rect draggable="true" style={{width: ${width + 20}, height: 26, fill: 'transparent' }}>
          <text style={{ fontSize: 12, marginLeft: 12, marginTop: 6 }}>${cfg.label}</text>
              <text style={{ marginLeft: ${
                width - 8
              }, marginTop: -10, stroke: ${color}, fill: '#000', cursor: 'pointer', opacity: ${
        cfg.hover ? 0.75 : 0
      }, next: 'inline' }} action="add">+</text>
              <text style={{ marginLeft: ${
                width - 4
              }, marginTop: -10, stroke: ${color}, fill: '#000', cursor: 'pointer', opacity: ${
        cfg.hover ? 0.75 : 0
      }, next: 'inline' }} action="delete">-</text>
        </rect>
        <rect style={{ fill: ${color}, width: ${width + 24}, height: 2, x: 0, y: 32 }} />

      </group>
    `;
    },
    getAnchorPoints() {
      return [
        [0, 0.965],
        [1, 0.965],
      ];
    },
  },
  'single-node',
);
G6.registerBehavior('dice-mindmap', {
  getEvents() {
    return {
      'node:click': 'clickNode',
      'node:dblclick': 'editNode',
      'node:mouseenter': 'hoverNode',
      'node:mouseleave': 'hoverNodeOut',
    };
  },
  clickNode(evt) {
    const model = evt.item.get('model');
    const name = evt.target.get('action');
    switch (name) {
      case 'add':
        const newId =
          model.id +
          '-' +
          (((model.children || []).reduce((a, b) => {
              const num = Number(b.id.split('-').pop());
              return a < num ? num : a;
            }, 0) || 0) +
            1);
        evt.currentTarget.updateItem(evt.item, {
          children: (model.children || []).concat([{
            id: newId,
            direction: newId.charCodeAt(newId.length - 1) % 2 === 0 ? 'right' : 'left',
            label: 'New',
            type: 'dice-mind-map-leaf',
            color: model.color || colorArr[Math.floor(Math.random() * colorArr.length)],
          }, ]),
        });
        evt.currentTarget.layout(false);
        break;
      case 'delete':
        const parent = evt.item.get('parent');
        evt.currentTarget.updateItem(parent, {
          children: (parent.get('model').children || []).filter((e) => e.id !== model.id),
        });
        evt.currentTarget.layout(false);
        break;
      case 'edit':
        break;
      default:
        return;
    }
  },
  editNode(evt) {
    const item = evt.item;
    const model = item.get('model');
    const {
      x,
      y
    } = item.calculateBBox();
    const graph = evt.currentTarget;
    const realPosition = evt.currentTarget.getClientByPoint(x, y);
    const el = document.createElement('div');
    const fontSizeMap = {
      'dice-mind-map-root': 24,
      'dice-mind-map-sub': 18,
      'dice-mind-map-leaf': 16,
    };
    el.style.fontSize = fontSizeMap[model.type] + 'px';
    el.style.position = 'fixed';
    el.style.top = realPosition.y + 'px';
    el.style.left = realPosition.x + 'px';
    el.style.paddingLeft = '12px';
    el.style.transformOrigin = 'top left';
    el.style.transform = `scale(${evt.currentTarget.getZoom()})`;
    const input = document.createElement('input');
    input.style.border = 'none';
    input.value = model.label;
    input.style.width = Util.getTextSize(model.label, fontSizeMap[model.type])[0] + 'px';
    input.className = 'dice-input';
    el.className = 'dice-input';
    el.appendChild(input);
    document.body.appendChild(el);
    const destroyEl = () => {
      document.body.removeChild(el);
    };
    const clickEvt = (event) => {
      if (!(event.target && event.target.className && event.target.className.includes('dice-input'))) {
        window.removeEventListener('mousedown', clickEvt);
        window.removeEventListener('scroll', clickEvt);
        graph.updateItem(item, {
          label: input.value,
        });
        graph.layout(false);
        graph.off('wheelZoom', clickEvt);
        destroyEl();
      }
    };
    graph.on('wheelZoom', clickEvt);
    window.addEventListener('mousedown', clickEvt);
    window.addEventListener('scroll', clickEvt);
    input.addEventListener('keyup', (event) => {
      if (event.key === 'Enter') {
        clickEvt({
          target: {},
        });
      }
    });
  },
  hoverNode(evt) {
    evt.currentTarget.updateItem(evt.item, {
      hover: true,
    });
  },
  hoverNodeOut(evt) {
    evt.currentTarget.updateItem(evt.item, {
      hover: false,
    });
  },
});
G6.registerBehavior('scroll-canvas', {
  getEvents: function getEvents() {
    return {
      wheel: 'onWheel',
    };
  },

  onWheel: function onWheel(ev) {
    const {
      graph
    } = this;
    if (!graph) {
      return;
    }
    if (ev.ctrlKey) {
      const canvas = graph.get('canvas');
      const point = canvas.getPointByClient(ev.clientX, ev.clientY);
      let ratio = graph.getZoom();
      if (ev.wheelDelta > 0) {
        ratio += ratio * 0.05;
      } else {
        ratio *= ratio * 0.05;
      }
      graph.zoomTo(ratio, {
        x: point.x,
        y: point.y,
      });
    } else {
      const x = ev.deltaX || ev.movementX;
      const y = ev.deltaY || ev.movementY || (-ev.wheelDelta * 125) / 3;
      graph.translate(-x, -y);
    }
    ev.preventDefault();
  },
});

const dataTransform = (data) => {
  const changeData = (d, level = 0, color) => {
    const data = {
      ...d,
    };
    switch (level) {
      case 0:
        data.type = 'dice-mind-map-root';
        break;
      case 1:
        data.type = 'dice-mind-map-sub';
        break;
      default:
        data.type = 'dice-mind-map-leaf';
        break;
    }

    data.hover = false;

    if (color) {
      data.color = color;
    }

    if (level === 1 && !d.direction) {
      if (!d.direction) {
        data.direction = d.id.charCodeAt(d.id.length - 1) % 2 === 0 ? 'right' : 'left';
      }
    }

    if (d.children) {
      data.children = d.children.map((child) => changeData(child, level + 1, data.color));
    }
    return data;
  };
  return changeData(data);
};

const container = document.getElementById('container');
// const el = document.createElement('pre');
// el.innerHTML =
//   'Double click on node to change title, hover node to display ops buttons\n双击修改节点标题, hover节点显示操作按钮';

// container.appendChild(el);

const width = container.scrollWidth;
const height = 800;
const tree = new G6.TreeGraph({
  container: 'container',
  width,
  height,
  fitView: true,
  fitViewPadding: [10, 20],
  layout: {
    type: 'mindmap',
    direction: 'H',
    getHeight: () => {
      return 16;
    },
    getWidth: (node) => {
      return node.level === 0 ?
        Util.getTextSize(node.label, 16)[0] + 12 :
        Util.getTextSize(node.label, 12)[0];
    },
    getVGap: () => {
      return 10;
    },
    getHGap: () => {
      return 60;
    },
    getSide: (node) => {
      return node.data.direction;
    },
  },
  defaultEdge: {
    type: 'cubic-horizontal',
    style: {
      lineWidth: 2,
    },
  },
  minZoom: 0.5,
  modes: {
    default: ['drag-canvas', 'zoom-canvas', 'dice-mindmap'],
  },
});

tree.data(dataTransform(rawData));

tree.render();


</script>
</body>
</html>

```

### **实现原理简介**

1、主题思路为：在 naotu.html 中写入脑图的初始配置，在 streamlit 网页中通过 open 的方法，将 html 内容读取出来，使用 st.components.v1.html 组件在 streamlit 网页中进行渲染出来。  
2、依赖的 js 文件引入方法为：

```text
<script src="https://gw.alipayobjects.com/os/lib/antv/g6/4.6.4/dist/g6.min.js"></script>

```

这个库是蚂蚁集团开发的一个针对关系类型的处理库，可以很方便的在网页中进行关系图的绘制，本文针对脑图的案例进行分享。

3、G6 库的网页首页地址：  
[https://antv-g6.gitee.io/zh](http://link.zhihu.com/?target=https%3A//antv-g6.gitee.io/zh)  
4、naotu.html 网页中的初始值需要在定义的如下 javascript 字典中进行写入

```js
const rawData = {
  label: '学校',
  id: '0',
  children: [{
      label: '幼儿园',
      id: '0-1',
      color: '#5AD8A6',
      children: [{
          label: '小班',
          id: '0-1-1',
        },
        {
          label: '中班',
          id: '0-1-2',
        },
      ],
    },
    {
      label: '小学',
      id: '0-2',
      color: '#F6BD16',
      children: [{
          label: '1年级',
          id: '0-2-1',
        },
        {
          label: '2年级',
          id: '0-2-2',
        },
      ],
    },
    {
      label: '初中',
      id: '0-3',
      color: '#269A99',
      children: [{
          label: '初一',
          id: '0-3-1',
        },
        {
          label: '初二',
          id: '0-3-2',
        },
      ],
    },
    {
      label: '高中',
      id: '0-4',
      color: '#269A99',
      children: [{
          label: '高一',
          id: '0-4-1',
        },
        {
          label: '高二',
          id: '0-4-2',
        },
      ],
    },
  ],
};

```

方法和 Python 里的字典方法非常类似，子分支通过 children 数组方法进行添加。

作者同时添加了对屏幕进行截图的功能，可以在使用的时候参考。  

### Streamlit 交流群

欢迎对 Streamlit 网页应用编程感兴趣的朋友，加入我们的讨论群，一起碰撞灵感。 
 [https://www.zhihu.com/people/beyondthehorizon](https://www.zhihu.com/people/beyondthehorizon)
