# Fed-e-task-03-05

## 1、Vue 3.0 性能提升主要是通过哪几方面体现的？

* 响应式系统升级：使用 `Proxy` 对象重写响应式系统
  * 可以监听动态新增的属性
  * 可以监听删除的属性
  * 可以监听数组的索引和 `length` 属性
* 编译优化：标记和提升所有的静态根节点，`diff` 的时候只需要对比动态节点内容
  * 新引入了 `Fragments` 既**片段**特性，模板中不需要再创建一个唯一的根节点，模板中可以直接放文本内容或者许多统计的标签，需要升级 `vetur` 插件
  * 会通过标记和提升静态节点，然后通过 `Patch flag` 在 `diff` 时会跳过静态根节点，只需要去更新动态节点中的内容，大大提升了 `diff` 的性能
  * 通过缓存事件处理函数减少了不必要的更新操作
* 源码体积优化：
  * 移除了一些不常用的 `api`，例如：`inline-template`、`filter` 等，可以让最终的代码体积变小
  * 对 `Tree-shaking` 的支持更好，通过编译阶段的静态分析，找到没有引用的模块在打包的时候过滤掉，让打包后的体积更小

## 2、Vue 3.0 所采用的 Composition Api 与 Vue 2.x 使用的 Options Api 有什么区别？

**Options Api**
* 创建组件时会定义 `data`、`methods`、`props`、`computed` `生命周期函数` 等属性来描述组件，同一个功能逻辑的代码可能会涉及到这些不同的选项，导致组件逻辑难以阅读和理解
* 难以提取组件中可重用的逻辑

**Composition Api**
* 基于函数的 `api`
* 可以更灵活的组织组件的逻辑：查看某个逻辑时只需要关注具体的函数即可，当前的逻辑代码都封装在函数内部
* 有利对代码的提取和重用

## 3、Proxy 相对于 Object.defineProperty 有哪些优点？

* `Proxy` 可以直接监听对象而非属性
* `Proxy` 可以直接监听数组的变化
* `Proxy` 有多达 13 种拦截方法，`apply、ownKeys、deleteProperty、has` 等都是 `Object.defineProperty` 所不具备的
* `Proxy` 返回的是一个新对象，可以只操作返回的新对象达到目的，而 `Object.defineProperty` 只能遍历对象属性直接修改

## 4、Vue 3.0 在编译方面有哪些优化？

* 新引入了 `Fragments` 既**片段**特性，模板中不需要再创建一个唯一的根节点，模板中可以直接放文本内容或者许多统计的标签，需要升级 `vetur` 插件
* 会通过标记和提升静态节点，然后通过 `Patch flag` 在 `diff` 时会跳过静态根节点，只需要去更新动态节点中的内容，大大提升了 `diff` 的性能
* 通过缓存事件处理函数减少了不必要的更新操作

## 5、Vue.js 3.0 响应式系统的实现原理？

* `Proxy` 对象实现属性监听
* 多层属性嵌套，在访问属性过程中处理下一级属性
* 默认监听动态添加的属性
* 默认监听属性的删除操作
* 默认监听数组索引和 `length` 属性
* 可以作为单独的模块使用

1. `reactive`
   * 接收一个参数，首先会判断这个参数是否是对象，如果是不直接返回
   * 创建拦截器对象 `handler`，设置 `get/set/deleteProperty`
     * `get`：接收 => `target，key，receiver`
       * 收集依赖（`track`）
       * 返回 `target` 中对应的 `key` 的值：通过 `Reflect.get(target, key, receiver)` 来获取
         * 如果当前 `key` 属性对应的值也是对象，同样的位这个对象创建拦截器对象 `handler`，设置 `get/set/deleteProperty`
         * 如果则直接返回
     * `set`：接收 => `target，key，value，receiver`
       * 获取 `key` 属性的值：通过 `Reflect.get(target, key, receiver)` 来获取，用来判断传入的新值是否与旧值相等
         * 如果两值相等不需要做任何处理
         * 如果不等，调用 `Reflect.set(target, key, value, receiver)` 重新修改这个属性的值，并且触发更新（`trigger`）
     * `deleteProperty`：接收 => `target，key`
       * 首先判断 `target` 中是否有自己的 `key` 属性，如果有，成功删除这个 `key` 属性后再触发更新（`trigger`），最后返回删除是否成功
   * 创建并返回 `Proxy` 对象
2. `effect`
  * 接收一个函数作为参数 => callback
  * 首先执行一次 `callback()` 在函数中访问响应式对象的属性，在这个过程中去收集依赖，在收集依赖的过程中要把 `callback` 存储起来，在函数外部定义变量 `activeEffect` 保存，让之后的 `track` 函数能够访问到这里的 `callback`
3. `track`
  * 接收两个参数：`target、key`，在函数外部定义 `let targetMap = new WeakMap()`
  * 首先判断 `activeEffect`
    * 如果值为 `null` 说明当前没有要收集的依赖，所以直接返回
    * 否则的话在 `targetMap` 中根据当前的 `target` 来找 `depsMap`，因为当前的 `target` 就是 `targetMap` 当中的键
    * 判断是否找到了 `depsMap`，因为 `target` 可能没有收集过依赖，如果没有找到，那么要为当前的 `target` 创建一个对应的 `depsMap` 去存储对应的键 `key` 和对应的 `dep` 对象，把它添加到 `targetMap` 中，并且给 `depsMap` 赋值
    * 给 `dep` 添加 `activeEffect`
4. `trigger`
  * 接收两个参数：`target、key`
  * 根据 `target` 在 `targetMap` 中找到 `depsMap`
  * 判断是否找到 `depsMap`，如果没有找到，直接返回；如果找到了 `depsMap`：
    * 那么再根据 `key` 找到对应的 `dep` 集合（`depsMap.get(key)`）
    * 判断 `dep` 是否有值，有值则遍历 `dep` 集合，然后去执行里面的每一个 `effect` 函数