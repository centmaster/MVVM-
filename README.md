## Vue React San 框架间响应式数据的比较

起因：大漠喷了尤大神以后让我真的像看看不同框架间的区别到底在哪里。各自的优缺点。没有选择Angular而是选择了学长安利的百度框架San。让我们一起来看看吧！

### 先来看三个小demo吧

##### vue:<https://centmaster.github.io/MVVM-compare/show/vue/>

打开控制台，我们可以看到效果。其中unusedcount改变了father中对象的数据，但数据没有关联到视图上，所以页面没有被再次渲染。也可以看到父子节点之间的渲染各自独立。

##### react:<https://centmaster.github.io/MVVM-compare/show/react/>

对比vue，我们可以看到，react中父节点的渲染会出发子节点的渲染，而完全没有用到的数据改变也会使得页面渲染。子节点child2由于添加的shoudcomponentupdate为false，不会被渲染。

##### San:<https://centmaster.github.io/MVVM-compare/san-test/>

san中，我们可以看到，虽然san的数据操作与react更像 ，但也是精准渲染。（ps：这里我们没有使用双向数据绑定）

**vue**:[https://centmaster.github.io/MVVM-compare/show/vue/](https://centmaster.github.io/MVVM-compare/show/vue/)



### 三个框架的相应实现

#### Vue

#### 预备知识：

##### 1.Object.defineProperty(obj, prop,descriptor)

这是响应式监听的核心。我们把监听绑定在setter和getter上用来实现响应式。一下为更细节的介绍。

descriptor中定义的参数用来定义或修改的属性的描述符

`configurable` 当且仅当该属性的 configurable 为 true 时，该属性`描述符`才能够被改变，也能够被删除。

`enumerable`当且仅当该属性的 enumerable 为 true 时，该属性才能够出现在对象的枚举属性中。。

属性特性 `enumerable` 决定这个属性是否能被 `for...in` 循环或 `Object.keys` 方法遍历得到

`writable`当且仅当该属性的 writable 为 true 时，该属性才能被`[赋值运算符]`改变。

`value`该属性对应的值。可以是任何有效的 JavaScript 值（数值，对象，函数等）。

`get`一个给属性提供 getter 的方法，如果没有 getter 则为 `undefined`。该方法返回值被用作属性值。

`set`一个给属性提供 setter 的方法，如果没有 setter 则为 `undefined`。该方法将接受唯一参数，并将该参数的新值分配给该属性。

##### 2.虚拟dom

现在的vue也支持jsx的写法。vue有两种写法

```javascript
new Vue({
  data: {
    text: "before",
  },
  template: `
    <div>
      <span>text:</span> {{text}}
    </div>`
})

// render函数写法，类似react的jsx写法
new Vue({
  data: {
    text: "before",
  },
  render (h) {
    return (
      <div>
        <span>text:</span> {{text}}
      </div>
    )
  }
})
```

第一种为vue模版写法，第二种为jsx。无论二者哪种写法，最后都会被解析成这样：

```javascript
new Vue({
  data: {
    text: "before",
  },
  render(){
    return this.__h__('div', {}, [
      this.__h__('span', {}, [this.__toString__(this.text)])
    ])
  }
})
```

虚拟的dom节点长这个样子：

```javascript
function VNode(tag, data, children, text) {
  return {
    tag: tag, // html标签名
    data: data, // 包含诸如 class 和 style 这些标签上的属性
    children: children, // 子节点
    text: text // 文本节点
  }
}
```

完整的长这个样子：

```javascript
function VNode(tag, data, children, text) {
  return {
    tag: tag,
    data: data,
    children: children,
    text: text
  }
}

class Vue {
  constructor(options) {
    this.$options = options
    const vdom = this._update()
    console.log(vdom)
  }
  _update() {
    return this._render.call(this)
  }
  _render() {
    const vnode = this.$options.render.call(this)
    return vnode
  }
  __h__(tag, attr, children) {
    return VNode(tag, attr, children.map((child)=>{
      if(typeof child === 'string'){
        return VNode(undefined, undefined, undefined, child)
      }else{
        return child
      }
    }))
  }
  __toString__(val) {
    return val == null ? '' : typeof val === 'object' ? JSON.stringify(val, null, 2) : String(val);
  }
}

var demo = new Vue({
  el: '#demo',
  data: {
    text: "before",
  },
  render(){
    return this.__h__('div', {}, [
      this.__h__('span', {}, [this.__toString__(this.text)])
    ])
  }
})
```

#### 实现监听

##### 监听data下所有属性，实现响应式

```javascript
class Vue {
  constructor(options) {
    this.$options = options
    this._data = options.data
    observer(options.data, this._update.bind(this))
    this._update()
  }
  _update(){
    this.$options.render()
  }
}

function observer(obj) {
  Object.keys(obj).forEach((key) => {
    if (typeof obj[key] === 'object') {
      new observer(obj[key], cb)
    }
    defineReactive(obj, key, obj[key])
  })
}

function defineReactive(obj, key, val, cb) {
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: () => {
      console.log('你访问了' + key)
      return val
    },
    set: newVal => {
      if (newVal === val)
        return
      console.log('你设置了' + key)
      console.log('新的' + key + ' = ' + newVal)
      val = newVal
      cb()
    }
  })
}

var demo1 = new Vue({
  el: '#demo',
  data: {
    text: "before"
  },
  render(){
    console.log("我要render了")
  }
})
```

上式实现了对于修改对象的监听，但还不够好。demo._data.text这种写法未免太丑了吧。于是我们添加一层代理。（不懂的同学详见es6的proxy）

```javascript
_proxy(key) {
    const self = this
    Object.defineProperty(self, key, {
      configurable: true,
      enumerable: true,
      get: function proxyGetter() {
        return self._data[key]
      },
      set: function proxySetter(val) {
        self._data[key] = val
      }
    })
  }
```

然后在构造函数中加上这么一句话

```javascript
Object.keys(options.data).forEach(key => this._proxy(key))
```

到此为止，data已经变成了响应式。不过还没完！我们会发现还有一个问题可以优化。

```javascript
new Vue({
  template: `
    <div>
      <span>name:</span> {{name}}
    <div>`,
  data: {
    name: 'js',
    age: 24
  }
})

setTimeout(function(){
  demo.age = 25
}, 3000)
```

事实是，如果三秒之后修改了age属性 ，页面的name也会重新渲染。这很奇怪对不对。age这种完全不需要的东西就不需要渲染了啊。

##### 实现依赖收集，精确渲染

我们发现，当执行render函数渲染的时候，会对所有依赖的data进行获取操作。其他的元素没有依赖自然我们也就不用管。

我们需要subs这个数组来储存依赖的属性。然后在get中添加add方法，在setter中添加notify方法监听变化。

```javascript
class Dep {
  constructor() {
    this.subs = []
  }
  add(cb) {
    this.subs.push(cb)
  }
  notify() {
    console.log(this.subs)
    this.subs.forEach((cb) => cb())
  }
}

function defineReactive(obj, key, val, cb) {
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    // 省略
  })
}
```

另外我们还要用Dep.target来区分进入get的是不是依赖收集的get。

最后po上全部代码。

```javascript
function VNode(tag, data, children, text) {
  return {
    tag: tag,
    data: data,
    children: children,
    text: text
  }
}

class Vue {
  constructor(options) {
    this.$options = options
    this._data = options.data
    Object.keys(options.data).forEach(key => this._proxy(key))
    observer(options.data)
    const vdom = watch(this, this._render.bind(this), this._update.bind(this))
    console.log(vdom)
  }
  _proxy(key) {
    const self = this
    Object.defineProperty(self, key, {
      configurable: true,
      enumerable: true,
      get: function proxyGetter() {
        return self._data[key]
      },
      set: function proxySetter(val) {
        self._data[key] = val
      }
    })
  }
  _update() {
    console.log("我需要更新");
    const vdom = this._render.call(this)
    console.log(vdom);
  }
  _render() {
    return this.$options.render.call(this)
  }
  __h__(tag, attr, children) {
    return VNode(tag, attr, children.map((child) => {
      if (typeof child === 'string') {
        return VNode(undefined, undefined, undefined, child)
      } else {
        return child
      }
    }))
  }
  __toString__(val) {
    return val == null ? '' : typeof val === 'object' ? JSON.stringify(val, null, 2) : String(val);
  }
}

function observer(obj) {
  Object.keys(obj).forEach((key) => {
    if (typeof obj[key] === 'object') {
      new observer(obj[key])
    }
    defineReactive(obj, key, obj[key])
  })
}

function defineReactive(obj, key, val) {
  const dep = new Dep()
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: () => {
      if (Dep.target) {
        dep.add(Dep.target)
        Dep.target = null
      }
      console.log('你访问了' + key)
      return val
    },
    set: newVal => {
      if (newVal === val)
        return
      console.log('你设置了' + key)
      console.log('新的' + key + ' = ' + newVal)
      val = newVal
      dep.notify()
    }
  })
}

function watch(vm, exp, cb) {
  Dep.target = cb
  return exp()
}

class Dep {
  constructor() {
    this.subs = []
  }
  add(cb) {
    this.subs.push(cb)
  }
  notify() {
    this.subs.forEach((cb) => cb())
  }
}
Dep.target = null


var demo = new Vue({
  el: '#demo',
  data: {
    text: "before",
    test: {
      a: '1'
    },
    t: 1
  },
  render() {
    return this.__h__('div', {}, [
      this.__h__('span', {}, [this.__toString__(this.text)]),
      this.__h__('span', {}, [this.__toString__(this.test.a)])
    ])
  }
})
```

#### React





#### San







### 三者之间对比

在san的身上，时而看到vue的影子，时而看到react的影子。设计者们按照自己的对框架的理解拼装了san。

|                     |  vue   |          react           |             San             |
| ------------------- | :----: | :----------------------: | :-------------------------: |
| 数据操作（set/get/Array) | 直接操作数据 | setState／this.state.data | this.data.set／this.data.get |
|                     |        |                          |                             |
|                     |        |                          |                             |
|                     |        |                          |                             |
|                     |        |                          |                             |