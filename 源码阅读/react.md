# React源码

## 1 从使用开始

一个最原始的react组件定义是这样完成的：

````javascript
import React from 'react'
import ReactDOM from 'react-dom'

const app = React.createElement('div', { name:'root' }, 'Content...')
ReactDOM.render(app, document.getElementById('root'))
````

其中引入了两个对象：React和ReactDOM，其中React对象负责创建React实例节点元素，ReactDOM负责向DOM挂载创建的app节点

````json
// 打印app对象
{
	$$typeof: Symbol(react.element)
    key: null
    props: {title: "myDiv", children: "Div Content"}
    ref: null
    type: "div"
    _owner: null
    _store: {validated: false}
    _self: null
    _source: null
    __proto__: Object
}
````

### React.createElement

