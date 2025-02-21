---
title: GPU渲染架构与优化技术
source: https://www.cnblogs.com/wujianming-110117/p/17695021.html
author: 
published: 2023-09-12
created: 2025-02-21
description: 
tags:
  - clippings
---
**GPU渲染架构与优化技术（续）**

5.1. 渲染架构以及GPU优化技巧

5.1.1 GPU图渲染前言

目前所有的基本采用平铺渲染（基于图块的GPU架构，简称为TBR）渲染主流的渲染架构。这里主要介绍介绍TBR的优缺点。它还将Arm Mali基于图块的GPU架构设计与通常在台式机或控制台中发现的更传统的即时模式GPU进行了比较。  
    Mali GPU使用基于图块的渲染架构。这意味着GPU将输出帧缓冲区渲染为几个不同的较小子区域，称为图块。然后，它会在完成后将每个图块写出到内存中。使用Mali GPU，这些图块很小，每个图块仅16x16像素。

5.1.2. 常用的两种GPU渲染架构

目前常用的两种GPU渲染架构是平铺渲染（基于图块的GPU架构），主要用于，还有一种是Immediate Mode GPUs（简称IMR, 即时模式架构）传统的台式机GPU架构。

5.1.3. 即时模式渲染

1.简单介绍

简称IMR, 也就是全屏，因为它不去分模块，传统的台式机GPU架构通常称为即时模式架构。即时模式GPU将渲染处理为严格的命令流，在每个绘图调用中的每个图元上依次执行顶点和片段着色器。

伪代码如下：

```verilog
 draw in renderPass
 primitive in draw
 vertex in primitive
vertex
 primitive not culled
 fragment in primitive
fragment
```

2.优点

硬件数据流和内存交互图：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041624824-1395319442.png)

图上可以看出，顶点着色器以及其他与几何相关的着色器的输出可以保留在GPU内部

的芯片，着色器的输出可以存储在FIFO缓冲区中，直到流水线的下一个阶段准备使用数据

为止，GPU很少使用外部存储器带宽来存储和检索中间几何结果。（备注：DDR为数据

流， FIFO：First In First Out队列）。

IMR的优势分析图：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041338885-1516664181.png)

IMR的优势是每个图元直接提交渲染，管道没有中断，渲染速度快，管道并行起来时，每个光栅核心只要负责渲染器分给它的图元即可，无需其他控制逻辑，只需在像素渲染器后，对光栅出的像素做个排序。

3.缺点

1）如果有很大的图形（主要是三角形）需要被渲染，那帧缓存就会很大，比如对于整个屏幕的颜色渲染或者深度渲染就会消耗很多存储资源，但是片上是没有这么多资源的，因此就要频繁读取DDR。很多和当前帧有关的操作( 比如混合, 深度测试或者模版测试)都需要读取这个工作装置，存储器上的带宽负载可能会非常高，并且这样能耗也很高，对于来说，这种方式很不利于设备运行；  
2）z test跟混合都要频繁从帧缓存里读数据，毕竟帧缓存是位于内存上，带宽压力和功耗自然高；  
3）Overdraw的问题，比如应用在一帧里先画了棵树，然后画了面墙刚好遮住了树，在IMR下树仍然要在像素着色器里采样纹理，而纹理也是放在内存，访存功耗大。

5.1.4. 基于平铺的渲染

1.说明

基于图块渲染也称基于瓦片渲染或基于小方块渲染，它是一种在光学空间中通过规则的网格细分计算机图形图像并分别渲染网格（grid）或图块（tile）各部分的过程。这种设计的优点在于，与立即绘制整个帧的立即模式渲染系统相比，它减少了对内存和带宽的消耗。这使图块渲染系统的使用特别常见于低功耗硬件设备。图块渲染有时也被称为中置排序（sort middle）架构，因为它在绘图流水线中间，而不是接近结束时，进行几何排序。

2.以Mali GPU为例

Mali GPU采用不同的方法来处理渲染过程，这就是所谓的基于图块的渲染方法。此方法旨在最大程度地减少片段着色期间，GPU需要访问的外部存储器的数量。  
    基于图块的渲染将屏幕分成小块，并对每个小图块进行着色，直到将其写出到内存中为止。为了使这项工作有效，GPU必须预先知道哪些几何图形有助于每个图块。因此，基于图块的渲染器，将每个渲染过程分为两个处理过程：  
1）第一遍执行所有与几何相关的处理，并生成图块列表数据结构，该结构指示哪些图元对每个屏幕图块起作用。  
2）第二遍将逐块执行所有片段处理，并在完成后将切片写回到内存中。请注意，Mali GPU渲染16x16的图块。

伪代码如下：

```undefined
# Pass one
```
```verilog
 draw in renderPass
 primitive in draw
 vertex in primitive
vertex
 primitive not culled
primitive
```
```verilog
 tile in renderPass
 primitive in tile
 fragment in primitive
fragment
```

下图显示了硬件数据流以及与内存的交互：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041316322-1496336448.png)

优势：解决了传统模型的带宽问题，因为碎片着色器，每次都是读取一个小块放在片上，不需要频繁读取内存，直到最后操作完成，再写入内存。甚至还能够通过压缩平铺的方法，进一步减少对于内存的读写。另外，在图像有一些区域固定不动的时候，通过调用函数判断平铺是否相同，减少重复的渲染。

3.优势

下图显示了TBR的优势：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041218263-1919528874.png)

对于IMR所有read z/帧缓存，到了TBR通通不需要。TBR只需渲染完tile后把on-chip的pixel写到frame buffer(不需要写z，因为下一帧不需要用到前一帧的z和color)。这个好处在于TBR将Screen 平铺。这样，每次渲染的区域变小，小到可以把z/帧缓存搬到on-chip，快，省电。

另外另外两点优势是：  
1）TBR给消除Overdraw提供了机会，PowerVR用了HSR技术，Mali用了前向像素消除技术，目标一样，就是要最大限度减少被遮挡pixel的纹理和着色。  
2）TBR主要是缓存友好, 在cache里头的速度要比全局内存的速度快的多，以及有可能降低渲染率的代价，降低带宽，省电。

4.缺点

1）这个操作需要在vertex阶段之后，将输出的几何数据写入到DDR，然后才被碎片着色器读取。这之间也就是tile写入DDR的开销和碎片着色器渲染读取DDR开销的平衡。另外还有一些操作(比如tessellation)也不适用于TBR；  
2）如果某些三角形叠加在数个图块（Overdraw），则需要绘制数次。这意味着总渲染时间将高于即时渲染模式。

5.1.6. 两种渲染架构对比

下图显示了两种渲染架构的对比：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041154582-1053825233.png)

说明：

1）IMR的管道畅通无干扰，排序简单，TBR的排序较复杂，但也给低功耗优化提供了灵活的选择；  
2）几何图的transform和场景的平铺，然后往内存里写入几何图的数据和每个tile所要渲染的几何图，相对来说多了内存消耗；  
3）PC屏幕大，PC game场景复杂，对Tile list压力大，另外PC追求frame rate，所以很少用TBR，即使用了，遇到复杂游戏场景估计会切换到IMR。

5.2. 平铺和全屏方式的光栅化相比有什么优劣？

早期的渲染方式都是IMR(Immediate Mode 渲染，也就是Full Screen，因为它不去分Tile)，**IMR的优势**是每个primitive直接提交渲染，管道没有中断，渲染速度快，管道并行起来时，每个Raster core只要负责渲染分给它的[primitive](https://www.zhihu.com/search?q=primitive&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)即可，无需其他控制逻辑，只需在像素着色器后对Raster出的pixel做个排序：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041113212-612832912.png)

**IMR****的劣势**在于带宽压力和功耗较大：

1. Z测试与混合都要频繁从帧缓存里读数据，毕竟帧缓存是位于内存上，带宽压力和功耗自然高；
2. Overdraw的问题，比如应用在一帧里先画了棵树，然后画了面墙刚好遮住了树，在IMR下树仍然要在像素着色器里采样纹理，而纹理也是放在内存，访存功耗大。

正因为这种劣势，许多Mobile GPU转向TBR(Tile Based 渲染)，比如Imagination家的PowerVR，Arm家的Mali，Qualcomm家的Adreno（从AMD的Imageon收过来的），其实PC也有过尝试TBR，但最终或失败或取消，如微软的Talisman, PowerVR的Kyro，Intel的Larrabee都失败了，Nvidia的PC GPU Maxwell据说用了TBR做优化（但NV的mobile GPU tegra是IMR的，好吧）：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041053471-249000627.png)

为什么mobile GPU要转向TBR呢，因为**TBR**给**解决带宽功耗大**的两个源头提供了机会：

1. 对于IMR所有**read z/**[**帧缓存**](https://www.zhihu.com/search?q=framebuffer&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)**，到了TBR通通不需要**。TBR只需渲染完tile后把on-chip的pixel写到[frame buffer](https://www.zhihu.com/search?q=frame%20buffer&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)(不需要写z，因为下一帧不需要用到前一帧的z和color)。  
这个好处在于TBR将Screen 平铺。这样，每次渲染的区域变小，小到可以把z/帧缓存搬到on-chip，快，省电。  
**Tiled****也意味着Deferred:要延迟到整个场景的primitive都收到后才能开始Raster**。为什么？试想，刚拿到整个场景一半的primitive就开始Raster了，那么[渲染](https://www.zhihu.com/search?q=render&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)结束后z buffer就必须写回帧缓存，然后另一半的primitive开始raster时还必须把z/帧缓存从内存读回来，这样一来就大打折扣了。
1. **TBR****给消除Overdraw提供了机会**，PowerVR用了HSR技术，Mali用了前向像素消除技术，目标一样，就是要最大限度减少被遮挡pixel的[纹理](https://www.zhihu.com/search?q=texturing&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)和着色，具体见后文。

Tiled要求Defer，把管道提前打断，从parallel 渲染的角度看，IMR跟TBR是Sort Last和Sort Middle的区别：

Sort Middle:

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912041014398-54941797.png)

 Sort Last:

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040957794-1470937029.png)

但凡并行渲染，都希望[vertex](https://www.zhihu.com/search?q=vertex&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)直接找IDLE的shader，[raster](https://www.zhihu.com/search?q=raster&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)等资源执行，吐出数据，每个硬件资源之间不用互相通信，结果不需要统筹，但Graphics API的渲染是有顺序的，例如混合时Triangle的顺序决定[混合 pixel](https://www.zhihu.com/search?q=blending%20pixel&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)的先后，而并行的渲染快慢不易，最终必须有个阶段做个排序（Sort），用IMR的话，是到了[pixel 着色](https://www.zhihu.com/search?q=pixel%20shading&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)后才sort，简单；用TBR的话，是在几何图变化后，在Raster前做Sort，复杂，但有优化空间。

再说说TBR的劣势，比较下IMR和TBR两者的管道内存访问：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040914708-513381491.png)

TBR的管道被分成两部分：

1）第一部分处理几何图的transform和场景的平铺，然后往内存里写入几何图的数据和每个tile所要渲染的几何图，好吧，跟IMR比起来多了内存的开销，读写，这个是Trade off，没有绝对好坏，总之说是机会，优化做得好就赚。例如Tile Size就是个Trade off点，大Tile意味着更少的Tile，重复setup的primitive（一个primitive覆盖多个tile）更少，但也意味着每个tile有更多的triangle，[on-chip buffer](https://www.zhihu.com/search?q=on-chip%20buffer&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)更大。

不过Tile list需要把每个draw的state info和所有primitive数据都保存起来，场景大的时候内存会overflow，overflow的问题可以优化，比如选择一部分tile(PowerVR的[macro tile](https://www.zhihu.com/search?q=macro%20tile&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D))做渲染(这时需要读写内存上的[z/帧缓存](https://www.zhihu.com/search?q=z%2Fframebuffer&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)，牺牲下bandwidth和power，没办法）然后释放这部分tile的内存。

PC屏幕大，PC game场景复杂，对Tile list压力大，另外PC追求frame rate，所以很少用TBR，即使用了，遇到复杂游戏场景估计会切换到IMR。

TBR的管道被分成两部分：  
1）第一部分处理几何图的transform和场景的平铺，然后往内存里写入几何图的数据和每个tile所要渲染的几何图，好吧，跟IMR比起来多了内存的开销，读写，这个是Trade off，没有绝对好坏，总之说是机会，优化做得好就赚。例如Tile Size就是个Trade off点，大Tile意味着更少的Tile，重复setup的primitive（一个primitive覆盖多个tile）更少，但也意味着每个tile有更多的triangle，[on-chip buffer](https://www.zhihu.com/search?q=on-chip%20buffer&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)更大。

不过Tile list需要把每个draw的state info和所有primitive数据都保存起来，场景大的时候内存会overflow，overflow的问题可以优化，比如选择一部分tile(PowerVR的[macro tile](https://www.zhihu.com/search?q=macro%20tile&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D))做渲染(这时需要读写内存上的[z/帧缓存](https://www.zhihu.com/search?q=z%2Fframebuffer&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)，牺牲下bandwidth和power，没办法）然后释放这部分tile的内存。

PC屏幕大，PC game场景复杂，对Tile list压力大，另外PC追求frame rate，所以很少用TBR，即使用了，遇到复杂游戏场景估计会切换到IMR。

2）第二部分是[tile raster](https://www.zhihu.com/search?q=tile%20raster&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)，**HSR**跟**Forward Pixel Kill**就是在这个阶段做优化。

PowerVR的整体架构是这样子的（Imagination官网图片）：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040853931-1242484825.png)

**HSR**对完成覆盖每个Tile的每个primitive的每个pixel做z test，最终保留最近的pixel（如果有混合，还需要保留透明半透明的pixel），最终每个pixel [location](https://www.zhihu.com/search?q=location&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)只有一个pixel进着色（如果无混合的话）。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040810761-1478464540.png)

TBDR W/HSR。

HSR=去除隐藏表面。

对每个投影射线中的所有目标进行排序。

使用平铺缩小数据集大小。

只需要绘制最近的不透明和更近的透明目标对象。

剩余片段可以被杀死->不被提取。

**前向像素消除**会让原始封面到的每个像素，都进着色器线程（准确是quad，因为像素着色器是以quad为单位的），Mali用FPK逻辑和FPK缓存完成前向像素消除，其输入为每个pass z test的[quad](https://www.zhihu.com/search?q=quad&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)（意味着每个input的quad是已收到的，对应同一位置的所有quad中距离眼睛最近）。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040750720-704443147.png)

- 如果Raster新产生的[quad pass test](https://www.zhihu.com/search?q=quad%20pass%20test&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)，并且quad的4个pixel被fully covered，那就把与该quad具有相同位置的，更早的（意味着更远）那些thread全终止（它们可能还在FPK Buffer或已经在近碎片着色器了）。
- 另外，当quad被两个较近triangle组合起来cover到时，较远的triangle对应该位置的quad也不需要做着色。因此为进一步优化，Mali保存了整个tile所有Quad最近一次的coverage，如果FPK新近的quad不是full covered，但与该quad最近的一次coverage相或后是full coverage，则类似1），要把更早的thread全终止，即发出[kill信号](https://www.zhihu.com/search?q=kill%E4%BF%A1%E5%8F%B7&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040717624-548349497.png)

另外，**TBR和TBDR是两个很容易被混淆的概念，因为各家厂商用的术语不一样**，

其实在ARM看来，TBR延迟了渲染（第一个阶段的整个场景被平铺后），所以他家认为TBR跟TBDR（基于平铺的延迟渲染）是同一个概念；  
    而在Imagination看来，PowerVR的HSR把纹理和着色也延迟了（剔除不可见pixle之后），它家认为TBR+HSR才是真正意义的TBDR。

所以可以看出，IMR的管道畅通无干扰，排序简单，TBR的排序较复杂，但也给低功耗优化提供了灵活的选择。另外TBR 管道的分割让管道中断了，各种defer，跟IMR比起来，速度也可能会进一步被影响而变慢。

**总结一下，TBR用增大内存资源，以及（有可能）降低**[**渲染率**](https://www.zhihu.com/search?q=render%20rate&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A136096531%7D)**的代价，获得降低bandwidth，power的效益。**

【Metal2研发笔录（二）：传统延迟渲染和TBDR】

\[toc\]

二、延迟渲染原理回顾

延迟渲染相比于前向渲染可以更加高效的渲染大量的光源场景。前向渲染中，对于场景中通过深度测试的每个物体，要依次针对每个光源进行光照计算，当场景复杂、光源数量增多，计算量会急剧增加，效率低下；

而在延迟渲染中，光照计算推迟到第二步，对于每个光源场景在屏幕空间只进行一次光照计算，光源的增加对计算量影响线性的。

延迟渲染的实现方式目前依托不同的硬件结构有两种，像macOS等PC平台由于GPU是IMR（immediate mode 渲染）架构，延迟渲染的实现至少需要两个Pass。而iOS等平台的GPU支持TBDR架构，利用tile 内存可以实现在一个Pass中进行延迟渲染，减少CPU和GPU之间的数据带宽，提高了渲染效率。

2.1 传统延迟渲染

传统的延迟渲染一般分成两个步骤（2个Pass）：

- ***第一个Pass：渲染G-缓存。***

第一个Pass正常渲染一遍场景，经过顶点着色器模型坐标变换，和片段着色器，计算色彩（difusse）、法线（normal）、高光（specular）、深度（depth）、阴影（shadow）等，并把结果通过MRT缓存到内存中备用。

- ***第二个Pass：延迟光照计算以及颜色合成。***

第一个Pass缓存的G-缓存贴图会从CPU中传进第二个Pass进行进一步的绘制，第二个Pass中会利用G-缓存贴图中的数据重构每个片段的位置信息进行每个光源的光照计算。最后结果会叠加光照的计算结果和阴影等输出最终的像素颜色。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040627460-1274747899.png)

2.2 单Pass延迟渲染（iOS、tvOS GPUs）

基于TBDR的GPU架构，可以实现将渲染出的G-缓存保存在tile 内存中，不需要再写入到system 内存中，就避免了将G-缓存从GPU写入到CPU然后第二个Pass GPU又从CPU读取G-缓存的步骤，降低了CPU和GPU之间的带宽消耗。

Metal中控制GPU是否将tile 内存中的贴图数据写入到CPU的系统内存的方式是配置renderCommandEncoder的storeAction和纹理贴图的storageMode。loadAction是用来配置渲染开始时是否清空RT等动作。storeAction是用来配置渲染结束是否将渲染 pass的结果保存到attachment中等动作。

***几种常用storgeMode的含义：***

- MTLStorageModeShared：表示资源保存在system 内存，且CPU和GPU都可以访问；
- MTLStorageModePrivate：表示资源只有GPU可以访问；
- MTLStorageMode内存less：表示资源只有GPU可以访问，且生命周期只是临时存在于一个渲染 pass期间；
- MTLStorageModeManaged：表示CPU和GPU分别会维护一份资源的拷贝，并且资源具有“可见性”，即无论哪边对资源进行了更改，CPU和GPU都可见都会进行更新同步；

如果将storeAction设置为MTLStoreActionStore表示RT的结果会从GPU的tile 内存写入到CPU的system 内存，即在system 内存中保存RT的备份。如果渲染后期还需要用到system 内存中备份的RT，就需要从system 内存中读取备份的RT到GPU的贴图缓存中。所以传统的双Pass延迟渲染中需要在第一个Pass和第二个Pass期间将G-缓存保存到system 内存中。

`_renderPassDescriptor.colorAttachments[AAPLRenderTargetAlbedo].storeAction = MTLStoreActionStore;`

`GBufferTextureDesc.storageMode = MTLStorageModePrivate;`

 

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040605279-1459942990.png)

 而基于TBDR架构，GPU是可以随时从tile 内存读取数据的，因此不需要等到从system 内存中读取G-缓存再进行光照计算，而是可以一步到位紧接着读取tile 内存中的数据RT进行延迟光照计算，并将最终的结果保存到system 内存用于显示即可。这样就不希望GPU再把G-缓存保存到system 内存，因此Metal中可以设纹理的storageMode为MTLStorageMode内存less即可，storeAction的值可以设为MTLStoreActionDontCare：

`_renderPassDescriptor.colorAttachments[AAPLRenderTargetAlbedo].storeAction = MTLStoreActionDontCare;`

`GBufferTextureDesc.storageMode = MTLStorageMode``内存less;`

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040530430-1635516775.png)

***注意：***

1. 这里基于TBDR架构允许GPU的FS片段着色器访问渲染 targets（color\[id\]）进行混合计算的特性就是programmable 混合，当然基于TBDR下FS也可以通过ImageBlocks特性访问同样的数据实现同样的功能。programmable 混合和Metal2的新特性ImageBlocks都可以实现一些类似的功能，但原理不同，可以重点学习比较各自的特点。
2. 在Tile based 着色中，G-缓存是被分成tile-size大小来保存的，因此可以将所有物体一次性渲染到tile-sized G-缓存中留在On-Chip 内存。要注意并不是仅仅G-缓存不保存到system 内存那么简单，实现的前提是Tile based，否则On-Chip 内存是无法装得下完整的屏幕大小的G-缓存的。Tile based是由于平台计算性能有限应运而生的GPU架构。

2.3 TBDR架构原理

上面提到TBDR架构下可以将RT保存在tile 内存降低GPU和CPU之间的带宽，但是并没有讲清楚TBDR实现的原理和底层流程。这里强调一下TBDR架构的原理和Metal中启用TBDR架构的方法。

**TBDR****原理**

IMR架构立即渲染的意思是每次提交一个模型物体就进行单独渲染，最后把所有物体混合。而TBDR架构是考虑到平台带宽压力导致手机发热而设计的，与IMR架构不同的是：TBDR是等待场景所有物体都提交之后再统一进行处理，然后将屏幕空间内的所有物体按照设置的tile size大小将屏幕分割成小块单独进行处理，这样所有在同一块tile上的几何图元会同时进行渲染，不在tile内的片段会在光栅化之前被剔除掉。这样一个tile可以在GPU方面快速进行渲染，最后将结果在CPU方面再拼成完整的一张屏幕图像。

TBDR架构是端牺牲效率换取带宽从而降低能耗的设计，在GPU上有一块缓存tile数据的cache，一块tile的渲染需要的数据可以直接从这块cache上读取在GPU上快速渲染，而不需要从system 内存来回传送数据。

关于TBDR架构（TBR和TBDR的区别）的具体原理建议阅读下面这篇文章的详细介绍：

**Metal****使用平铺架构**

事实上Metal并没有提供显式的方法去启用Tile Based，而是根据某些场景的代码实现和设置提示GPU启用tile 内存的。例如有两种启用的情况如下：

1）一个是上面提到的加载和存储操作和storgeMode的设置，当设置RT不保存到system 内存并且只给GPU访问的时候，GPU就会启用Tile Based将RT切成tile size大小保存在tile 内存进行快速处理。这就是单pass延迟渲染启用Tile base的方法；

2）另外一种情况就是Tile Based 着色，这种情况其实是有显式的启用方法的，会有一个专门的描述RenderPipelineState的MTLTileRenderPipelineDescriptor，可以指定采样规模和tileFunction等,例如Tile Based Forward+ 中culling阶段使用Tile 着色的描述方式：

`        MTLTileRenderPipelineDescriptor *tileRenderPipelineDescriptor = [MTLTileRenderPipelineDescriptor new];`

`        tileRenderPipelineDescriptor.label = @"Light Culling";`

`        tileRenderPipelineDescriptor.rasterSampleCount = AAPLNumSamples;`

`        tileRenderPipelineDescriptor.colorAttachments[0].pixelFormat = MTLPixelFormatBGRA8Unorm;`

`        tileRenderPipelineDescriptor.colorAttachments[1].pixelFormat = MTLPixelFormatR32Float;`

`        tileRenderPipelineDescriptor.threadgroupSizeMatchesTileSize = YES;`

`        tileRenderPipelineDescriptor.tileFunction = lightCullingKernel;`

三、Metal2新特性：光栅顺序组（ROG，Raster Order Groups光栅顺序组）

3.1 光栅顺序组的作用

ROG是干什么的呢？***官方解释：准确的控制并行的碎片着色器线程访问同一个像素的顺序。***

通俗点说，就是在渲染场景物体的时候，有些前后重叠遮挡的物体身上的碎片着色器可能会同时访问同一个坐标的那个像素数据，造成竞争，导致结果错误。而ROG就是用来同步这个像素的访问次序，防止竞争的发生。

这样解释可能还是不够直观，这里来看官方给出的一个例子。

假设有下面这种情况，镜头场景中有两个重叠的三角形，开发者代码中绘制的时候对于这种透明物体会按照从后往前的顺序绘制，也就是先调用后面蓝色三角形的绘制调用，然后再调用前面绿色三角形的绘制调用，Metal也会按照代码的顺序去执行绘制调用指令，这样看似乎这两次绘制调用是依次串行执行，但实际上并不是这样，GPU上的运算过程是高度并行的，虽然CPU发出的指令是先绘制蓝三角形，但在GPU上Metal并不能保证蓝三角形的碎片着色器会比绿三角形的先执行，Metal只能保证在混合混合的时候是按照绘制调用的顺序执行的，如下图所示：

那么问题来了，混合虽然保证串行不重叠了，但是混合之前的读写操作并无法保证串行，蓝三角形碎片着色器将混合后的结果写入像素的同时，可能绿三角的碎片着色器正在读取该像素的颜色，造成了竞争，如下图所示：

ROG就是解决上面这种数据读写冲突问题的。

3.2 光栅顺序组解决读写冲突

ROG解决读写冲突的方式为线程同步，即同步同一个像素或者采样点（如果是per-采样着色模式）对应的thread线程。实现上，开发者只要用ROG属性标记数据内存，这样多个线程访问同一个像素数据的时候就会等待当前线程写入数据结束再访问。下图展示了ROG同步两个线程，使得线程2等待线程1写入数据结束后才开始继续读取数据：

光栅顺序组就仅仅是用来同步线程解决读写冲突的吗？不仅仅如此，光栅顺序组在Metal2 A11上进行了扩展，作为新特性用于实现更多强大的功能，用途更广。

3.3 Metal2 A11新特性：Multiple 光栅顺序组

Metal2 A11开始对光栅顺序组进行了扩展，除了可以实现同步单通道imageblock和threadgroup 内存数据，还开始支持多个ROG的定义使用，开发者可以更加细粒度的控制线程的同步，进一步减少线程的等待时间。

Multiple 光栅顺序组优化渲染的典型例子就是本文主题中提到的：单Pass延迟渲染。

前面说到传统的双Pass延迟渲染，第一个Pass渲出G-缓存保存到system 内存，然后第二个Pass读取system 内存中的G-缓存进行延迟光照计算。然后A11的tile 内存的存在得以实现Tile based 着色，使得G-缓存被分成tile-sized大小从而继续保存在GPU imageblock内存中，直接继续进行延迟光照计算，在一个Pass中完成了延迟渲染，降低了数据带宽。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040433586-1319256393.png)

那么这里光栅顺序组是如何对单Pass延迟渲染进行性能优化的呢？

知道延迟渲染主要解决多光源场景渲染的效率问题，一般的GPU在进行多线程、多光源延迟光照计算时的过程是像下面这样的：

这种情况第二个光源想要读取G-缓存对当前像素进行光照计算时，必须等待第一个光源计算并写入结束才能开始读取G-缓存（光照计算结果和G-缓存放在一起）。

现在可以通过定义多个光栅顺序组来优化这个问题。开发者只要将G-缓存中的贴图资源和光照计算结果分开放到不同的光栅顺序组即可，例如将Lighting光照计算结果放到第一组，将G-缓存的albedo，normal，depth等放到第二组，这样A11就可以将这两组分开，第二个光源就随时可以读取第二组的G-缓存数据进行光照计算，只在写入第一组的Lighting光照计算结果时进行同步等待即可。优化后流程如下：

官方的延迟渲染Demo中单Pass的实现中已经实现了利用Multiple 光栅顺序组进行性能优化：

四、Metal2新特性：图像数据块（ImageBlocks）

4.1 ImageBlocks须知

1. ImageBlocks特性是从Metal2开始在ios上开始支持的，不支持macOS；
2. ImageBlocks仅可用于A11上的***分块函数和kernel函数(被分块函数和内核函数共享)***，ImageBlocks整合到了片段着色阶段和tile 着色阶段，也可用于kernel函数计算。分块函数中只能访问当前分块位置对应的ImageBlocks像素数据，而kernel函数中每个thread都可以访问到所在的threadgroup对应的整个ImageBlocks图像数据块；
3. 实际上ImageBlocks在iOS设备上一直是存在的，只是到了Metal2在A11才向开发者开放，开发者可以灵活自定义ImageBlocks的数据结构，可以通过（x，y）坐标和采样index来定位访问ImageBlocks的数据；

4.2 ImageBlocks结构

ImageBlocks是一个n \* m的二维数据结构，有宽度和高度，还有***像素深度***。ImageBlocks中的每个像素都可包含多个成员，每个成员保存在各自的切片当中。如下图，表示该ImageBlocks有三个切片，分别是albedo，specular和normal，也就是每个都像素包含这三个成员。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040344217-1237356935.png)

注意须知中说到ImageBlocks被分块函数和内核函数共享，另外ImageBlocks的生命周期是跨越整个tile阶段以及不同的绘制调用s和dispatches持续存在的，***意味着渲染流程和计算操作可以混合在一起，在一个Pass中完成***，就是利用这一点仅在GPU上实现很多经典的图形学算法，避免和CPU频繁的来回传送数据，大大降低带宽。

4.3 隐式（implicit）ImageBlocks和显式（explicit）ImageBlocks

隐式ImageBlocks其实就是默认从tile 内存接收数据的ImageBlocks，是在使用attachments渲染的时候通过loadAction和storeAction定义绑定的，隐式的ImageBlocks的数据组织跟color attachments的attribute属性一致（实质是Metal自动创建了一个Implicit ImageBlocks来匹配color attachment中的行为），每个成员每个attribute对应一个\[\[color(id)\]\]。

显式ImageBlocks则是开发者可以在shader中自定义ImageBlocks的layout结构。关于ImageBlocks在分块函数和kernel函数中的具体用法另外写文章总结，此处暂时省略。

`typedef struct`

`{`

`    half4 lighting [[color(0)]];`

`    float depth    [[color(1)]];`

`} ColorData;`

`template <int NUM_LAYERS>`

`struct OITData`

`{`

`    static constexpr constant short s_numLayers = NUM_LAYERS;`

`    rgba8storage colors         [[raster_order_group(0)]] [NUM_LAYERS];`

`    half         depths         [[raster_order_group(0)]] [NUM_LAYERS];`

`    r8storage    transmittances [[raster_order_group(0)]] [NUM_LAYERS];`

`};`

`// The imageblock structure`

`template <int NUM_LAYERS>`

`struct OITImageblock`

`{`

`    OITData<NUM_LAYERS> oitData;`

`};`

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040234645-1338035811.png)

***PLUS******：官方单Pass延迟渲染的Demo是利用programmable 混合在FS中直接访问GBuffer实现的，实际上也同样可以通过ImageBlocks来在FS中访问GBuffer。***

五、Xcode基本抓帧调试

Xcode编辑器自身提供了强大的抓帧工具和可视化的渲染流程展示，详细数据分析，开发者可以很方便的分析数据和控制渲染流程。

程序运行起来前需要先在编辑器：Product -> Scheme -> Edit Scheme -> Run -> Options 下，设置Metal API Validation为Enabled获取详细的调试数据：

程序运行起来后点击Capture Frame按钮截取一帧进行数据分析：

左侧导航栏展示了一帧中所有的渲染指令执行顺序和层次关系，点击对应指令右侧会展示可视化的渲染步骤：

右边窗口主要展示可视化的渲染流程和渲染过程中的各种资源数据等，其中渲染流程图中每个封闭的框表示一个Pass，每次\[commandBuffer commit\];表示封闭框结束。封闭框内部每一组数据对应一个renderEncoder，每次\[renderEncoder endEncoding\];对应框内一行的资源结束。每一组资源之间的连线表示数据的后续使用关系。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040110065-1937425457.png)

选中某一个drawprimitives绘制指令（绘制调用）可以看到shader的VS和FS中的数据流，以及color attachments的情况：

6.4 平行光和阴影计算

6.5 Cull the Light Volumes

6.6 绘制天空盒和粒子

七、总结

这篇文章以延迟渲染为背景，主要总结以下几个重要话题：

- 延迟渲染的原理，以及基于Metal新特性和平台TBDR的GPU架构实现单Pass延迟渲染的原理，背后降低数据带宽的原理等；
- 光栅顺序组的原理，以及新特性下优化TBDR性能的方法和原理；
- ImageBlock原理简介，应用场景和意义，Implicit ImageBlock和Explicit Block的区别；
- Xcode中Metal引擎渲染的基本调试方法；
- 官方延迟渲染Demo源码的结构的分析，知识点的应用。

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912040019110-1120734568.png)

 八、延迟渲染Demo源码分析

有了以上的知识储备现在再来分析官方的延迟渲染Demo就没有理论障碍了。

6.1 延迟渲染的一帧

Demo中基于延迟渲染场景实现了下面的步骤和效果：

1. 阴影贴图（Shadow map）；
2. 渲染G-buffer；
3. 平行光计算（Directional light）；
4. 光照掩盖（Light mask）；
5. 点光源计算（Point lights）；
6. 天空盒渲染（Skybox）；
7. 粒子绘制（Fairy lights）；

在iOS和tvOS上由于TBDR架构的支持得以在一个Pass中完成延迟渲染，因此可以依次连续完成上面的步骤：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912035706726-752116554.png)

而macOS的IMR GPU架构只能实现双Pass延迟渲染，因此要先在一个Pass中渲染G-buffer：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912035835880-1348400717.png)

 等commandBuffer commit之后，然后再进行后面的延迟光照计算等步骤：

![](https://img2023.cnblogs.com/blog/1251718/202309/1251718-20230912035637239-849036512.png)

参考文献链接

https://zhuanlan.zhihu.com/p/92840602