---
layout: post
title: 基于 ant design table 实现的虚拟滚动列表
date: 2019-09-17 10:20:28
excerpt: 只是一种思路，不支持无限滚动且有很多未知 bug
lang: zh_CN
categories: 
- frontend
---

**TLDR;**

想直接看源码(带 demo) ，点[这里](https://github.com/yiyizym/infinite_table)；不要直接用在生产环境，保证会出现各种问题。

**正文**

不少用过 ant design table (下文简称 Table) 的朋友，可能都会遇到过当一页加载的数量超过 150 条时，Table 组件渲染慢且后续操作（如点击勾选框）反应慢的情况。

社区早就有人提出优化长列表渲染性能的 [issue](https://github.com/ant-design/ant-design/issues/3789) ，针对 Table 的性能优化，官方也计划于今年第四季度推出的 ant design 4.0 版本支持 [虚拟滚动](https://github.com/ant-design/ant-design/issues/16911)。

但是我有点担心进度能不能赶上，毕竟还有两个星期就到第四季度了，实现虚拟滚动的 Table 预览版都还没有出来。另一方面，因为用过 Table 的大部分功能，踩过不少坑，看过几个虚拟滚动的实现，也看过 rc-table 的源码，我觉得不从底层重新实现 Table （不再使用 table/thead/tbody 等等原生元素），很难解决虚拟滚动将要面临的问题。

长列表的性能问题，主要体现在两个方面：

- 初次渲染耗时长
- 滚动时有迟滞感

原因在于渲染长列表需要相应数量的 DOM 元素，DOM 元素多，渲染时定位、重绘的工作量就大，也会暴露很多平时不被注意的性能问题。

虚拟滚动，无论具体实现方式如何，它的目标都是只渲染用户在当前视窗能看到的列表条目，看不到的就不渲染，因此能解决长列表的性能问题。

虚拟滚动的实现方式，据我所知有两种：

一种是在列表的前、后各插入一个兄弟（占位）元素，令三者的高度相加刚好等于将所有列表条目渲染出来后的高度（这样滚动条才能正常显示），然后监听滚动事件、动态设置两个占位元素的高度，并更新列表的条目。

另一种是列表外层包一个容器，容器应用相对定位，它有一个应用绝对定位的大小为 1px X 1px 的子元素，这个子元素的 top 设置为所有列表条目渲染出来后的高度（撑开滚动条），列表可以应用相对定位或者是绝对定位，在容器上监听滚动事件，动态设置列表的 top ，以及更新列表的条目。

两种方式大同小异，社区有人用第一种方式实现 Table 的长列表性能优化，感兴趣的可以[看看](https://github.com/ctq123/ant-virtual-table/)；第二种方式的具体实现可以看[这里](https://developers.google.com/web/updates/2016/07/infinite-scroller)。

我希望能一次性加载所有数据，然后用虚拟滚动显示数据。找了好久，没有找到现成的轮子，工作所迫，得亲自动手造。下面是我的一些思路和实践。

首先我希望能尽量依靠现有的接口实现虚拟滚动。因为我已经应用上不少它的功能，找不到别的能实现相同功能的组件。最好能在 Table 的外面做一层封装，把虚拟滚动相关的代码都写在外面，让 Table 不知道自己处在虚拟滚动的现实里，但从其他轮子已有的问题看出，天下间没这么便宜的事。依靠现有接口一定程度的侵入内部实现是必要的。

Table 接受一个属性 components ，它允许我们传入自定义的 table / tr / td 。在一番尝试之后，发现可以使用自定义 table 来实现上面提到的第二种虚拟滚动。

实现功能并不难，主要是一些数学计算，只用了 100 多行代码。优化性能、解决一些怪异问题才是最花时间的。

性能优化一般的套路是，如果发现页面卡顿，打开开发者工具，切换到 Performance 标签页面，开始记录，重复导致卡顿的操作，停止记录。从时间线中找到帧率很低的部分，在 Summary 标签页中， chrome 会记录主线程的时间都花在哪些类型的任务上，展开 Main ，chrome 会在耗时较长的任务的右上角上标记一个红色三角形，告诉你这里可能有问题，点击这些任务，可以定位触发这些任务的代码。如下图：

![Performance]({{ site.url }}/assets/chrome_performance.jpg)*chrome_performance*

这次 Table 性能优化的两个方向是：

- 在 Scripting 方面，减少耗时过长的函数的执行次数；
- 在 Rendering 方面，去掉会触发 reflow 的代码。

第一个方向的优化也分两点：

- 对 Table 里出现的组件，尽量改用 React.memo 或者 React.PureComponent 减少不必要的 render ；
- 用 debounce 和 requestAnimationFrame 以及记录上次计算结果等等方法，减少滚动监听事件回调的执行次数

第二个方向的优化，因为 Table 有好几处内部代码会触发 reflow (具体见[源码](https://github.com/yiyizym/infinite_table)注释)，所以优化到最后会发现自己无能为力。

值得一提的是，如果你在 Table 内部使用了 ant design 的 Button 组件，你会发现这个 Button 组件（可能）会触发 reflow 。它内部使用了一个 `fixTwoCNChar` 方法（往两个汉字中间加一个空格），这个方法内部有这样一行代码：

```javascript
var buttonText = this.buttonNode.textContent || this.buttonNode.innerText;
```

`innerText` 跟 `textContent` 的区别是前者不会返回 `display: none` 的内容，因此需要知道 DOM 当前的最新状态，触发 reflow 。

`||` 是短路操作符，如果从 `textContent` 取到 truthy 的值，不会再去取 `innerText`。

问题是，如果连 `textContent` 都取不到内容， `innerText` 就更不会有内容了，我猜这样写的原因是解决兼容性问题，缺点是当 `textContent` 取到 falsy 的值，就会触发 reflow 。

官网有[文档](https://ant.design/components/button-cn/#如何移除两个汉字之间的空格？)说明如何去掉 `fixTwoCNChar` ，你也可以用一个空函数直接 override Button 组件的这个方法。

怪异的问题也解决了一些，比如有 fixed 列时，列/行错位；滚动监听事件的回调绑定了两次等等。这些问题都要深入 Table 的内部实现才能知道原因，解决办法也很取巧以及不可靠，就不多说了。对实现感兴趣的推荐先看[这里](https://juejin.im/post/5d593f1ef265da03ae7873b5)，再看源码。

目前发现唯一解决不了的问题就是有 fixed 列时，滚动停止后会再次向上跳动一段距离（从 1px 到 20 px 不等）。

问题原因定位到 Table 内部几处代码触发了 reflow ，继而触发 react 的 onScroll 事件回调。因为 reflow 是内部触发的， onScroll 也不是我能解绑/取消/劫持的，所以我什么也做不了。

以上，这只是一种不成熟的思路和实践，除了等官方的实现，好像没有尽善尽美的方法。抛砖引玉，希望对你有帮助。

[源码在这里](https://github.com/yiyizym/infinite_table)