# fiber

# 一、为什么需要fiber

ReactElement的结构用于描述dom的结构，但是这样的结构却无法实现时间分片、和异步更新，所以现在需要一个结构来承载这些功能。这个结构需要可以相互连接成树结构、串联reactElement和真实DOM结构、保存组件相关信息、需要保存更新相关信息，由此fiber应运而生

```json
const element = {
  $$typeof: REACT_ELEMENT_TYPE,
  type: type, // 创建元素时传入的类型（标签名或组件函数）
  key: key, // 元素在列表中的唯一标识
  ref: ref, // 元素的引用
  props: props, // 元素的属性
  _owner: owner, // 创建该元素的组件实例
};
```

举个例子

```js
{
  "type": "div", // 元素类型为 div
  "props": {
    "className": "container", // 添加类名 container
    "onClick": "handleClick" // 添加点击事件处理函数 handleClick
  },
  "children": [
    {
      "type": "h1", // 元素类型为 h1
      "props": {
        "className": "title", // 添加类名 title
        "style": {
          "color": "red" // 设置红色字体样式
        }
      },
      "children": [
        "Hello, World!" // h1 元素的文本内容为 Hello, World!
      ]
    },
    {
      "type": "p", // 元素类型为 p
      "props": {
        "className": "content" // 添加类名 content
      },
      "children": [
        "This is a paragraph." // p 元素的文本内容为 This is a paragraph.
      ]
    }
  ]
}
```

# 二、fiber是什么？

fiber可以认为是react16之后的虚拟DOM

# 三、fiber的组成

由于需要可以相互连接成树结构、串联reactElement和真实DOM结构、保存组件相关信息、需要保存更新相关信息，所以fiber的属性主要分为如下三大类

```js
export class FiberNode {
    // =============实例属性==================
    this.tag = tag;
    this.key = key;
    // HostComponent div ,div DOM
    this.stateNode = null;
    // fiberNode的类型，比如FunctionComponent,就是函数组件本身
    this.type = null;

    // =============构成树状结构==================
    // 指向父fiberNode
    this.return = null;
    // 指向右侧兄弟fiberNode
    this.sibling = null;
    // 指向子fiberNode
    this.child = null;
    // 同级子节点index
    this.index = 0;
    this.ref = null;

    //====================作为工作单元============
    // 刚开始工作的props
    this.pendingProps = pendingProps;
    // 工作完确定的props
    this.memorizeProps = null;
    this.memorizedState = null;
    // 更新队列
    this.updateQueue = null;
    this.alternate = null;

    // 副作用
    this.flags = NoFlags;
    this.subtreeFlags = NoFlags;
}
```

# 四、fiber的含义&实现原理

## 1.保存组件基本信息

```react
// Fiber对应组件的类型 Function/Class/Host...
this.tag = tag;
// key属性
this.key = key;
// 大部分情况同type，某些情况不同，比如FunctionComponent使用React.memo包裹
this.elementType = null;
// 对于 FunctionComponent，指函数本身，对于ClassComponent，指class，对于HostComponent，指DOM节点tagName
this.type = null;
// Fiber对应的真实DOM节点
this.stateNode = null;
```

## 2.作为动态的工作单元

保存更新相关信息

```js
// 保存本次更新造成的状态改变相关信息
this.pendingProps = pendingProps;
this.memoizedProps = null;
this.updateQueue = null;
this.memoizedState = null;
this.dependencies = null;

this.mode = mode;

// 保存本次更新会造成的DOM操作
this.effectTag = NoEffect;
this.nextEffect = null;

this.firstEffect = null;
this.lastEffect = null;

// 调度优先级相关
this.lanes = NoLanes;
this.childLanes = NoLanes;
```

## 3.作为架构来说

每个Fiber节点有个对应的`React element`，多个`Fiber节点`是如何连接形成树呢？靠如下三个属性：

```react
// 指向父级Fiber节点
this.return = null;
// 指向子Fiber节点
this.child = null;
// 指向右边第一个兄弟Fiber节点
this.sibling = null;
```



```react
function App() {
  return (
    <div>
      hello
      <span>world</span>
    </div>
  )
}
```



<img src="https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/image-20230615164326632.png" alt="image-20230615164326632" style="zoom:50%;" />

# 五、fiber的工作原理

## 1.双缓存

当我们用`canvas`绘制动画，每一帧绘制前都会调用`ctx.clearRect`清除上一帧的画面。

如果当前帧画面计算量比较大，导致清除上一帧画面到绘制当前帧画面之间有较长间隙，就会出现白屏。

为了解决这个问题，我们可以在内存中绘制当前帧动画，绘制完毕后直接用当前帧替换上一帧画面，由于省去了两帧替换间的计算时间，不会出现从白屏到出现画面的闪烁情况。

这种**在内存中构建并直接替换**的技术叫做[双缓存](https://baike.baidu.com/item/双缓冲)。

`React`使用“双缓存”来完成`Fiber树`的构建与替换——对应着`DOM树`的创建与更新。

## 2.双缓存fiber树

在 React 中，双缓存的概念也可以应用于其内部的 Fiber 树结构。Fiber 是 React 16 中引入的一种新的调度算法和数据结构，用于实现增量式的、可中断的和可恢复的更新过程。

在 Fiber 架构中，存在两个主要的 Fiber 树：当前正在渲染的树（current tree）和准备好的树（work-in-progress tree）。这两个树可以看作是前后缓冲区的概念在 Fiber 架构中的应用。

具体的过程如下：

1. 初始渲染时，首先构建当前正在渲染的树（current tree）作为初始状态的表现。
2. 当进行更新时，React 会创建一个准备好的树（work-in-progress tree），该树与当前树保持一致的结构。
3. 在更新过程中，React 会以增量的方式遍历 Fiber 树的节点，根据变更计算出新的状态。
4. 在计算完成后，React 会将准备好的树（work-in-progress tree）与当前树（current tree）进行交换，将新的树作为当前树。

通过这种方式，React 实现了双缓存的概念，将当前渲染和更新过程分离开来，避免了渲染过程中的长时间阻塞，提高了用户界面的响应性能。

值得注意的是，React Fiber 架构中的双缓存与传统图形渲染中的双缓存有所不同。在传统图形渲染中，双缓存用于实现平滑的图像更新；而在 React Fiber 中，双缓存用于实现增量式的渲染和可中断的更新过程。这是一种在 React 内部用于优化和调度的机制，并不直接涉及到图像渲染的层面。

总结起来，React Fiber 架构中的双缓存是一种用于实现增量式的、可中断的和可恢复的更新过程的机制，通过当前树和准备好的树的交替使用，提高了 React 应用的性能和响应性能。

## 3.fiber构建替换过程

### (1) mount时

```react
function App() {
  const [num, add] = useState(0);
  return (
    <p onClick={() => add(num + 1)}>{num}</p>
  )
}

ReactDOM.render(<App/>, document.getElementById('root'));
```

1. 首次执行`ReactDOM.render`会创建`fiberRootNode`（源码中叫`fiberRoot`）和`rootFiber`。其中`fiberRootNode`是整个应用的根节点，`rootFiber`是`<App/>`所在组件树的根节点。

之所以要区分`fiberRootNode`与`rootFiber`，是因为在应用中我们可以多次调用`ReactDOM.render`渲染不同的组件树，他们会拥有不同的`rootFiber`。但是整个应用的根节点只有一个，那就是`fiberRootNode`。

`fiberRootNode`的`current`会指向当前页面上已渲染内容对应`Fiber树`，即`current Fiber树`。

![image-20230525221025297](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/image-20230525221025297.png)

```
fiberRootNode.current = rootFiber;
```

由于是首屏渲染，页面中还没有挂载任何`DOM`，所以`fiberRootNode.current`指向的`rootFiber`没有任何`子Fiber节点`（即`current Fiber树`为空）。

2. 接下来进入`render阶段`，根据组件返回的`JSX`在内存中依次创建`Fiber节点`并连接在一起构建`Fiber树`，被称为`workInProgress Fiber树`。（下图中右侧为内存中构建的树，左侧为页面显示的树）

在构建`workInProgress Fiber树`时会尝试复用`current Fiber树`中已有的`Fiber节点`内的属性，在`首屏渲染`时只有`rootFiber`存在对应的`current fiber`（即`rootFiber.alternate`）。

![image-20230525221212722](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/image-20230525221212722.png)

3. 图中右侧已构建完的`workInProgress Fiber树`在`commit阶段`渲染到页面。

此时`DOM`更新为右侧树对应的样子。`fiberRootNode`的`current`指针指向`workInProgress Fiber树`使其变为`current Fiber 树`。

![image-20230525221348034](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/image-20230525221348034.png)





### (2) update时

1. 接下来我们点击`p节点`触发状态改变，这会开启一次新的`render阶段`并构建一棵新的`workInProgress Fiber 树`。

![image-20230525221542344](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/image-20230525221542344.png)

和`mount`时一样，`workInProgress fiber`的创建可以复用`current Fiber树`对应的节点数据。

> 这个决定是否复用的过程就是Diff算法

2. `workInProgress Fiber 树`在`render阶段`完成构建后进入`commit阶段`渲染到页面上。渲染完毕后，`workInProgress Fiber 树`变为`current Fiber 树`。

![image-20230525221814028](https://cdn.jsdelivr.net/gh/HushanCode/DrawingBed/img/image-20230525221814028.png)















