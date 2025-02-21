# 1. Basic Concepts

## 1.1 Rendering

### 1.1.1 Attachment
Attachment的概念跟OpenGL/OpenGL ES中的Attachment一致，表示的是我们渲染时用作承载输出贴图的容器，类似于D3D中的RenderTarget，包括Color Buffer跟Depth/Stencil Buffer，在Vulkan中对应的就是Color Attachment与Depth/Stencil Attachment，注意，输入的texture在vulkan中并不是attachments，且只有那些带有图像格式的内存数据才能被用作Attachments，像D3D的Constant Buffer这种不具备固有贴图格式的数据是不能用作Attachment的。

跟RenderBuffer的创建一样，Attachment的创建也是通过预先指定description来完成的：

```cpp
VkAttachmentDescription attachments[2];
attachments[0].format = info.format;
attachments[0].samples = NUM_SAMPLES;
attachments[0].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attachments[0].storeOp = VK_ATTACHMENT_STORE_OP_STORE;
attachments[0].stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
attachments[0].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachments[0].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attachments[0].finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
attachments[0].flags = 0;

attachments[1].format = info.depth.format;
attachments[1].samples = NUM_SAMPLES;
attachments[1].loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
attachments[1].storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachments[1].stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
attachments[1].stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
attachments[1].initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
attachments[1].finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
attachments[1].flags = 0;
```

### 1.1.2 Subpass

Subpass是用于对Attachment进行读写的工作流，每个Subpass都会执行一次后面介绍的Pipeline（相当于一次渲染），同时会在初始化时完成此前Attachment设定的Load操作，并在渲染结束时完成Attachment设定的Store操作。

这个工作流是通过VkAttachmentReference实现与Attachment的关联：

```cpp
VkAttachmentReference color_reference = {};
color_reference.attachment = 0;
color_reference.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;

VkAttachmentReference depth_reference = {};
depth_reference.attachment = 1;
depth_reference.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;

VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
subpass.flags = 0;
subpass.inputAttachmentCount = 0;
subpass.pInputAttachments = NULL;
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &color_reference;
subpass.pResolveAttachments = NULL;
subpass.pDepthStencilAttachment = &depth_reference;
subpass.preserveAttachmentCount = 0;
subpass.pPreserveAttachments = NULL;
```

Subpass是Vulkan专门针对移动端TBDR架构设计的特性，在Vulkan中，每个渲染的RenderPass由多个Subpass组成，对于每个Subpass，我们会显式指定其输入域输出，在条件满足的时候，GPU驱动会自动将符合条件的多个subpass组合成一个renderpass，而经过组合后的单个RenderPass中的多个Subpass才是真正意义上的Subpass，否则在实际执行中，哪些非真正意义上的Subpass最终会被当成单个的Renderpass来执行，并没有达到Subpass的效率。

同时，只有经过组合后的单个RenderPass中的多个Subpass，其输出结果才会被保留在On-Chip Memory上，其他的Subpass都是按照RenderPass的执行逻辑，在结束的时候将数据写回到System Memory，在下一个RenderPass再从System Memory读回。

基于上述信息，我们在实践中使用Subpass的时候，就需要注意，不是手动指定一个subpass，就能够充分利用TBDR架构了，而是需要确保subpass能够成功的被组合到一个RenderPass中才行，也就是需要注意哪些情况下，subpass不会被组合到一个RenderPass中，下面给出其中一些无法组合的情形：

- subpass之间并不存在输入输出上的依赖关系
- subpass占用了过高的On-Chip Memory，导致无法全部保留在On-Chip上，只能写入到System Memory
- Subpass使用了此前Resolve后的Attachment（导致MultiSample Num变化）
- Subpass之间具有不同的MultiSample Num
- Subpass没有设置Color/Depth-Stencil Attachment
- Subpass使用了不同的ViewMasks
- 使用了Alias Attachment

### 1.1.3 RenderPass

RenderPass是对Subpass与attachment的封装，相当于将两者打包到一个盒子里，需要满足Subpass所关联的Attachments都要是Renderpass中所指定的，此外同一个RenderPass可以包含一系列的Subpass，多个subpass之间的执行顺序是不固定的，如果要指定执行顺序，就需要通过SubPassDependencies来施加约束：

```cpp
VkRenderPassCreateInfo rp_info = {};
rp_info.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
rp_info.pNext = NULL;
rp_info.attachmentCount = 2;
rp_info.pAttachments = attachments;
rp_info.subpassCount = 1;
rp_info.pSubpasses = &subpass;
rp_info.dependencyCount = 0;
rp_info.pDependencies = NULL;

res = vkCreateRenderPass(info.device, &rp_info, NULL, &info.render_pass);
```

  ### 1.1.4 VkMemory等

> VkMemory：很简单，就是指内存中一块连续的字节序列  
> VkImage：在VkMemory的基础上添加了一定的图像格式，即解析VkMemory所指内存的方式，由此我们可以通过一个像素（或者图素）来访问这个内存中的某一位置，而不需要通过指定字节的方式来访问这块内存。  
> VkImageView：选取VkImage中的一部分或者对VkImage中的格式作一定的改变以适应不同的使用环境（兼容不同的接口）  
> ————————————————  
> 版权声明：本文为CSDN博主「syddf_shadow」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。  
> 原文链接：[https://blog.csdn.net/yjr3426619/article/details/97394692](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fyjr3426619%2Farticle%2Fdetails%2F97394692)

  
### 1.1.5 Pipeline

指的是定义了渲染管线的各个阶段相关参数的数据结构，比如指定VS的输入数据，Vertex Shader，Pixel Shader，相关渲染配置等参数，当其中任意参数发生变化时，都需要重新创建一个新的pipeline。

Pipeline跟Subpass是一一对应的，即每个Pipeline都对应一个subpass，而每个subpass都对应一个pipeline，那么这里的一个问题是，为什么要多此一举，直接将二者合二为一不是更好吗？syddf_shadow给出的答案是：

> 如果需要实现延迟渲染，只用RenderPass，如果整个场景都是不透明的物体，那么我们至少需要两个RenderPass：第一个用于填充G-Buffer，第二个用于作光照计算得到最终的颜色。那么执行的流程是：  
> 第一个RenderPass进行渲染->将渲染结果写回到RenderTarget（G-Buffer）的内存中->第二个RenderPass读G-Buffer->第二个RenderPass进行渲染->将渲染结果写回到RenderTarget(Color Attachment)中  
> 可以看到中间有两步是要向RAM中写数据和读数据的，这里可以优化，第二个RenderPass如果能直接读取第一个RenderPass的结果，比如把第一个RenderPass的结果放在GPU芯片上的内存中或者直接合并两个Pass的操作，那么就省去了跨级存储器读写的高昂代价。  
> 这就是SubPass存在的意义，Vulkan中对于一个RenderPass内的Subpass，如果某一个SubPass需要前一个SubPass的渲染结果并且第二个SubPass中的任何一个像素只需读取前一个SubPass中同一像素位置所存储的数据时（不允许访问其他位置的像素），就会对此做一个与内存操作有关的优化。  
> 所以如果我们做类似于延迟渲染这样的多个Pass之间不会有像素级的随机存取操作时，就可以采用多个SubPass来提升性能。而如果是像SSAO、BLOOM这种需要用到多个不同位置像素的数据时，就要采用多个RenderPass。  
> ————————————————  
> 版权声明：本文为CSDN博主「syddf_shadow」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。  
> 原文链接：[https://blog.csdn.net/yjr3426619/article/details/97394692](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fyjr3426619%2Farticle%2Fdetails%2F97394692)

总的来说，Vulkan的Subpass是从Arm早期的PLS（Pixel Local Storage）做法中来的，其目的在于如果前后两次绘制存在输出数据-输入数据之间的转换，那么就可以考虑将第一次绘制的输出数据放到显存等Local Storage上，之后绘制的时候就可以直接取用，避免数据从Local Storage写入到内存之后再重新读回来导致的消耗。不过这种做法是有限制的，因为Local Storage（更正式的说法叫做On-Chip Cache，简称OCC）是有限的，在早期的时候，每个像素只有16bits，而后面做了升级，比如mali G71，每个pixel有128bits，那么在MSAA x2的情况下，color/depth都可以塞入到OCC上面，可以实现基本free（轻微sample平均的消耗）的抗锯齿效果，而对于MSAA x4的情况，depth就不能塞入到OCC上，导致消耗会有比较明显的增加，但是后面做了进一步升级，在mali G72以及更新的硬件中，每个pixel的OCC提高到256bit，所以在前向渲染下，即使是msaa x4，也是almost free的了。

## 1.2 Vulkan Context Setup

这一节主要是Vulkan架构与Setup相关的一些知识，跟前面Rendering中Vulkan是如何使用的不同，这一节会侧重介绍Vulkan是什么，其底层设计框架是什么。

### 1.2.1 Validation Layers

Vulkan API的设计理念是尽可能的减少对驱动层的overhead逻辑，而实现这个理念的一个做法是在API调用之前尽量不做或者少做调用有效性确认，因此即使调用方法时传入了错误的参数（简单的像是枚举数值错误，传入空指针等）都不会报错，从而最终以crash或者undefined behavior来报警。

之所以这样做，是因为Vulkan坚持开发者需要对API极其熟悉，知道传入的每一个参数的意义，但这种要求无疑过于坚硬，对开发者极为不友好，Vulkan设计者也清楚知道这一点，因此提供了一套有效性验证方案来进行弥补，这套方案就是Validation Layers，这是一套Vulkan使用时可选的组件，如果开发者对自己的代码非常有信心，可以无需理会，如果没啥信心或者需要追查bug，可以考虑将之开启。

Validation Layers通过挂接（hook into）到函数API中来执行额外的操作，这些操作包括：

- 检测参数的有效性
- 追踪对象的构造与析构
- 检查线程安全性
- 输出函数调用的详细日志
- 对函数调用进行统计，以便于进行profile & replay

下面给出Validation Layers的实现逻辑，简单来说就是对需要调用的API进行一层额外的封装：

```cpp
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("Null pointer passed to required parameter!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

Validation Layers可以很方便的进行开关，比如对于Debug开启，对于Release关闭，同时还可以根据需要添加任意多层的有效性检查，使用灵活，执行高效。不过Vulkan默认是不附带任何的Validation Layers的，不过这些Validation Layers都是[开源](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FKhronosGroup%2FVulkan-ValidationLayers)的，因此集成起来也很方便。

要想使用Validation Layers，首先需要完成安装工作，比如LunarG Validation Layer是专为PC设计的，需要在PC上完成Validation Layer的安装才可以使用。

Vulkan中的Validation Layers包括两类，分别是instance相关与device相关。instance检查的是跟全局vulkan对象相关的实例数据，而device则只检查特定GPU对应的函数调用。虽然如今device相关的检查基本上都被废弃了，但是specification文档依然推荐大家在device层面开启validation检查。

上面介绍了Validation Layers是什么，下面来看下vulkan提供的标准的Validation Layers要怎么用。跟扩展一样，validation layers需要通过名字来开启，所有的标准validation layers都是以VK_LAYER_KHRONOS_validation名字打包在Vulkan SDK中的，以这个为例，要想开启，需要完成如下几个步骤：

```cpp
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

const std::vector<const char*> validationLayers = {
    "VK_LAYER_KHRONOS_validation"
};

#ifdef NDEBUG
    const bool enableValidationLayers = false;
#else
    const bool enableValidationLayers = true;
#endif
```

添加一个函数判断需要的layer是否是available的。

```cpp
bool checkValidationLayerSupport() {
    uint32_t layerCount;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

    return false;
}
```

对之前的VK_LAYER_KHRONOS_validation layer进行验证

```cpp
for (const char* layerName : validationLayers) {
    bool layerFound = false;

    for (const auto& layerProperties : availableLayers) {
        if (strcmp(layerName, layerProperties.layerName) == 0) {
            layerFound = true;
            break;
        }
    }

    if (!layerFound) {
        return false;
    }
}

return true;
```

在layer available的情况下，添加debug代码。

```cpp
void createInstance() {
    if (enableValidationLayers && !checkValidationLayerSupport()) {
        throw std::runtime_error("validation layers requested, but not available!");
    }

    ...
}
```

对vkInstanceCreateInfo进行更改

```cpp
if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

如果validation layer可用，那么在创建instance的时候就不会遇到VK_ERROR_LAYER_NOT_PRESENT 报错。

关于Validation Layers的更多信息，可以参考[Vulkan编程指南(章节6-校验层)](https://links.jianshu.com/go?to=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F56604991)，此外，UE4.23中实现的Vulkan API也是添加了校验层的，默认是关闭的，需要通过开关r.Vulkan.EnableValidation进行打开。

### 1.2.2 Vukan Setup Structure

这一节会介绍Vulkan使用前的一些准备工作，包括VkInstance，VkDevice等相关的知识。

Vulkan是一个比较暴露（explicit）的API，对显卡等硬件资源提供了很高直接控制权限，也正因如此，开发者在使用之前需要经过较多的设置，比如加载扩展，在多个可用GPU中选择一个进行使用，创建vkInstance以及VkDevice等初始结构等。

#### 1.2.2.1 VkInstance

VkInstance是Vulkan的根（root）结构，这个结构表示的是整个API的Context，在创建VkInstance的时候，我们可以开启validation layers，加载所需要的扩展，同时还可以添加一个error回调绑定。

通常来说，每个vulkan的应用只需要一个VkInstance就够了，因为其代表的是一个应用的全局Context。

#### 1.2.2.2 VkPhysicalDevice

VkInstance创建成功后，就可以对当前的硬件进行搜索，查询是否具有满足VkInstance条件的GPU。Vulkan会将当前硬件上可用的GPU以及各个GPU所具备的能力等信息放在VkPhysicalDevice结构中，这个结构可以看成是一个GPU的引用或者说投影。

在一个独占式的PC上，因为只有一个独显GPU，因此只有一个VkPhysicalDevice是有效的，也就不需要进行选择。而在一些具有两个GPU的笔记本上（独显+集显），在这种情况下，就需要应用从这两个GPU中挑选一个符合自己需要的GPU作为渲染的target。

除了用作多个GPU挑选的一个入口之外，VkPhysicalDevice还可以用于查询GPU的能力，比如显存等信息，这在后续的应用中有着非常重要的作用。

#### 1.2.2.3 VkDevice

当我们拿到了符合条件的GPU的VkPhysicalDevice之后，我们就可以以之作为参数创建VkDevice。

VkDevice可以看成是GPU硬件的驱动，也是应用跟GPU进行交互的窗口，我们通常会称之为Logical Device。每个VkDevice创建的时候会需要传入一个所需要的扩展的列表作为参数，不过这里建议只传入每个应用真正需要用到的扩展进去，这样可以提升应用的性能，毕竟每个扩展都会要求驱动做一些工作来进行支持，从而导致性能下降。

Vulkan设计之处设定的一个很重要的目标是支持多GPU之间的数据与资源共享，而这一点正是通过VkDevice实现的，我们可以通过为每个GPU创建一个VkDevice，之后进行数据的共享，比如举个例子，我们可以为独显创建一个VkDevice用作渲染支持，而为集显创建一个VkDevice用作一些并行计算相关的支持工作。

一个物理设备可能有多种功能，可以把一种功能归为一个逻辑设备。一个物理设备可以对应多个逻辑设备。

#### 1.2.2.4 vkCommandBuffer

Command Buffers（CB）是用于记录后续提交到Device Queue中执行的命令的对象，vulkan提供了两层Command Buffer结构：

1. Primary Command Buffer，初级CB，可以实现对二级CB的调用执行，这个CB会被提交到Queue中
2. Secondary Command Buffer，二级CB，不会直接提交到Queue，可以被初级CB调用执行。

CB中的命令包含绑定pipeline的命令、设置CB的descriptor的命令、修改dynamic states的命令、渲染命令、dispatch命令、执行二级CB的命令、拷贝buffer/image的命令等。每个CB对state的管理是完全独立的，且初级CB跟二级CB之间的State是没有继承关系的，且每个CB在开始记录命令之前，其状态是undefined，且初级CB在一个execute的二级CB开始之后，状态也会变成undefined的。

而由于CB对象创建销毁成本较高，因此vulkan增加了一个CommandPool的对象，用于进行Command Buffer创销加速节省开销。

```
CommandBuffer是先收集一大堆命令，然后用vkQueueSubmit提交给设备的Queue。这个CommandBuffer是可以创建多个，BeginRenderPass调用时候传的CommandBuffer一般就是主CommandBuffer，而没传到RenderPass的都是子CommandBuffer，这样在多个线程上可以分别处理自己的命令到自己的子CommandBuffer上，最后都做完后通过ExecuteCommands把子CommandBuffer都合并到主CommandBuffer上，然后提交主CommandBuffer。
```

  
  
作者：离原春草  
链接：https://www.jianshu.com/p/b19704d966a8  
来源：简书  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。