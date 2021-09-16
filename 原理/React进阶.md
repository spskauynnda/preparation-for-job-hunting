[toc]

## React基础

### JSX

#### 一. jsx变成什么

1. babel编译

   ReactComponent -> React.createElement

````javascript
React.createElement(  type,  [props],  [...children] )
````

2. createElement 

   element -> react element

3. 调和

    react element -> fiber

#### 2. render过程改造

1. 第 1 步：扁平化，规范化 children 数组。

   **React.Children.toArray** 

   ```js
   const flatChildren = React.Children.toArray(children)
   ```

2. 第 2 步：遍历 children ，验证 React.element 元素节点，除去文本节点。

   **React.Children.forEach**

   ```js
   const newChildren :any= []
   React.Children.forEach(flatChildren,(item)=>{
       if(React.isValidElement(item)) newChildren.push(item)
   })
   ```

3. 第 3 步：用 React.createElement ，插入到 children 最后

   **React.createElement**

   ````js
    /* 第三步，插入新的节点 */
   const lastChildren = React.createElement(`div`,{ className :'last' } ,`say goodbye`)
   newChildren.push(lastChildren)
   ````

4. 第 4 步: 已经修改了 children，现在做的是，通过 cloneElement 创建新的容器元素。

   **React.cloneElement**

   ````js
   /* 第 4 步：修改容器节点 */
   const newReactElement =  React.cloneElement(reactElement,{} ,...newChildren )
   ````











## React原理

### React核心原理——事件原理

#### 一、事件系统八问

* 为什么要有自己的事件系统

  根源：不同端的不同浏览器，事件系统**兼容性**不同。所以要实现一个兼容全浏览器的框架，就需要创建一个兼容全浏览器的事件系统

  * 为什么给元素绑定的事件、冒泡捕获机制、事件源都重写了？

    因为react不是绑定在真实的DOM上，所以需要模拟一套事件流

* 什么是事件合成

* 怎么实现批量更新

* 怎么实现模拟冒泡和捕获

* 怎么通过dom元素找到与之匹配的fiber

  事件绑定时，绑定的dispatchEvent事件会传入对应的真实事件源元素，通过dom元素找到fiber

* 为什么不能用return false 来阻止事件的默认行为

* 事件绑定在哪里，事件是如何完成绑定的

* V17中事件系统有何改变

  v17之前：事件绑定在document上；v17之后：事件绑定在应用对应的容器container上（这样更利于一个 html 下存在多个应用--微前端）

#### 二、事件流

**1. 冒泡及捕获**

冒泡方法：onClick、onChange...

捕获方法：onClickCapture、onChangeCapture...

**2. 阻止冒泡** 

`e.stopPropagation()`

````
export default function Index(){
    const handleClick=(e)=> {
        e.stopPropagation() /* 阻止事件冒泡，handleFatherClick 事件将不再触发 */
    }
    const handleFatherClick=()=> console.log('冒泡到父级')
    return <div onClick={ handleFatherClick } >
        <div onClick={ handleClick } >点击</div>
    </div>
}
````

结果：父元素的onClick事件将不再被冒泡触发

**3. 阻止默认行为**

先看原生事件，在原生事件中可以通过`e.preventDefault()`和`return false`来阻止事件默认行为（例如文本的拖动选择、图片的拖拽等）

在React中，重写了事件流，`return false`不再有效，而`e.preventDefault()`也和原生的不一样

#### 三、事件合成

* 事件不绑定在元素上，而是挂在在document或container元素上

* 不会一次性绑定所有事件，而是代码写了什么才绑定什么，但不一定是一对一的绑定关系，比如onClick会绑定click事件，而**onChange会绑定`[blur, change, focus, keydown, keyup]`多个事件**

**事件插件机制**

定义了react事件与js事件的一对多关系 （为什么要一对多？）

````js
// 1. registrationNameModules
const registrationNameModules = {
    onBlur: SimpleEventPlugin,
    onClick: SimpleEventPlugin,
    onClickCapture: SimpleEventPlugin,
    onChange: ChangeEventPlugin,
    onChangeCapture: ChangeEventPlugin,
    onMouseEnter: EnterLeaveEventPlugin,
    onMouseLeave: EnterLeaveEventPlugin,
    ...
}
````

`````js
// 2. registrationNameDependencies
{
    onBlur: ['blur'],
    onClick: ['click'],
    onClickCapture: ['click'],
    onChange: ['blur', 'change', 'click', 'focus', 'input', 'keydown', 'keyup', 'selectionchange'],
    onMouseEnter: ['mouseout', 'mouseover'],
    onMouseLeave: ['mouseout', 'mouseover'],
    ...
}
`````

https://zhuanlan.zhihu.com/p/165099741

总结：

1. ReactDOM 在初始化的时候，通过 EventPluginHub 对象加载事件插件，事件插件的加载是要遵循一定的顺序的，这个顺序与合成事件的依赖有关。
2. 每个插件负责不同类型的事件，但是插件的数据结构是一致的，首先，插件会声明负责的事件，以及这些事件的基本配置，最终要的是，插件提供了一个方法，通过该方法可以将原生事件包装成 React 合成事件。
3. React 的如果想要事件在捕获阶段触发，只需要在事件监听后加 Capture 就可以了，onClick -》 onClickCapture。
4. React 事件将具体的实现以插件的形式由各个平台（brower 和 native）提供（比如我们今天分析的插件就是由 ReactDOM 包提供），向上暴露为统一的 React 合成事件，这种类似于接口的设计方式，使得 React 事件实现了跨平台和跨设备。

#### 四、事件绑定

**事件存储**

handleChange、handleClick等事件，存储在了哪里？

——存在了fiber对象的memoizedProps属性中

**事件监听注册**

步骤1：在 diff props阶段（diffProperties），调用legacyListenToEvent

步骤2：在legacyListenToEvent中，依靠registrationNameDependencies对象，遍历React合成事件对应的js事件，对其进行事件绑定：

````
const listener = dispatchEvent.bind(null,'click',eventSystemFlags,document) 
/* TODO: 重要, 这里进行真正的事件绑定。*/
document.addEventListener('click',listener,false)
````

可以发现，并没有直接将事件`handleClick`本身绑定到document上，而是绑定了一个传入事件类型`click`参数的`dispatchEvent`函数

#### 五、事件触发

**1. 批量更新**

(原理待补充)

**2. 合成事件源**

接下来会通过 onClick 找到对应的处理插件 SimpleEventPlugin ，合成新的事件源 e ，里面包含了 preventDefault 和 stopPropagation 等方法。

**3. 形成事件执行队列**

在第一步通过原生 DOM 获取到对应的 fiber ，接着会从这个 fiber 向上遍历，遇到元素类型 fiber ，就会收集事件，用一个数组`dispatchListeners`收集事件：

- 如果遇到捕获阶段事件 onClickCapture ，就会 unshift 放在数组前面。以此模拟事件捕获阶段。
- 如果遇到冒泡阶段事件 onClick ，就会 push 到数组后面，模拟事件冒泡阶段。
- 一直收集到最顶端 app ，形成执行队列，在接下来阶段，依次执行队列里面的函数。

**React如何模拟阻止事件冒泡**

````js
function runEventsInBatch(){
    const dispatchListeners = event._dispatchListeners;
    if (Array.isArray(dispatchListeners)) {
    for (let i = 0; i < dispatchListeners.length; i++) {
      if (event.isPropagationStopped()) { /* 判断是否已经阻止事件冒泡 */
        break;
      }    
      dispatchListeners[i](event) /* 执行真正的处理函数 及handleClick1... */
    }
  }
}
````





## React生态

### React-router

#### 一、路由原理

**history、React-router、React-router-dom 三者关系**

- **history：** history 是整个 React-router 的核心，里面包括两种路由模式下改变路由的方法，和监听路由变化方法等。
- **react-router：\**既然有了 history 路由监听/改变的核心，那么需要\**调度组件**负责派发这些路由的更新，也需要**容器组件**通过路由更新，来渲染视图。所以说 React-router 在 history 核心基础上，增加了 Router ，Switch ，Route 等组件来处理视图渲染。
- **react-router-dom：** 在 react-router 基础上，增加了一些 UI 层面的拓展比如 Link ，NavLink 。以及两种模式的根部路由 BrowserRouter ，HashRouter 。

**两种路由主要方式**

- history 模式下：`http://www.xxx.com/home`

- hash 模式下： `http://www.xxx.com/#/home`

  对于 BrowserRouter 或者是 HashRouter，实际上原理很简单，就是React-Router-dom 根据 history 提供的 createBrowserHistory 或者 createHashHistory 创建出不同的 history 对象











## todos

- [ ] Promise

  - [ ] 基本api
  - [ ] 异常捕获
  - [ ] 链式调用原理
  - [ ] 常见应用题型
- [ ] React

  - [ ] 基础

    - [ ] jsx
    - [ ] class、function Component
    - [ ] state更新机制（单向数据流？）setState和useState区别
    - [ ] props理解
    - [ ] 组件生命周期、useEffect和useLayoutEffect是如何替代生命周期的？
    - [ ] ref

  - [ ] 优化

    - [ ] 渲染调优

  - [ ] React生态

    - [ ] react-router原理
    - [ ] react-redux
    - [ ] dva原理

  - [ ] React设计模式

    - [ ] 组合模式
    - [ ] render props 模式
    - [ ] HOC | 装饰器模式
    - [ ] 提供者模式
    - [ ] 自定义 hooks 模式

  - [ ] React核心原理

    - [x] 事件原理
    - [ ] 调和原理
    - [ ] 调度原理
    - [ ] hooks
    - [ ] diff

  - [ ] 实战

    * 实现表单系统
    * 实现状态管理工具
    * 实现路由功能
    * 自定义 hooks 实践
- [ ] Webpack

  - [ ] 
- [ ] 简历相关

  - [ ] 组件库治理
  - [ ] Promise：race（设置最大等待时长）finally（设置终态的loading兜底）链式调用
  - [ ] element-ui源码
- [ ] JS基础
- [ ] 计算机网络
  - [ ] 





https://blog.csdn.net/weixin_32183427/article/details/112179628

https://juejin.cn/book/6945998773818490884/section/6959723748450631694