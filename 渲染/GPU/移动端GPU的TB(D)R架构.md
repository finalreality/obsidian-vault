---
title: 手机Gpu参数和架构 手机gpu型号排名_mob6454cc6dac54的技术博客_51CTO博客
source: https://blog.51cto.com/u_16099251/6737147
author: 
published: 
created: 2025-02-21
description: 
tags:
---
**文章目录**

- 移动端GPU的TB(D)R架构
- 移动端GPU概况
- 移动端和桌面端功耗对比
- 移动端和桌面端带宽对比
- 名词解释
- IMR（Immediate-mode Rendering）立即渲染
- TB(D)R（Tile-Based (Deferred) Rendering）
- “真假TBDR”
- TB(D)R流程
- TBR的优缺点
- 第一个Defer：Binning过程
- 第二个Defer：Early-DepthTest
- 移动端TBR优化

给大家分享了关于【图形渲染】的学习资料：

 [https://edu.51cto.com/course/27591.html](https://edu.51cto.com/course/27591.html?utm_platform=pc&utm_medium=51cto&utm_source=shequ&utm_content=bk_article)

## 移动端GPU的TB(D)R架构

## 移动端GPU概况

下图为2020年手机安卓端SoC的Top10排行  

![手机Gpu参数和架构 手机gpu型号排名_移动端](https://s2.51cto.com/images/blog/202307/16113131_64b36493e48f314806.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

- 目前的top300中，从手机SoC厂商来看，高通占比49.2%，相比于其他手机厂商优势较大，其次是华为28.6%，联发科21%，三星1.1%，其余厂商均未进入top300

下图为安卓端GPU的top10排行  

![手机Gpu参数和架构 手机gpu型号排名_数据_02](https://s2.51cto.com/images/blog/202307/16113131_64b36493e7fc331835.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

- GPU上，高通的Adreno与Mali的GPU市场分别占比49.2%与48.2%，而PowerVR-GPU在安卓机器上使用较少，占比仅有2.6%

### 移动端和桌面端功耗对比

- 桌面级主流性能平台功耗一般为300W（R7/I7+X60级别显卡），游戏主机150-200W
- 入门和旗舰游戏本平台功耗为100W
- 主流笔记本为50-60W，超极本为15-25W
- 旗舰平板为8-15W
- 旗舰手机为5-8W，主流手机为3-5W

主流的桌面端功耗在手机端的100倍左右

### 移动端和桌面端带宽对比

![手机Gpu参数和架构 手机gpu型号排名_数据_03](https://s2.51cto.com/images/blog/202307/16113132_64b364941eb2b44105.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

- 骁龙888的带宽也只有32，和桌面端依旧有100倍的差距

## 名词解释

**SoC（System on Chip）**

- SoC是把CPU、GPU、内存、通信基带、GPS模块等等整合在一起的芯片的称呼
- 常见的有A系SoC（苹果），骁龙SoC（高通），麒麟SoC（华为），联发科SoC，猎户座SoC（三星）
- 去年苹果推出了M1芯片，已经应用在了Mac和iPad上，说明了移动端和桌面端通用的芯片已经出现了

**System Memory**

- SoC中GPU和CPU共用一块片内LPDDR物理内存，就是我们常说的手机内存，也叫System Memory，大小为几个G
- CPU和GPU还分别有自己的高速SRAM的Cache缓存，也叫On-chip Memory，大小在几百K到几M
- 不同距离的内存访问存在不同的时间消耗，距离越近消耗越低，读取System Memory的时间消耗大概是On-chip Memory的几倍到几十倍
- SoC上的GPU和CPU共享一个内存地址空间，而桌面端的CPU和GPU的内存地址空间是分开的

**On-Chip Memory**

- 在TB(D)R架构下会存储Tile的颜色、深度和模板缓冲，读写修改都很快

**Stall**

- 当一个GPU核心的两次计算结果之间有依赖关系而必须串行时，等待的过程便是Stall

**Fill Rate**

- 像素填充率 = ROP运行的时钟频率 × ROP的个数 × 每个时钟ROP可以处理的像素个数

## IMR（Immediate-mode Rendering）立即渲染

用户应用程序提交的数据先经过顶点Shader的处理，然后以先进先出的顺序提交给片元着色器，片元着色器最终把结果写入FrameBuffer中

下图为简化的IMR流程图及伪代码表示  

![手机Gpu参数和架构 手机gpu型号排名_技术美术_04](https://s2.51cto.com/images/blog/202307/16113132_64b364941c45a96688.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

![手机Gpu参数和架构 手机gpu型号排名_数据_05](https://s2.51cto.com/images/blog/202307/16113132_64b36494441b54706.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

下图为PowerVR官网上的详细的IMR示意图  

![手机Gpu参数和架构 手机gpu型号排名_技术美术_06](https://s2.51cto.com/images/blog/202307/16113132_64b3649421d1441202.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

## TB(D)R（Tile-Based (Deferred) Rendering）

TB(D)R是目前主流的移动GPU渲染架构

- TB(D)R简单的意思：屏幕被分块（16 \* 16像素或32 \* 32像素）渲染
- TBR：VS - Defer - RS - PS
- TBDR：VS - Defer - RS - Defer - PS

- 从渲染数据的角度来看，Defer就是“阻塞+批处理”GPU的“一帧”的多个数据，然后一起处理

### “真假TBDR”

严格意义上的TBDR是PowerVR公司最先提出并保有专利的，其他公司不能这么叫。但是由于手机厂商之间的互相“借鉴学习”，从2016年起，现在市面上的GPU都已经是类似TBDR的架构，因此也可以用TBDR来泛指当前的所有移动端GPU架构

### TB(D)R流程

TB(D)R宏观上总共分为两个阶段

1. 执行所有与几何相关的处理，生成Primitive List（图元列表），并确定每个tile上有哪些primitive（分图元）
2. 逐块执行光栅化及其后续处理，并在完成后将Frame Buffer从Tile Buffer写回System Memory中（相比于传统的立即渲染框架，不是直接写回系统内存，而是写到片上内存里）

![手机Gpu参数和架构 手机gpu型号排名_Memory_07](https://s2.51cto.com/images/blog/202307/16113132_64b3649425ce88575.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

![手机Gpu参数和架构 手机gpu型号排名_数据_08](https://s2.51cto.com/images/blog/202307/16113132_64b36494291c081549.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

下图为TBDR的详细流程  

![手机Gpu参数和架构 手机gpu型号排名_技术美术_09](https://s2.51cto.com/images/blog/202307/16113132_64b3649452b9557245.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

- 相比IMR增加了Tiling过程，会把顶点经过几何处理之后得到的数据写入到**系统内存**中
- 经过了逐片元操作之后的渲染结果被写入到**片上内存**中，最后才写回系统内存

通过下图简化的TBR和IMR结构图，可以更直观对比出二者的区别  

![手机Gpu参数和架构 手机gpu型号排名_Memory_10](https://s2.51cto.com/images/blog/202307/16113132_64b3649457b622023.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "在这里插入图片描述")

- 黄色为处理单元，蓝色为存储单元，黑框为独有的部分

下图为动态演示TBR的渲染流程  

![手机Gpu参数和架构 手机gpu型号排名_手机Gpu参数和架构_11](https://s2.51cto.com/images/blog/202307/16113132_64b3649466f0792636.gif "请添加图片描述")

### TBR的优缺点

TBR的核心目的是降低带宽、减少功耗，但渲染帧率上并**不比IMR快**

优点：

1. TBR给消除Overdraw提供了机会，PowerVR用了HSR技术，Mali用了Forward Pixel Killing技术，目标一样，就是要最大限度减少被遮挡pixel的texturing和shading
2. TBR主要是 cached（缓存） friendly, 在cache里的速度要比全局内存的速度快的多，以及有可能降低render rate的代价，降低带宽，省电

缺点：

1. binning过程是在vertex阶段之后，将输出的几何数据写入到DDR，然后才被fragment shader读取。几何数据过多的管线容易在此次遇到性能瓶颈
2. 如果某些三角形叠加在数个tile上，需要绘制数次，这意味着总渲染时间将高于即时渲染模式

## 第一个Defer：Binning过程

确定了图元该由哪些块元来渲染  

![手机Gpu参数和架构 手机gpu型号排名_数据_12](https://s2.51cto.com/images/blog/202307/16113132_64b36494550f272419.png?x-oss-process=image/watermark,size_14,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=,x-oss-process=image/resize,m_fixed,w_1184/format,webp "请添加图片描述")

- 当发现一个项目中Binning耗时特别多的话，就要考虑几何数据是不是过多了

## 第二个Defer：Early-DepthTest

不同的GPU的处理方法不同  
Android：

- Qualcomm Adreno 采用外置模块LRZ。在正常渲染管线前，先多执行一次VS 生成低精度depth texture，以提前剔除不可见的triangles（实现细节未公开）。相当于直接用硬件做occlusion culling，功能类似软光栅遮挡剔除。因为做LRZ时执行VS只需用到position信息，所以单独抽出position stream，能带来bandwidth和cache的优化
- Arm Mali 采用Forward Pixel Kill技术。在Early-Z之后，使用一个FIFO队列，队列中的每个单位会有位置信息和深度信息Z越大距离越远。当有相同位置的单位进入队列时，Z更大的那个会被“Killed”从队列中移除

IOS：

- PowerVR的HSR技术。模拟发射一条射线，当遇到第一个不透明的物体之后，打断后面三角形的ps处理

下图为HSR的效果示意

## 移动端TBR优化

- 记得不使用Framebuffer的时候clear或者discard  
主要是清空积存在tile buff上的中间数据，所以在unity里面对render texture的使用也特别说明了一下，当不再使用这个rt之前，调用一次Discard。在OpenGL ES上善用glClear，gllnvalidateFrameBuffer避免不必要的Resolve（刷system memory）行为
- 不要在一帧里面频繁的切换framebuffer的绑定  
本质上就是减少tile buffer 和system memory之间的 的stall 操作
- 对于移动平台，建议你使用 Alpha 混合，而非 Alpha 测试。在实际使用中，你应该分析并比较 Alpha 测试和 Alpha 混合的表现，因为这取决于具体内容，因此需要测量，通常在移动平台上应避免使用 Alpha 混合来实现透明。需要进行 Alpha 混合时，尝试缩小混合区域的覆盖范围
- 不得不做Alpha Test的时候，需要增加一个preZpass
- 图片尽量压缩 例如:ASTC ETC2
- 图片尽量开启mipmap
- 尽量使用从Vertex Shader传来的Varying变量UV值采样贴图（连续的），不要在FragmentShader里动态计算贴图的UV值（非连续的），否则CacheMiss
- 在延迟渲染尽量利用Tile Buffer 存储数据
- 如果你在Unity 里面调整 ProjectSetting/Quality/Rendering/Texture Quality 不同的设置，或者不同的分辨率下，帧率有很多的变化，那么十有八九是带宽出了问题
- MSAA(增加对framebuffer读取的次数)其实在TBDR上反而是非常快速的
- 少在FS 中使用 discard 函数，调用gl\_FragDepth从而打断Early-DT( HLSL中为Clip，GLSL中为discard )
- 在shader中区分使用float、half、fix：  
（1）带宽用量减少  
（2）GPU中使用的周期数减少，因为着色器编译器可以优化你的代码以提高并行化程度  
（3）要求的统一变量寄存器数量减少，这反过来又降低了寄存器数量溢出风险  
（4）详情参考 [《Unity3D内建着色器源码剖析》作者熊大的优化建议](http://www.xionggf.com/post/unity3d/shader/u3d_shader_optimization/)
- 在移动端的TB(D)R架构中，顶点处理部分，容易成为瓶颈，避免使用曲面细分shader，置换贴图等负操作，提倡使用模型LOD,本质上减少FrameData的压力

给大家分享了关于【C/C++】的学习资料：

 [https://edu.51cto.com/course/34470.html](https://edu.51cto.com/course/34470.html?utm_platform=pc&utm_medium=51cto&utm_source=shequ&utm_content=bk_article)

本文章为转载内容，我们尊重原作者对文章享有的著作权。如有内容错误或侵权问题，欢迎原作者联系我们进行内容更正或删除文章。

- **赞**
- **收藏**
- **评论**
- **举报**

**相关文章**

[![](https://ucenter.51cto.com/images/noavatar_middle.gif?x-oss-process=image/format,webp/ignore-error,1)](https://blog.51cto.com/u_16099251)

- 199.7万

人气

- 9

评论

- 20

收藏

**近期文章**

- [1.nginx结合pagespeed](https://blog.51cto.com/u_9270117/13369494 "nginx结合pagespeed")
- [2.本地大模型编程实战(20)用langgraph和智能体实现RAG(Retrieval Augmented Generation,检索增强生成)(4)](https://blog.51cto.com/u_16841336/13368907 "本地大模型编程实战(20)用langgraph和智能体实现RAG(Retrieval Augmented Generation,检索增强生成)(4)")
- [3.fork-fork总共几个进程](https://blog.51cto.com/u_9270117/13369878 "fork-fork总共几个进程")
- [4.让ROOT权限一次获取，终生受用](https://blog.51cto.com/u_9270117/13369743 "让ROOT权限一次获取，终生受用")
- [5.openldap---ldapsearch使用](https://blog.51cto.com/u_9270117/13369706 "openldap---ldapsearch使用")

[![新人福利](https://s2.51cto.com/blog/activity/bride/DetailsBride.gif?x-oss-process=image/ignore-error,1)](https://blog.51cto.com/activity-first-publish#xiang)

**文章目录**

- 移动端GPU的TB(D)R架构
- 移动端GPU概况
- 移动端和桌面端功耗对比
- 移动端和桌面端带宽对比
- 名词解释
- IMR（Immediate-mode Rendering）立即渲染
- TB(D)R（Tile-Based (Deferred) Rendering）
- “真假TBDR”
- TB(D)R流程
- TBR的优缺点
- 第一个Defer：Binning过程
- 第二个Defer：Early-DepthTest
- 移动端TBR优化