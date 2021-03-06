---
layout: post
title: 浅谈CSS2.1可视化格式模型
excerpt: "对CSS2.1可视化格式模型的一些总结"
date: 2014-03-27 16:47:38
keywords: 可视化格式模型, 包含块, 盒子, 定位方案, 浮动, 常规流
lang: zh_CN
categories: frontend
---
近日看了CSS2.1规范文档中对可视化格式模型（visual formatting model）的描述，弄懂了很多之前理解模糊的概念。顺手写写总结，理理思路，以便日后忘记了可以回头看看。

网页上看得到的所有元素都是一个个大小不同、或相互嵌套、或上下堆叠、或水平排列的盒子。

可视化格式模型说的就是如何摆放这些盒子。盒子的布局与以下几个因素有关：

1. [文档树](http://www.w3.org/TR/CSS21/conform.html#doctree)中元素之间的关系；
2. 盒子的类型和尺寸；
3. 定位方案；
4. 其他外部信息（如视点、图片的本征尺寸等）

## 文档树中各元素的关系

文档树中各元素的关系无非就是祖先、同辈、后代。值得一说的是包含块（containing blocks）的概念。一个盒子在真正被定位时需要一个具体的大环境，盒子被包含在这个大环境之中，这个大环境就是此盒子的包含块，具体来说盒子的包含块是从盒子的某个祖先盒子中挑选出来的，不一定就是它的父亲盒子。因为盒子之内可以嵌套其他盒子，换句话说盒子可以有子孙后代，此盒子也可充当后代盒子的包含块。

## 盒子的类型、尺寸

盒子的类型细分起来有很多种，由 display 属性的值决定。常用的 display 属性值有：

- block
	此值可使元素产生块盒子
- inline
	此值可使元素产生一个或多个行内盒子
- inline-block
	此值可使元素产生一个行内级块容器。行内级块容器对包含在其内部的元素表现为块盒子，其自身——即对同辈元素及祖先元素而言——是一个原子行内级盒子。
- none
	此值可使元素不产生任何盒子，其本身、内容、后代元素在格式化结构中消失。同辈元素、祖先元素就当其根本不存在一样。

display 还有很多其他的取值，在此不作研究。

下面来说说盒子的两大类型：块盒子、行内盒子。

### 块盒子

在说块盒子前，要先说说块级元素。块级元素最显著的特征是其开头和结尾处都会另起一行。[这里](http://reference.sitepoint.com/html/block-level)罗列了所有的块级元素。块级元素会产生块级盒子（准确来说是原始块级盒子），这个块级盒子会充当包含在其内的元素的包含块。

块级盒子在多数情况下（除非它是表格盒子或元素本身是重置元素。重置元素指元素本身的尺寸和外形由外部信息决定，不由 CSS 决定。images/plugins/form 元素都属于重置元素。[此处](http://reference.sitepoint.com/css/replacedelements)有更详细的描述）也是块容器盒子。块级盒子与块容器盒子不是包含与被包含的关系，它们之间有重叠的部分，也有不重叠的部分。非重置行内块盒子和非重置表格单元是块容器盒子，但并不是块级盒子。既是块级盒子也是块容器盒子的可被称作：块盒子。

**块容器盒子要么只包含块级盒子，要么只包含行内级盒子。**这句话很重要。

#### 匿名块级盒子

	<div>
		Some text
    	<p>More text</p>
	</div>

假设 div 与 p 的 display 属性值均为 blcok 。实际如下图，div 标签生成一个块容器盒子，若块容器盒子内包含块级盒子（本例正是此种情况：p为块级盒子），则强制此块容器盒子只包含块级盒子。文本节点“Some text”此时将被一个匿名块级盒子包裹。

产生匿名块级盒子的情况是，当一个行内盒子包含一个流内（in-flow，流在这里指常规流）块级盒子时，这个行内盒子会被块级盒子腰斩成两个匿名块级盒子。由此形成的三个块级盒子是同辈的关系。匿名块级盒子还有很多细节，详见[文档](http://www.w3.org/TR/CSS21/visuren.html)。

### 行内盒子

与块级元素和块级盒子对应的是行内级元素和行内级盒子。

行内级元素与块级元素相反，它不会在开头和结尾处另起一行，[这里](https://developer.mozilla.org/en-US/docs/HTML/Inline_elements)罗列了所有的行内级元素。行内级盒子由行内级元素产生。

与块级盒子一样，并不是所有的行内级盒子都是行内盒子，由重置行内级元素、inline-block 元素、inline-table 元素产生的行内级盒子就不是行内盒子，这些盒子被称为原子行内级盒子。

#### 匿名行内盒子

也存在匿名行内盒子，见下面的例子。p 产生的块容器盒子内不包含块级盒子，却包含一个行内盒子（由 em 产生），此时将强制 p 产生的块容器盒子只包含行内盒子。文本节点“Some”和“text”将被一个匿名行内盒子包裹。匿名行内盒子还有很多细节，详见[文档](http://www.w3.org/TR/CSS21/visuren.html)。

	<p>Some<em>emphasized</em>text</p>


**盒子的尺寸**

盒子的尺寸并不受限与包含块，若比包含块的尺寸要大，盒子将会溢出。

## 定位方案

定位方案分以下三种：

1. 常规流；
2. 浮动；
3. 绝对定位；

### 常规流

只要元素没有被设置为浮动或绝对定位，元素就位于常规流之内（否则称之为脱离常规流），采用常规流定位。

位于常规流内的元素，其产生的盒子要么处于块格式化上下文要么处于行内格式化上下文。理所当然地，块级盒子参与构成块格式化上下文，行内级盒子参与构成行内格式化上下文。

**块格式化上下文**

浮动元素、绝对定位元素、不是块盒子的块容器（如 inline-block 等）、设置 overflow 属性值且值不为 visible 的块盒子都将会为其内容（内容是一个术语，对其准确解释见[这里](http://www.w3.org/TR/CSS21/conform.html#doctree)）产生一个新的块格式化上下文。

在块格式化上下文之中，所有盒子都将从包含块的顶部开始，从上到下垂直堆叠，它们之间的垂直间距由 margin 属性确定，两个相邻的块级盒子之间还将发生垂直边距的塌陷（collapse）。每个盒子的左外边缘将与包含块的左边缘（这里指内容的的左边缘）重合。

**行内格式化上下文**

在行内格式化上下文之中，盒子将从包含块的顶部开始，从左到右排布。盒子的垂直对齐方式有：顶部对齐、底部对齐、基线对齐。

这里还有一个重要的概念：行盒子（line box），行内盒子排成一行，这一行就形成行盒子。

行盒子的宽度由两点决定：包含块的宽度、浮动的存在与否。行盒子的高度是个比较复杂的问题，详见[此处](http://www.w3.org/TR/CSS21/visudet.html#line-height),行盒子的高度永远大于或等于其包含的盒子。

除去存在浮动的情况，行盒子的左边缘将与包含块的左边缘重合。

当构成行盒子的所有行内级盒子的总宽度比行盒子的宽度要短时，行内级盒子的水平排布方式将由行盒子的 text-align 属性决定。反过来，当一个行内盒子的宽度比行盒子的宽度要大，行内盒子将被分割成几个行内盒子摆放在几个行盒子当中，除非这个行内盒子被设置为不可分割的，这种情况下，行内盒子将会溢出行盒子。

行盒子的相关细节比较多，详见[文档](http://www.w3.org/TR/CSS21/visuren.html#positioning-scheme)。

### 浮动

只要元素设置 float 属性值且其不为 none ，元素就将会脱离常规流且尽可能向左（或右）移动。
（这里的内容指文本、行内元素）可能会排布在浮动的周围。

元素产生的浮动盒子将一直向左（或右）移动直到它的外边缘与包含块的边缘或者其他浮动的外边缘重合。如果此时存在同辈的行盒子，浮动将与行盒子的顶部垂直对齐。

如果当前的水平空间不足以容纳浮动，浮动将会下移到要么放得下它要么只有它一个浮动的地方。

**浮动盒子脱离常规流，所以块盒子在布局时视它为无物，但行盒子却受其影响，紧邻浮动的行盒子会排布在浮动的周围，前提是浮动的高度（outer height，算上padding、border、margin）是个正值，**这是文字围绕图片的原理。

浮动的外边距从不与相邻的其他盒子发生塌陷。

浮动可以覆盖（overlap）处于常规流中的其他块盒子。如果发生覆盖，浮动将呈现在块盒子的前面（前面指的是更接近用户），而在行内盒子的后面。

浮动的盒子除了尽可能向左（或右）移动之外，它还尽可能向上移动，当两者冲突时，它会选择尽量向上移动。

设置元素的 clear 属性，可使其左侧（或右侧，或两侧）不与浮动相邻。

浮动的还有不少细节，详见[文档](http://www.w3.org/TR/CSS21/visuren.html#positioning-scheme)。

### 绝对定位

设置元素的postion属性值为 absolute 或 fixed 可使元素采用绝对定位，此时元素脱离常规流。

### display、position和 float 的关系

1. 如果 display 设置为 none ，则 position 和 float 属性无论为何值都不会生效，元素不产生盒子；
2. 否则，当 position 设置为 absolute 或 fixed 时，float 属性的计算值为 none ，display 属性的计算值如下表。盒子的位置为包含块及 top/right/left/bottom 等值确定；
3. 否则，当 float 为非 none 的值时，其 display 属性的计算值如下表；
4. 否则，当元素为根元素时，display 属性的计算值如下表，有一种例外情况就是声明值为 list-item 其计算值可以是 block 或者 list-item ；
5. 否则，其他的 dispaly 属性值表现为其声明的值。


| specified value        | computed value
| ------------- |:-------------:| 
| inline-table     | table | 
| inline,inline-block,table-row-group,table-column,table-column-group,table-header-group,table-footer-group,table-row,table-cell,table-caption     | block      | 
| others | same as specified      | 


## 参考资料

Cascading Style Sheets Level 2 Revision 1 (CSS 2.1) Specification, [Visual formatting model](http://www.w3.org/TR/CSS21/visuren.html);

Github上面的一篇同主题[文章](https://github.com/sunnylost/sunnylost.github.com/blob/master/draft/css-visual-formatting-model.md);

## 后记

关于可视化格式模型，还有很多知识点没有谈及，比如视点，相对定位，绝对定位时包含块的选择，z-index，等等。写技术文章很辛苦，而且总感觉挂一漏万，不过写技术文章有且于内化知识，也算是一道苦口良药吧。
