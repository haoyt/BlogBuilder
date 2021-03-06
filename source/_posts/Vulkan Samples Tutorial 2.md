---
title: Vulkan Samples Tutorial 2
date: 2017-04-07 10:18:23
tags: Vulkan
---

<!-- TOC -->

- [Swapchain](#swapchain)
    - [Vulkan and the Windowing System](#vulkan-and-the-windowing-system)
    - [Revisiting Instance and Device Extensions](#revisiting-instance-and-device-extensions)
    - [Queue Family and Present](#queue-family-and-present)
    - [Swapchain Create Info](#swapchain-create-info)
    - [Different Queue Families for Graphics and Present](#different-queue-families-for-graphics-and-present)
    - [Create Swapchain](#create-swapchain)
    - [Create Image Views](#create-image-views)

<!-- /TOC -->

## Swapchain

本章节的代码在文件 `05-init_swapchain.cpp` 中。

**Swapchain** 即帧缓存机制，它是使用了一系列的帧缓存区用来保证帧刷新率稳定的技术。帧缓存区通常放在显卡内存中，但也可以放在内存中。Swapchain 需要通过图形API开启并使用，未使用 Swapchain 会导致刷新不一致等问题。每个 swap chain 至少有两个缓冲区，一个作为屏幕显示缓冲区，另外一个作为后台缓冲区。两个缓冲区一般被称为 **presentation** 和 **swapping** 。

本小节讲述了如何创建**swapchain**。首先需要做的就是创建用来渲染的图形缓冲区(image buffers)。
![Swapchain](Vulkan Samples Tutorial 2/Swapchain.png)
> 上图展示了Swapchain与其他各部分之间的联系，图中的其他部分与下面所讲的内容关系也十分密切，需要注意。

### Vulkan and the Windowing System

与其他的图形API相似的是，Vulkan将有关于窗口系统的API与图形核心API分离开了。在Vulkan中，窗口系统的细节是通过 **WSI** 扩展(Window System Integration extensions)呈献给开发者的。我们可以从Vulkan的设计规范中找到与WSI相关的扩展的文档。在[LunarG LunarXchange website](https://vulkan.lunarg.com/)和[Khronos Vulkan Registry](https://www.khronos.org/registry/vulkan/)这两个地方可以找到Vulkan API规范和发布的扩展的使用规范。

WSI扩展包含了对多种平台的支持。使用WSI扩展是通过定义下面几个宏定义来实现的：
+ VK_USE_PLATFORM_ANDROID_KHR - Android
+ VK_USE_PLATFORM_MIR_KHR - Mir
+ VK_USE_PLATFORM_WAYLAND_KHR - Wayland
+ VK_USE_PLATFORM_WIN32_KHR - Microsoft Windows
+ VK_USE_PLATFORM_XCB_KHR - X Window System, using the XCB library (Apples)
+ VK_USE_PLATFORM_XLIB_KHR - X Window System, using the Xlib library (Apples)
**KHR** 命名后缀表示扩展是按照 **Khronos** 组织的规范实现的。

Surface Abstraction

Vulkan 使用 `VkSurfaceKHR` 对象来作为本地平台显示层或者窗口的抽象化对象。该对象是在**VK_KHR_surface**扩展中定义的。WSI扩展的作用是创建，操纵或者销毁显示层对象(surface objects)。

### Revisiting Instance and Device Extensions

在前面的几个章节中，我们推迟了扩展的设置，现在需要我们回顾之前 Instance 和 Device 的扩展设置，从而能够让我们学会如何启动WSI扩展。

Instance Extensions

使用WSI扩展的首要步骤是，启动表示层扩展(surface extension)。查看本章样例中使用的`init_instance_extension_names()`函数的实现代码，我们发现样例是将`VK_KHR_SURFACE_EXTENSION_NAME`添加到Instance扩展列表：
```
void init_instance_extension_names(struct sample_info &info) {
    info.instance_extension_names.push_back(VK_KHR_SURFACE_EXTENSION_NAME);
#ifdef __ANDROID__
    info.instance_extension_names.push_back(VK_KHR_ANDROID_SURFACE_EXTENSION_NAME);
#elif defined(_WIN32)
    info.instance_extension_names.push_back(VK_KHR_WIN32_SURFACE_EXTENSION_NAME);
#else
    info.instance_extension_names.push_back(VK_KHR_XCB_SURFACE_EXTENSION_NAME);
#endif
}
```
该函数不仅添加了通用Surface扩展，还针对其他平台添加对应的扩展。比如，针对Win32平台会将`VK_KHR_WIN32_SURFACE_EXTENSION_NAME`添加到Instance扩展列表中。

这些扩展将会在Instance创建时加载，加载的实现代码可以在`init_instance()`中进行查看。
> 注意：在`init_instance()`中我们看到`instance_extension_names`列表是通过传指针数组将内容的地址设置到`ppEnabledExtensionNames`上的。

Device Extensions

Swapchain 是一个图像缓存的列表，GPU向其中输入图像，该列表的图像会被呈现到显示输出设备。每当GPU向其中写入图像数据，device-level 扩展就会开始处理 Swapchain。所以，在进行 device 初始化之前需要指定device扩展，在`init_device_extension_names()`函数中，我们看到使用的device扩展是`VK_KHR_SWAPCHAIN_EXTENSION_NAME`:
```
void init_device_extension_names(struct sample_info &info) {
    info.device_extension_names.push_back(VK_KHR_SWAPCHAIN_EXTENSION_NAME);
}
```
这些扩展将在创建Device时进行加载，加载的实现代码可以在`init_device()`中进行查看，与`init_instance()`类似。

Instance & Device Extensions recap：
+ 本节样例使用 `init_instance_extension_names()` 函数来加载常用的surface扩展，并将平台相关的对应扩展也加到了instance扩展列表中了。
+ 本节样例使用 `init_device_extension_names()` 函数来加载一个Swapchain设备扩展。

### Queue Family and Present

**Present** 操作，就是使一个Swapchain图像缓冲放到物理显示设备上的操作。当我们的应用程序需要显示图像，那就需要向某一个GPU设备队列发送一个**呈现**(Present)请求（具体方法是调用`vkQueuePresentKHR()`实现的）但接受请求的GPU队列需要能够支持present请求，或者支持graphics和present请求。下面的代码就展示了如何找到一个支持graphics和present操作的GPU队列：
```
// Iterate over each queue to learn whether it supports presenting:
VkBool32 *pSupportsPresent =
    (VkBool32 *)malloc(info.queue_family_count * sizeof(VkBool32));
for (uint32_t i = 0; i < info.queue_family_count; i++) {
    vkGetPhysicalDeviceSurfaceSupportKHR(info.gpus[0], i, info.surface,
                                         &pSupportsPresent[i]);
}

// Search for a graphics and a present queue in the array of queue
// families, try to find one that supports both
info.graphics_queue_family_index = UINT32_MAX;
info.present_queue_family_index = UINT32_MAX;
for (uint32_t i = 0; i < info.queue_family_count; ++i) {
    if ((info.queue_props[i].queueFlags & VK_QUEUE_GRAPHICS_BIT) != 0) {
        if (info.graphics_queue_family_index == UINT32_MAX)
            info.graphics_queue_family_index = i;

        if (pSupportsPresent[i] == VK_TRUE) {
            info.graphics_queue_family_index = i;
            info.present_queue_family_index = i;
            break;
        }
    }
}

if (info.present_queue_family_index == UINT32_MAX) {
    // If didn't find a queue that supports both graphics and present, then
    // find a separate present queue.
    for (size_t i = 0; i < info.queue_family_count; ++i)
        if (pSupportsPresent[i] == VK_TRUE) {
            info.present_queue_family_index = i;
            break;
        }
}
free(pSupportsPresent);

// Generate error if could not find queues that support graphics
    // and present
    if (info.graphics_queue_family_index == UINT32_MAX ||
        info.present_queue_family_index == UINT32_MAX) {
        std::cout << "Could not find a queues for graphics and "
                     "present\n";
        exit(-1);
    }
```
上面这份代码，再次使用在这之前便已获取的变量`info.queue_family_count`，通过调用`vkGetPhysicalDeviceSurfaceSupportKHR()`函数来获取每一个queue family的是否支持surface的标志。然后遍历所有的queue family，来寻找同时支持present和graphics的GPU队列族。最后，如果发现没有找到同时支持present和graphics的GPU队列族，则去寻找一个支持present的GPU队列族，将present和graphics功能分到两个GPU queue family上。接下来的代码，会从`graphics_queue_family_index`获取执行图形命令的队列编号，从`present_queue_family_index`获取执行呈现操作的队列编号。

`free()`上面的if语句实际上有些多余，但这里如此写是为了针对没有都支持graphics和present的GPU队列的情况做说明的。

最后，如果发现graphics或者present功能找不齐，该代码会结束运行。

### Swapchain Create Info

接下来更多的内容是设置Swapchain的Create Info结构体：
```
typedef struct VkSwapchainCreateInfoKHR {
    VkStructureType                  sType;
    const void*                      pNext;
    VkSwapchainCreateFlagsKHR        flags;
    VkSurfaceKHR                     surface;
    uint32_t                         minImageCount;
    VkFormat                         imageFormat;
    VkColorSpaceKHR                  imageColorSpace;
    VkExtent2D                       imageExtent;
    uint32_t                         imageArrayLayers;
    VkImageUsageFlags                imageUsage;
    VkSharingMode                    imageSharingMode;
    uint32_t                         queueFamilyIndexCount;
    const uint32_t*                  pQueueFamilyIndices;
    VkSurfaceTransformFlagBitsKHR    preTransform;
    VkCompositeAlphaFlagBitsKHR      compositeAlpha;
    VkPresentModeKHR                 presentMode;
    VkBool32                         clipped;
    VkSwapchainKHR                   oldSwapchain;
} VkSwapchainCreateInfoKHR;
```
下面的几段文章，讲述了创建设置Swapchain前后的全部过程。

Creating a Surface

要在instance和device中使用WSI扩展，首先需要创建一个`VkSurface`对象，然后才能用它来进行Swapchain的创建。在`05-init_swapchain.cpp`文件的`main`函数开始的地方，我们可以看到针对本地平台的Surface的创建过程：
```
#ifdef _WIN32
    VkWin32SurfaceCreateInfoKHR createInfo = {};
    createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
    createInfo.pNext = NULL;
    createInfo.hinstance = info.connection;
    createInfo.hwnd = info.window;
    res = vkCreateWin32SurfaceKHR(info.inst, &createInfo, NULL, &info.surface);
#endif
```
本段代码中的`info.connection`和`info.window`分别在`init_connection()`和`init_window()`函数中针对本地平台进行了创建和初始化。在Vulkan中，通过`vkCreateWin32SurfaceKHR()`函数创建的VkSurfaceKHR Surface对象表示一个用于处理平台窗口的对象。

> 注意：`init_connection()`和`init_window()`是平台相关的，这两个函数为我们连接显示设备创建窗口程序屏蔽了很多细节。

接下来我们需要将创建的`info.surface`对象设置到Swapchain Create Info结构体中：
```
VkSwapchainCreateInfoKHR swapchain_ci = {};
swapchain_ci.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
swapchain_ci.pNext = NULL;
swapchain_ci.surface = info.surface;
```

Device Surface Formats

当你创建Swapchain时，你需要指定surface的格式(Formats)。**Format** 在这里表示`VkFormat`枚举器中描述的像素格式，比如：`VK_FORMAT_B8G8R8A8_UNORM`格式就是一种Device SUrface Format。

本章节接下来的一段代码就是获取一个`VkSurfaceFormatKHR`结构体列表，该列表保存着显示设备支持的`VkFormat`格式以及其他信息。本样例代码不关心用什么格式，于是便采用了第一个可用格式作为将要使用的格式。

然后，我们在本章节代码的后面部分可以看到`VkFormat`被设置到Swapchain create info结构体中：
```
swapchain_ci.imageFormat = info.format;
```

Surface Capabilities

获取到Formats之后，我们需要获取到Surface Capabilities和Present Modes来填充Swapchain的create info结构体，在本章节代码中，我们看到调用了`vkGetPhysicalDeviceSurfaceCapabilitiesKHR()`和 `vkGetPhysicalDeviceSurfacePresentModesKHR()`两个函数分别获取Surface Capabilities和Present Modes，之后的部分我们可以看到这两个对象中的信息被设置到了Swapchain create info结构体中：
```
uint32_t desiredNumberOfSwapChainImages = surfCapabilities.minImageCount;

swapchain_ci.minImageCount = desiredNumberOfSwapChainImages;
swapchain_ci.imageExtent.width = swapChainExtent.width;
swapchain_ci.imageExtent.height = swapChainExtent.height;
swapchain_ci.preTransform = preTransform;
swapchain_ci.presentMode = swapchainPresentMode;
```
`minImageCount`属性，用于设置缓冲区使用策略：双缓冲区、三缓冲区等。本章节代码中是从使用`vkGetPhysicalDeviceSurfaceCapabilitiesKHR()`函数获取到到的`surfCapabilities`对象中查到Device支持的图像缓冲区的最小个数。


### Different Queue Families for Graphics and Present

在本章之前的代码中，我们找到了有graphics和present功能的GPU队列，如果他们并不是同一个队列，我们需要做一些其他设置，让它们能够共享图像缓冲：
```
swapchain_ci.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
swapchain_ci.queueFamilyIndexCount = 0;
swapchain_ci.pQueueFamilyIndices = NULL;
uint32_t queueFamilyIndices[2] = {
    (uint32_t)info.graphics_queue_family_index,
    (uint32_t)info.present_queue_family_index};
if (info.graphics_queue_family_index != info.present_queue_family_index) {
    // If the graphics and present queues are from different queue families,
    // we either have to explicitly transfer ownership of images between the
    // queues, or we have to create the swapchain with imageSharingMode
    // as VK_SHARING_MODE_CONCURRENT
    swapchain_ci.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    swapchain_ci.queueFamilyIndexCount = 2;
    swapchain_ci.pQueueFamilyIndices = queueFamilyIndices;
}
```
上面的代码是创建Swapchain常用的做法。本章节其余代码是设置Swapchain Create Info结构体的其他属性，可以参考用于将来自己的程序实现。


### Create Swapchain

在Swapchain Create Info结构体设置完成后，我们使用下面这一句代码创建Swapchain：
```
res = vkCreateSwapchainKHR(info.device, &swapchain_ci, NULL, &info.swap_chain);
```
`vkCreateSwapchainKHR()`函数创建了许多图像缓冲区用于建立Swapchain。

在程序进行时，我们可能需要获取每个Swapchain上的图像缓冲区，我们应该用下面这段符合之前获取格式的代码：
```
res = vkGetSwapchainImagesKHR(info.device, info.swap_chain,
                                  &info.swapchainImageCount, NULL);
    assert(res == VK_SUCCESS);

    VkImage *swapchainImages =
        (VkImage *)malloc(info.swapchainImageCount * sizeof(VkImage));
    assert(swapchainImages);
    res = vkGetSwapchainImagesKHR(info.device, info.swap_chain,
                                  &info.swapchainImageCount, swapchainImages);
    assert(res == VK_SUCCESS);
```
来获取Swapchain Images的列表，然后向其中输入数据，在之后通知GPU渲染他们。

### Create Image Views

`VkImageView`中存储着图像数据的信息，如：图像内存的排列格式，用于1D，2D还是3D，也用于记录图像的格式，颜色组成的顺序以及插件的信息。在本章节代码中，我们可以看到最后的`for`循环中使用`vkCreateImageView`创建了针对于每个Swapchain的image缓冲区的view缓冲区，存于info中。

在创建完image view之后，算是完成了创建Swapchain的过程，在后面的章节中我们将使用info中的iamge view。

---

本篇**Vulkan Samples Tutorial**原文是**LUNAEXCHANGE**中的[Vulkan Tutorial](https://vulkan.lunarg.com/doc/sdk/1.0.42.1/windows/tutorial/html/index.html)的译文。
并非逐字逐句翻译，如有错误之处请告知。O(∩_∩)O谢谢~~