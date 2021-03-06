# canvas-awesome-demos

## 安装依赖
```
npm install
```

## 运行服务
```
npm run server

open http://127.0.0.1:3030/menu.html
```

## 构建menu列表
```
npm run build
```


前言
这次我们要讨论的话题，从基础上说，是一个简单的话题，但是想要把它做周到，做全面，做深入，实际上是需要很多的工作，考虑很多的细节的。

这一次我们要说的话题就是有关于事件在Canvas上虚拟元素上的派发

最基本的情况
我们来说一个场景

有一个Canvas画布，画了一个矩形上去，现在要做的事情是，类似给这个矩形绑上事件，点击这个矩形内的区域就alert('click') 。矩形外的区域点击无响应。

那么该怎么做这件事情呢？
<canvas>也是个dom元素，它有dom原生的事件系统，这个没问题，但是在Canvas画布内部画上去的东西，没有独立的事件系统。只能依托于画布的事件来做这件事。也就是要判断鼠标点击的位置。

思路很直接：

获取到鼠标点击的位置相对于<canvas>画布的相对位置P0。
拿到画的这个矩形相对于画布的相对位置
最后直接判断点击点击的点是否落到这个矩形内就ok了。最终问题简化成点是否落在矩形内的问题。
复杂一些的情况
如果有多个矩形在画布内，可能相互都有交叠，每个矩形点击都应该出现不同的东西。当点的位置发现有多个矩形叠在一起的时候，我们期望的情况是只认为“鼠标位置的视觉最上层”的那个矩形点击有效。其他的都不应该触发。

怎么处理这种情况？
如果直接按照和上面类似的"面向过程"的方式来处理这件事情，会相当的困难。甚至在代码里都不能很好的区分哪个事件属于哪个矩形...
所以无疑，我们必定要引入在Canvas内“面向对象”的编码方式。在Flash AS 和传统DOM 的结构里，都默认集成了基础的对象系统。
AS里面有DisplayObject,
DOM里面可以直接操作HtmlElement
但是Canvas里面没有... 这需要我们自己做。
自己做“对象系统”，自己做“事件系统”。

常见的思路：

所有绘制出来的东西，不管是个矩形也好，图片也好，文字也好，都包装成一个对象，定义为sprite,或者layer
有了这个基础对象模型，在这个对象模型上做事件系统
假如我们已经做好了针对单个“sprite”对象绑定事件的功能。现在面对多个“sprite”对象绘制在一块画布上，并且可能会有重叠的情况，那么我们要处理的情况是要找出满足**“包含鼠标点击位置并且在视觉的最上层”**的那个矩形。
包含鼠标位置的所有矩形，这个容易拿到，遍历一遍对象就可以。
在视觉表现的最上层，在不设置特殊的globalCompositeOperation的情况下，其实就是最后绘制的元素。
那么最终在代码层面的逻辑简化为 “找到包含当前鼠标点击位置的所有矩形中的 最后绘制的那一个”
mouseover 和 mouseout的特殊处理
我们上面举得例子都是用的 点击click在举例。它的派发还相对容易些。而mouseover 和 mouseout 这种事件在 sprite这种虚拟元素上的 派发，会有其他的问题，我们来分析下场景：

所有虚拟对象sprite的事件派发，本质上都要经过<canvas>这个DomElement的事件来做，比如我在一个矩形的spriteA上绑定一个click事件，当满足触发条件的时候，触发事件回调 FuncA 。本质上是在<canvas>这个tag上绑定了一个'click' 事件，然后回调函数里面 做好了 spriteA 是否被点中 的判断逻辑， 再确定是否执行 spriteA 的回调 FuncA。
但是mouseover 和mouseout 不能用这种方式来处理。
因为<canvas>的 mouseover 和mouseout事件，只会在鼠标进入Canvas元素 和 离开Canvas元素的时候 触发一次，所以即使给 sprite 绑定了 mouseover 或者 mouseout 的事件，鼠标在<canvas> 元素里再怎么动，也不会再触发mouseover和mouseout。
那么怎么处理这种情况呢？该想到也应该想到了，在mousemove里面做处理...
理理思路：

在canvas元素的mousemove事件中，分析上一次move触发时鼠标位置下的元素列表ListA 和 当前move触发的 鼠标位置下的元素列表ListB。
假设ListA中 有 [SpriteA, SpriteB], ListB中有[SpriteA, SpriteB, SpriteC] 。那么就表示SpriteC是当前这次move刚进入的元素。
而且刚好SpriteC满足**“是鼠标位置下层叠元素上最后绘制的那一个”** 这个条件。那么表示SpriteC 的 mouseover 或者 mouseenter 触发条件满足。
mouseout 的触发判定也类似...
如果不是规整矩形，而是不规则多边形的sprite
上面我们说的情况都是基于规则矩形的情况来说的。如果不是一个规则的矩形，而是一个不规则的封闭多边形 或者 圆形。 这种情况下 sprite 的事件分发又该怎么处理呢？

其实本质上和上面说的规则的四边形的思路一致
只不过需要把 “鼠标是否落在矩形内” 的检测 变成 “鼠标是否落在这个不规则封闭多边形内”
然后重复上面的各种情况测试
还要考虑存在scale 和 rotate 的情况！！
事情还没有结束，上面所说的情况还不涉及到 存在 缩放 和 旋转的情况。

假设一个矩形的spriteA 以 它的中心为 旋转点 旋转了45度，那么这个时候鼠标 的位置 是否落在这个矩形中，判断中要变成“点是否落在一个旋转45度的矩形”中的问题。
同样，如果带缩放的元素做事件派发的判断。同样要判断的是 点是否落在 缩放变换后的矩形中。

解决方案：

在同一变换纬度内做判断。
transform变换最终可以转换成一个6维matrix矩阵，矩形的缩放或者旋转，其实都可以用矩阵变换来代替，那么，如果一个元素做了某种矩阵变换，为了保证判断的鼠标坐标 和 这个元素的变换纬度统一。可以让 鼠标这个点 也进行同样的矩阵变换。
然后在同样的变换纬度内重复上面讨论过的判断流程。
scale 和 rotate需要考虑的点
scale 和 rotate 一旦介入我们事件派发的考虑范畴。问题瞬间变复杂很多。

比如以下：

scale受影响的因素：
css中的transform:scale
canvas的context2d.scale()
canvas 的 style:width 和 style:height 和 canvas.width ,canvas.height 的比对.
比如

<canvas width="1000" height="600" style="width:500px; height:300px"></canvas>
等效于做了scale(0.5, 0.5) 的变换。

rotate受影响的因素：
css中的 transform:scale
canvas 的context2d.rotate
所以，如果想要完整Fix scale 和 rotate 以及其他transform变换对于事件派发的影响，上述的因素都必须考虑进去。

