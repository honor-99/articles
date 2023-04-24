<a name="index">**目录**</a>

- <a href="#ch1">**1 一个帧的构成**</a>
- <a href="#ch2">**2 窗口的表示——Surface**</a>
- <a href="#ch3">**3 帧合成服务——SurfaceFlinger**</a>
- <a href="#ch4">**4 窗口缓冲队列——BufferQueue**</a>
- <a href="#ch5">**5 两个重要的 HAL 模块——Gralloc 和 HWC**</a>
- <a href="#ch6">**6 显示系统各组件交互流程**</a>

<br>
<br>

### <a name="ch1">1 一个帧的构成</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

当我们点亮手机屏幕，看到的某一时刻的全屏画面就是一个 **帧**。

从微观来讲，一个帧的构成就是一个个的像素值，所以理论上来讲，最简单的办法就是由一个进程或服务搜集一屏画面的所有图像数据构成一帧即可。但是通常情况下一个全屏界面是由不同的进程来提供的，如图所示：

![Fullscreen frame compose](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/fullscreen_frame_compose.png "Fullscreen frame compose")

这是一个视频播放器的界面，整个界面包含：状态栏（status bar）、系统导航栏（system bar）、APP主窗口（main）、视频播放窗口（media player）4 个部分，每个部分的界面都是独立渲染的，其中 status bar 和 system bar 都属于 SystemUI进程，main 属于 APP进程，media player可以属于 APP进程也可以有自己独立的进程。可见一个进程既可以独立渲染一个界面，也可以包含多个渲染源，每个源独立渲染一个界面。

总之，一个帧是由不同的渲染源生成的，即先由各自渲染源独立生成子界面，然后再通过一个合成服务将所有子界面合成一个完整的帧。

我们约定，由一个渲染源生成的图像数据叫 **单元窗口**；再递归定义 **合成窗口** 为多个单元窗口合成或单元窗口与其它合成窗口合成所生成的图像数据；单元窗口与合成窗口统称为 **窗口**。显然，一个帧就是一个最终的合成窗口，那么除了帧以外的其它合成窗口我们称之为 **普通合成窗口**。

接下来分别简要介绍一下参与帧生成、合成和显示的各个组件，这些组件包括：窗口的表示——Surface、帧合成服务——SurfaceFlinger、窗口缓冲队列——BufferQueue、Android 硬件抽象层 HAL 以及两个重要的 HAL 模块，然后讲解一下各个组件之间的关系和交互流程，这样结合 [framebuffer和Vsync](https://github.com/huanzhiyazi/articles/issues/28)，整个 Android 的显示系统原理就基本完整了。

<br>
<br>

### <a name="ch2">2 窗口的表示——Surface</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

在 Android 中，负责生成窗口的接口叫 Surface，在应用层（java）和本地层（c/c++）中都有相应的实现。在术语上，也通常把 Surface 叫本地窗口。比如，我们在 APP 中看到的渲染的主界面就是一个 Surface，我们看到的顶部状态栏是一个 Surface，底部系统导航栏也是一个 Surface。

一个 Surface 的主要作用如下：

1. 负责管理一个窗口的各种属性，比如宽高、标志位、密度、格式（RGB颜色格式等）等。

2. 提供画布工具，用于绘制图像，生成单元窗口数据，比如我们通常的布局界面绘制用到了基于 Skia 的画布工具。图像数据还可以采用 OpenGL 绘制或者是视频解码器（通常由另一个接口 SurfaceView 实现）。

3. 向窗口缓冲队列（BufferQueue，由合成服务提供）申请窗口缓冲区，并将绘制好的图像数据写入窗口缓冲区中。

4. 将写好的部分帧缓冲区插入窗口缓冲队列，等待帧合成服务读取。

其中，只有作用 2 是单元窗口独有的，其它几个作用是除了帧（最终合成窗口）以外的所有单元窗口和普通合成窗口共有的，所以单元窗口和普通合成窗口都继承一个共同的抽象——ANativeWindow：

*[/frameworks/native/libs/nativewindow/include/system/window.h](http://aospxref.com/android-11.0.0_r21/xref/frameworks/native/libs/nativewindow/include/system/window.h)*
```c
// ...
struct ANativeWindow
{
    // 本地窗口属性
    const uint32_t flags;
    const int   minSwapInterval;
    const int   maxSwapInterval;
    const float xdpi;
    const float ydpi;
    intptr_t    oem[4];

    // 本地窗口功能
    int     (*setSwapInterval)(struct ANativeWindow* window, int interval);
    int     (*query)(const struct ANativeWindow* window, int what, int* value);
    int     (*perform)(struct ANativeWindow* window, int operation, ... );
    int     (*dequeueBuffer)(struct ANativeWindow* window, struct ANativeWindowBuffer** buffer, int* fenceFd);
    int     (*queueBuffer)(struct ANativeWindow* window, struct ANativeWindowBuffer* buffer, int fenceFd);
    int     (*cancelBuffer)(struct ANativeWindow* window, struct ANativeWindowBuffer* buffer, int fenceFd);
}
// ...
```

我们只需要关注其中最重要的两个方法 `dequeueBuffer` 和 `queueBuffer`，它们分别对应 Surface 的作用 3 和 作用 4。我们可以初步发现，从缓冲区的申请过程来看，BufferQueue 是 Surface 的服务提供方，而 BufferQueue 又是由 SurfaceFlinger 提供的，所以 Surface 和 SurfaceFlinger 之间是 CS 的通信模式，通信桥梁毫无疑问是 binder。

<br>
<br>

### <a name="ch3">3 帧合成服务——SurfaceFlinger</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

SurfaceFlinger 提供两种合成服务：

- **Client合成**：又叫软件合成，GPU合成，GL合成。其合成过程由 OpenGL 来完成，对于不能用硬件方式来进行合成的窗口，都需要委托 OpenGL 进行合成，比如某些特效的生成（背景模糊），或者窗口数超过硬件合成的限制等。存储 Client合成后的图像数据的缓冲区通常是由 framebuffer 驱动提供的（取决于厂商的实现，比如可能存储在 framebuffer 的 backbuffer 中，至于 backbuffer 是采用的软件缓冲技术还是 page-flipping 的方式，也取决于 OEM 的实现，详情可以参考 [framebuffer多缓冲实现原理](https://github.com/huanzhiyazi/articles/issues/28#ch3)）

- **Device合成**：又叫硬件合成。由具体的 OEM 根据其硬件特性来实现。专门用于 Device合成的模块叫 HWC (Hardware Composer)，HWC 向 OEM 提供用于硬件合成的接口，并由不同的 OEM 根据自身硬件特点来实现具体的合成算法。很显然，这个 HWC 是一个典型的 HAL 模块，我们将在后面具体介绍 HAL。

通常而言，Device合成的性能会比 Client合成要好。

目前主流的实现都是提前将不能进行 Device合成的窗口采用 Client合成，然后把 Client合成窗口和剩余的窗口一起进行 Device合成。所以，我们看到的合成模式通常如下图所示：

![Compose process](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/compose_process.png "Compose process")

如图所示，在 SurfaceFlinger 中也有一个 Surface 充当普通合成窗口。值得注意的是，在 Android 5.0 之前，普通合成窗口并不是由 Surface 表示的，而是有一个专门的类叫——FramebufferNativeWindow，因为事实上，只有一个普通合成窗口，它的合成数据是存储在 framebuffer 中的，这也是其名字 FramebufferNativeWindow 的由来。在 Android 5.0 之后，普通合成窗口也由普通的 Surface 表示，因为无论是单元窗口还是普通合成窗口，都可以抽象出来相同的三个操作：产生数据、申请缓冲区、填充缓冲区。这样对于 BufferQueue 而言，不用区分是单元窗口还是合成窗口了，简化了接口设计。

SurfaceFlinger 作为一个常驻的 binder 服务，在 init 进程启动时就被启动了。在 SurfaceFlinger 被启动之前，有两个重要的 HAL 模块也需要启动，一个是 Gralloc，用于 BufferQueue 缓冲区的实际分配；另一个是 HWC，用于进行 Device合成，还负责触发 Vsync 信号，通知 SurfaceFlinger 执行合成流程，另外 framebuffer 驱动一般也随 HWC 的启动而打开，以便提供屏幕基础信息和为 Client合成提供缓冲区。

Gralloc 和 HWC 都是由 init 进程触发启动的。

不难发现，Gralloc 和 HWC 充当了 SurfaceFlinger 的服务方，实际上它们也是通过 binder 完成的 CS 通信。从 Android 8.0 开始，framework 和 hal 层之间也通过 binder 驱动进行了隔离，以便于 framework 不用耦合与硬件无关的代码，也便于 OEM 更灵活地实现与自身硬件特性相关的模块。framework 与硬件服务之间进行通信的 binder 称作 hwbinder。

SurfaceFlinger 启动后，有几个重要的初始化：

1. Gralloc 客户端初始化，即建立 SurfaceFlinger 与 Gralloc HAL 之间的联系，用于缓冲区的分配。

2. GL渲染引擎初始化，用于 Client合成。

3. HWC 客户端初始化，即建立 SurfaceFlinger（客户端） 与 HWC HAL（服务方） 之间的联系，用于 Device合成以及注册 Vsync 通知触发合成过程。

4. 初始化并启动事件队列（类似于 Android 应用层的 Handler 机制），诸如 Vsync 通知合成、显示屏修改插拔等都将以事件的形式通知 SurfaceFlinger 进行处理。

值得注意的是，事件队列中显示屏增加事件产生后，将生成并初始化该显示屏对应的普通合成窗口 Surface，同时还要启动两个重要的服务线程 EventThread（ET） 和 DispVsyncThread（DT）。

**ET线程**

该线程主要处理客户端单元窗口下一帧绘制任务的调度请求，即 Surface 向 ET 询问，我有一个新的帧（这里的帧指的是单元窗口图像数据）绘制任务，是否可以执行了？ET 需要判断 Vsync 信号时机是否到达，一旦时机到了便通知 Surface 执行绘制任务。

ET线程是常驻的服务，ET线程在没有绘制任务的时候自我阻塞（wait），避免浪费 CPU 资源，直到有新的绘制任务且时机合适便自我唤醒（notify）。

那么，ET 如何判断 Vsync 信号时机到呢？这需要用到另一个协同服务线程 DT线程。

**DT线程**

该线程的主要功能是计算下一个 Vsync 信号的时机，并在 Vsync 信号时机到时唤醒 ET线程去遍历新的绘制任务并执行；同时还接收来自 HWC 的硬件 Vsync 信号更新，与自身计算的 Vsync 信号时机进行比对和调整。

同样，DT线程也是常驻的服务，在 Vsync 信号到来之前以及 HWC 通知更新之前是自我阻塞的，直到 Vsync 周期到或者 HWC 通知到才被唤醒。

需要注意的是，Surface 产生一个 `下一帧绘制任务的调度请求`（requestNextVsync）是通过 binder 来完成的；而 ET 通知 Surface 执行 `下一帧绘制任务`（onVsync） 却是通过 socket 完成的。为什么后者是 socket？因为在这一步，Surface 某种意义上是 ET 的服务方，但是 Surface 从一开始就是作为一个 binder 客户端存在的，它不能同时是 SurfaceFlinger 的 binder 客户端和 binder 服务端，binder 是单工的，所以这里用了 socket 作为替代方案，因为 socket 是双工 IPC，细节可以参考 BitTube 机制。下图描述了 Surface 向 SurfaceFlinger 订阅 Vsync 信号的过程：

![Surface register to Vsync](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/vsync_to_surface.png "Surface register to Vsync")

简言之，Surface 告知 ET，我要执行一个新的绘制任务了；ET 将此绘制任务缓存，并等待 DT 通知 Vsync 信号是否到；DT 计算出下一个 Vsync 信号时机，然后自我唤醒，同时唤醒 ET 执行缓存的绘制任务，准确来说是通知 Surface 执行绘制任务。

<br>
<br>

### <a name="ch4">4 窗口缓冲队列——BufferQueue</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

BufferQueue 作为共享资源，连接 Surface 和 SurfaceFlinger。其中，Surface 是资源生产者，SurfaceFlinger 是资源消费者。BufferQueue 的实现在 Android 不同版本中会有一些差异，但是作为生产者消费者模式的核心逻辑是不变的。

BufferQueue 有四个核心操作：

1. **dequeueBuffer**：向 BufferQueue 申请一块空闲缓冲区（目前版本中主流的最大缓冲区数量为 64 个，之前为 32 个，通常设置为 2 个或者 3 个，即黄油计划中的双缓冲和三缓冲机制），发起方为生产者（Surface）。之前已经申请过的缓冲区可以被复用，如果不符合要求（比如还没有申请过，缓冲区参数不匹配等）则需要重新申请新的缓冲区。

2. **queueBuffer**：向 BufferQueue 插入一块填充了有效数据的缓冲区，发起方为生产者（Surface）。

3. **acquireBuffer**：从 BufferQueue 摘取一块填充了有效数据的缓冲区用于合成或显示消费，发起方为消费者（SurfaceFlinger）。

4. **releaseBuffer**：将消费完毕的缓冲区释放，并送回 BufferQueue，发起方为消费者（SurfaceFlinger）。

缓冲队列中的每一块缓冲区也有四个核心状态：

1. **FREE**：初始状态，或者已被消费者 release，持有者为 BufferQueue，只能用于 dequeue 操作，可被 Surface 访问。

2. **DEQUEUED**：表示该块缓冲区已被生产者 dequeue，持有者为 Surface，只能用于 queue 操作，可被 Surface 访问。

3. **QUEUED**：表示该块缓冲区已经被生产者 queue，持有者为 BufferQueue，只能用于 acquire 操作，可被 SurfaceFlinger 访问。

4. **ACQUIRED**：表示该块缓冲区已经被消费者 aquire，持有者为 SurfaceFlinger，只能用于 release 操作，可被 SurfaceFlinger 访问。

生产者（Surface）、BufferQueue、消费者（SurfaceFlinger）三者之间的通信过程和缓冲区状态迁移示意图如下所示：

![BufferQueue](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/buffer_queue.png "BufferQueue")

值得注意的是，普通合成窗口 Surface 是在 SurfaceFlinger 里面的，即与 SurfaceFlinger 属于同一个进程，与单元窗口 Surface 的不同有二：

1. 单元窗口与 BufferQueue 之间通过 binder 通信的方式进行 dequeue 和 queue 操作，单元窗口侧访问 BufferQueue 的 binder 客户端叫 IGraphicBufferProducer；而普通合成窗口与 BufferQueue 属于同一进程，只需直接方法调用即可。

2. 单元窗口的生产目的是生成单元窗口数据，消费目的是用于窗口合成，包括 Client合成和 Device合成；而普通合成窗口的生产目的是进行 Client合成，消费目的是用于 Device合成和显示。

另外，每个 Surface 都对应一个 BufferQueue，这样 SurfaceFlinger 从不同的 BufferQueue 中取出来的缓冲区则对应不同的单元窗口或普通合成窗口。即，Surface 和 BufferQueue 是一对第一的关系，而这两者和 SurfaceFlinger 则是多对一的关系，如下图所示：

![BufferQueue relationship](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/buffer_queue_relationship.png "BufferQueue relationship")

<br>
<br>

### <a name="ch5">5 两个重要的 HAL 模块——Gralloc 和 HWC</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

我们知道，Android 作为操作系统，需要和底层的硬件进行通信，却没有办法知道不同原始设备制造商所定制的硬件差异。这是因为，不同的硬件厂商出于维护商业竞争优势，不会统一硬件的实现细节。但是操作系统又有需要整合不同厂商硬件的需要，不同的 OEM 为了便于自身产品销售，也需要遵循部分标准以兼容操作系统。

为此，Android 给不同类型的硬件提供了一整套统一的接口，这套接口就是一套协议，不同的硬件厂商必须实现这些接口，但是实现的细节却可以根据自身硬件的特性而有所不同。这样一来，操作系统对应用层屏蔽了同类型硬件因不同厂商定制的细节，只需关心接口的功能而进行调用即可。

这一套接口就叫做硬件抽象层（HAL），顾名思义，是对不同硬件的一种抽象，抽出共同的部分，屏蔽差异的部分，即求同存异。根据功能的不同，HAL 会被分割成不同的模块，Android 提供不同 HAL 模块的接口，并由不同的硬件厂商实现这些接口，这些接口实现还会包含不同设备驱动程序的实现。

孤立地讨论 HAL 意义不大，我们将其置于整个 Android 系统层次架构上来进行分析，其层次关系如下图所示：

![Android 系统架构](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/ape_fwk_all.png "Android 系统架构")

这是谷歌官方文档提供的架构图，具体说明可以参考[Android 系统架构](https://source.android.com/devices/architecture)。这里我们只针对显示系统中两个重要的 HAL 模块来做说明。

在第 3 节中，我们针对显示系统中两个重要的 HAL 模块——Gralloc 和 HWC 的功能做了简单的介绍。突出显示系统部分之后，Android 的层次架构如下图所示：

![hal](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/hal.png "hal")

几个要点如下：

1. 应用层的功能是绘制窗口，系统服务层作为应用层的服务方提供窗口合成服务。

2. 系统服务层的功能是进行窗口合成，HAL 层作为系统服务层的服务方提供缓冲区分配服务（Gralloc）和合成策略服务（HWC）。

3. Gralloc HAL 负责窗口缓冲区的分配，主要包括从何处分配内存、如何管理内存的申请和回收、实现共享内存机制；HWC HAL 提供窗口合成决策、负责 Device合成和显示。内核驱动层为 HAL 提供了访问文件设备的通道，需要注意的是，一个设备并不独属于一个 HAL 模块，比如 framebuffer 设备都会被 Gralloc 和 HWC 访问到。

4. 内核驱动层的功能是访问文件设备，这里的设备可以是虚拟设备（ion设备等）或硬件设备（屏幕等）。

5. 应用层和系统服务层之间用 binder 实现 IPC；系统服务和 HAL 之间用 hwbinder 实现 IPC；至于 HAL 访问内核驱动则是通过系统调用来实现的。

6. Gralloc 的一个重要功能是提供内存共享，因为应用层、系统服务层、HAL层三者之间分属不同的进程，窗口缓冲区作为大容量进程共享资源，必然需要以共享内存的方式来提供，而此共享内存的实现机制则依赖于内核驱动层的 ion设备。在早期的 Android 版本中，因为 HAL 并没有和系统服务层进行进程隔离，合成缓冲区 framebuffer 不需要作为共享内存的形式而存在，只有绘制缓冲区因为涉及到 Surface 和 SurfaceFlinger 之间的跨进程访问而需要以共享内存的形式而存在，故当时是将绘制缓冲区用 [Android匿名共享内存（ashmem）](https://github.com/huanzhiyazi/articles/issues/27)的方式来实现的。ion 提供了一种更为抽象的内存共享方式，其实现原理与 ashmem 类似。

7. HWC 在进行 Device合成之前，需要先决定哪些窗口可以进行 Device合成。具体而言，SurfaceFlinger 在执行合成之前，会询问 HWC 哪些窗口可以进行 Device合成，然后把不能进行 Device合成的窗口自行进行 Client合成作为普通合成窗口，然后统一与其它窗口一起交于 HWC 进行 Device合成。当然，在 SurfaceFlinger 询问 HWC 之前，也会有一些预处理，比如可以决定某些窗口强制进行 Client合成（比如有模糊背景需求的窗口）。关于 HWC 更加详细的作用描述可以参考谷歌官方文档——[硬件混合渲染器HAL](https://source.android.com/devices/graphics/hwc)。

<br>
<br>

### <a name="ch6">6 显示系统各组件交互流程</a><a style="float:right;text-decoration:none;" href="#index">[Top]</a>

以上我们单独介绍了显示系统各个重要组件的作用，并简单提及了某些组件之间的联系。本节作为一个总结，系统梳理一下各个组件间的交互流程是怎样的。

以下是各组件交互结构图：

![Android display process structure](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/android_display_process_structure.png "Android display process structure")

首先在上图中，有两个我们之前没有介绍的模块：

一个是图层（Layer），图层是 Surface 在其服务方的表示，在 SurfaceFlinger 中叫 Layer，在 HWC 中叫 hwLayer。从概念上讲理解到这一层就可以了，如果对细节感兴趣可以参考具体的源代码。

另一个是 HWComposer，这个是 SurfaceFlinger 中用于与 HWC HAL 进行沟通的代理。

接下来，我们具体介绍一下各个组件间的交互流程：

在客户端，各个不同渲染源将借助 Surface 绘制单元窗口数据，作为生产者将数据传送到 BufferQueue。不同的渲染源可以是 Skia（通常的窗口布局绘制工具）、OpenGL（如游戏渲染）、视频解码等。作为用例，以后将以 WindowManagerService 原理分析来说明 Skia 在其中的工作机制。

作为窗口缓冲区的 BufferQueue，将借助 Gralloc 进行缓冲区的申请和回收，Gralloc 借助 ion 对缓冲区以共享内存的方式进行访问。具体而言，面向单元窗口的缓冲区，来自于预分配的内存池；而面向普通合成窗口的缓冲区，来自于 framebuffer。Gralloc 屏蔽了这些共享内存的来源细节，SurfaceFlinger 只需要告诉 Gralloc 申请的内存用于何种用途即可，Gralloc 将据此决定具体的内存申请源头。

SurfaceFlinger 执行合成的流程如下：

1. **HWC 触发 vsync 信号**：vsync 信号将以回调的方式通知 SurfaceFlinger，SurfaceFlinger 收到 vsync 回调后开始执行下一帧的合成。

2. **SurfaceFlinger 更新图层**：SurfaceFlinger 遍历各个有效图层，从其对应的 BufferQueue 中获取最新的单元窗口绘制数据，以对图层进行更新。这一步的 BufferQueue 中的缓冲区来自于预分配内存。

3. **HWC 决策合成类型**：SurfaceFlinger 更新并准备好所有图层后，将图层参数告知 HWC HAL，HWC HAL 决定哪些图层可以执行 Device合成。

4. **SurfaceFlinger 执行 Client合成**：如果有 HWC 不能处理的图层，SurfaceFlinger 统一将它们交给 OpenGL 执行合成，其合成的数据作为一个普通合成窗口也插入到其对应的 BufferQueue 中，同时 SurfaceFlinger 还充当该 BufferQueue 的消费者将普通合成窗口取出并作为一个新的合成图层与其它普通图层一起准备交与 HWC 进行 Device合成。注意，这一步 BufferQueue 中的缓冲区来自于 framebuffer，也就是说 Client合成数据已经直接 post 到 framebuffer 中了。

5. **HWC 执行 Device合成**：HWC 将其余的图层连同 Client合成图层一起进行 Device合成。

6. **HWC 将合成的帧 post 到 framebuffer 显示**：要将帧显示出来，最终还是要将其 post 到 framebuffer 的 frontbuffer 中，这样显示控制器（display controller）才能从 framebuffer 中读取帧数据进行扫描。

各组件涉及到的跨进程通信方式有：

- **binder**：Surface（binder 客户端，以 IGraphicBufferProducer 作为代理） 与 SurfaceFlinger（binder 服务器，确切的说是 BufferQueue）之间。

- **hwbinder**：SurfaceFlinger（binder 客户端）与 HAL（binder 服务器，包括 Gralloc 和 HWC）。

- **共享内存**：单元窗口 Surface（生产者）与 SurfaceFlinger（消费者）和 HWC HAL（消费者）共享绘制内存；普通合成窗口 Surface（生产者）与 HWC（消费者）共享 Client合成内存。

最后，我们简单提及一下 Device合成的原理。目前，主流的 Device合成采用了一种叫 [overlay](https://en.wikipedia.org/wiki/Hardware_overlay) 的技术。简单来说，这是一种背景替换技术，类似于视频后期制作普遍用到的[色键(chroma key)技术](https://en.wikipedia.org/wiki/Chroma_key)。我们通常看到的电视里面的天气预报节目，主持人和气象背景图其实是后期合成的，在拍摄时，主持人的背景实际是一张纯绿色的幕布，和主持人一起组成一张前景图。在后期合成时，扫描前景图，并把绿色像素值替换为背景气象图同位置的像素值，这样便将前景主持人和背景气象图组合成我们最终看到的样子。

![Weather report chroma key](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/weather_report_chroma_key.jpg "Weather report chroma key")

我们再用一个示意图来理解一下有多个图层的情况：

![Overlay sample](https://raw.githubusercontent.com/huanzhiyazi/articles/master/%E6%8A%80%E6%9C%AF/android/Android%E6%98%BE%E7%A4%BA%E7%B3%BB%E7%BB%9F%E4%B9%8B%E2%80%94%E2%80%94Surface%E5%92%8CSurfaceFlinger/images/overlay_sample.png "Overlay sample")

可以看到，合成的原理就是每次给最上层的前景层替换一个背景层。图中有 A B C 三个 overlay 图层，当扫描绿色遮罩的时候，优先取最上层（C）的像素值进行替换，如果也是遮罩色，再取次上层（B）的像素值替换，以此类推。

关于 framebuffer 和 overlay 有一些有趣的话题，可以参考 [overlay 与 framebuffer 的区别](https://computergraphics.stackexchange.com/questions/5271/what-is-the-difference-in-overlay-and-framebuffer)，[SurfaceFlinger，framebuffer 和 overlay](https://www.mail-archive.com/android-porting@googlegroups.com/msg09615.html)





Garphics框架结构梳理
1 Overview
 
图1.1 Graphics整体框架
Android屏幕显示的机制通过向FrameBuffer写入内容来实现。如果只有一个FB，当APP写入速度>LCD显示速度时没问题；当APP写入速度<=LCD显示速度时，会卡顿和闪烁，为了解决这个问题，一般采用2个以上FB。对于Android系统来说，有很多个APP，如果这些APP同时向FB写入内容那显示的内容就乱了，因此通过SurfaceFlinger来管理。下图是Graphics的核心组件。
 
图1.2 Graphics的核心组件
	图像流生产者（Image Steam Producers）：一个图像流生产者表示可以产生图像buffer的任何组件。例如包括OpenGL ES、Canvas 2D和媒体服务器视频解码器。
	图像流消费者（Image Stream Consumers）：大部分情况下，通常的图像流消费者是SurfaceFlinger，SF是一个系统服务用于消费当前的可见surface并且将他们合成到display，前提是使用Window Manager提供的参数。SurfaceFlinger是唯一可以修改显示内容的服务。SurfaceFlinger使用OpenGL和HWC来来合成一组surface。其他的OpenGL ES应用也可以消耗图像流，例如Camera app消耗了camera预览图像流。非GL应用程序也可以是消费者，例如ImageReader类。
	窗口管理器（Window Manager）：控制窗口的Android系统服务，窗口是view的容器。一个窗口总是对应一个surface。这个服务监督窗口的生命周期，输入和焦点事件，屏幕方向，透明度，动画，位置，形变，z-order等等。窗口管理器发送所有的window元数据到SurfaceFlinger，从而SF可以使用这些数据来合成surface。
	硬件合成器（Hardware Composer）：显示子系统的硬件抽象。SurfaceFlinger可以授权一些特定的合成工作到硬件合成器，从而降低OpenGL和GPU的工作负担。
	图形内存分配器（Gralloc）：用于分配图像生成器请求的内存。
2 Framework Graphics
2.1 SurfaceFlinger
SurfaceFlinger是一个重要的系统服务，也是Android系统的一个关键进程，它最重要的作用就是合成surface，能够将不同应用的2D或者3D surface进行合成。通过底层的操作将合成后的数据发送到显示设备，最终显示在屏上。主要代码目录：frameworks/native/services/SurfaceFlinger。SurfaceFlinger主要做以下几件事情：
	给App提供buffer：通过gralloc模块向ashmen申请内存得到文件句柄fd，将fd通过binder机制传递给对应的app，app再执行mmap操作即可获得对应的buffer，如图2.1.1所示。
	将app发来的buffer（界面数据）进行合成：根据各个界面的layer（Z值，由WindowManager Service确定，如图2.1.2所示），把排序后的整体buffer传递给HardwareComposer。然后到Display环节，分为DPU合成和GSP合成，显示架构主要是DRM架构，最后输出到LCD上显示，该方式为HWC合成（Device）。
	通过Hardware UI绘制，由View确定。绘制部分通过OpenGL和Vulkan实现，App也可以直接调用EGL层接口来实现渲染功能，该方式为GPU合成（Client）。
 
图2.1.1 SurfaceFlinger整体框架
 
图2.1.2 合成surface示意图
2.1.1 Composition
SurfaceFlinger作为合成器，它分为GPU合成（Client）和HWC合成（Device）两种方式，根据不同场景和GSP/DPU能力去决定具体合成方式到HWComposer HAL，合成方式选择见表2.1，HWComposer根据具体场景和硬件能力返回SurfaceFlinger改变的compotionType作为最终结果。SurfaceFlinger在收集可见层的所有缓冲区之后，便会询问Hardware Composer应如何进行合成。
表2.1.1 SurfaceFlinger合成方式
合成方式	类型	描述
Device	Device	普通图层
	SolidColor	固定色，主要是透明效果
Client	Cusor	鼠标小箭头
	Client	GPU合成
Device合成是用专门的硬件合成器进行合成HWComposer，所以硬件合成的能力就取决于硬件的实现。其合成方式是将各个Layer的数据全部传给显示硬件，并告知它从不同的缓冲区读取屏幕不同部分的数据。HWComposer是Device合成的抽象。
Client合成方式是相对与硬件合成来说的，其合成方式是，将各个Layer的内容用GPU渲染到暂存缓冲区中，最后将暂存缓冲区传送到显示硬件。这个暂存缓冲区，我们称为FBTarget，每个Display设备有各自的FBTarget。Client合成，之前称为GLES合成，我们也可以称之为GPU合成。Client合成，采用RenderEngine进行合成。
2.1.2 Region
 
图2.1.3 各显示区域之间的关系
在Android系统中，定义了Region（一块任意形状的图形）的概念，它代表屏幕上的一个区域，它是由一个或多个Rect（矩形方块）组成的，这个区域有三种状态，可能是不可见的、部分可见的或者完全不可见的。图2.1.3中damage区域表示经常更新的区域。
2.1.3 Vsync
Vsync（Vertical Synchronization，垂直同步）是一种在PC上很早就广泛使用的技术，可以理解为是一种定时中断。SurfaceFlinger合成时机和app绘制时机都受到Vsync控制，Vsync也是分析是否掉帧的关键点。Android系统每隔16ms发出VSYNC信号，触发对UI进行渲染，屏幕的刷新过程是每一行从左到右（行刷新，水平刷新，Horizontal Scanning），从上到下（屏幕刷新，垂直刷新，Vertical Scanning）。当整个屏幕刷新完毕，即一个垂直刷新周期完成，会有短暂的空白期，此时发出VSync信号。
当没有VSync机制时，以下图为例，第一个16ms之内一切正常。第二个16ms之内因为CPU最后阶段才计算出了数据，导致GPU也是在第二段的末尾时间才进行了绘制，整个动作延后到了第三段内，从而影响了下一个画面的绘制。这时会出现Jank（闪烁，可以理解为卡顿或者停顿）。此时CPU和GPU可能被其他操作占用了，这就是卡顿出现的原因。
 
图2.1.4 没有VSync机制
当使用Vsync同步时，收到Vsync命令的时候，CPU和GPU才会必须立刻进行刷新/显示的动作。CPU/GPU接收vsync信号提前准备下一帧要显示的内容，所以能够及时准备好每一帧的数据，保证画面的流畅。如下所示：
 
图2.1.5 有VSync机制
当系统资源紧张性能降低时，导致GPU在处理某帧数据时太耗时，在Vsync信号到来时，buffer B的数据还没准备好，此时不得不显示buffer A的数据，这样导致后面CPU/GPU没有新的buffer准备数据，如下图，如果GPU忙不过来了，没有能及时完成，又会导致Jank，如下所示：
 
图2.1.6 系统资源紧张性能紧张时VSync机制
空白时间无事可做，后面Jank频出，因此又引入了triple buffering，如下所示：
 
图2.1.7 系统资源紧张性能紧张时triple buffering机制
当出现上面所述情况后，新增一个buffer C 可以减少CPU和GPU在Vsync同步间的空白间隙，此时CPU/GPU能够利用buffer C继续工作，后面buffer A 和 buffer B 依次处理下一帧数据。这样仅是产生了一个Jank，可以忽略不计，以后的流程就顺畅了。 
要注意以下2点：
	在多数正常情况下还是使用二级缓冲机制，三级缓冲只是在需要的时候才使用。
	三级缓冲并不是解决了jank的问题，而是出现jank时不影响后续的显示。
虽然vsync使得CPU/GPU/Display同步了，但App UI和SurfaceFlinger的工作显然是一个流水线的模型。即对于一帧内容，先等App UI画完了，SurfaceFlinger再出场对其进行合并渲染后放入framebuffer，最后整到屏幕上。而现有的VSync模型是让大家一起开始干活，这样对于同一帧内容，第一个VSync信号时App UI的数据开始准备，第二个VSync信号时SurfaceFlinger工作，第三个VSync信号时用户看到Display内容，这样就两个VSync period（每个16ms）过去了，影响用户体验。
解决这个问题的思路是：SurfaceFlinger在App UI准备好数据后及时开工做合成。Android 4.4（KitKat）中引入了VSync的虚拟化，即把硬件的VSync信号先同步到一个本地VSync模型中，再从中一分为二，引出两条VSync时间与之有固定偏移的线程。即App开始工作的时候是Vsync+offset1，surface工作的时刻在Vsync+offset1+offset2。而这个offset1和offset2都是可调的。示意图如下：
 
图2.1.8 VSync的虚拟化
产生Vsync的源头主要是硬件（HardwareComposer，一般）或者软件（VsyncThread）；由线程（DispSyncThread）来虚拟化Vsync信号，将Vsync分为Vsync-APP和Vsync-Surface，Vsync-App和Vsync-Surface是按需起作用的。
 
图2.1.9 APP需要更新界面和SF合成界面流程图
当APP需要更新界面时，发送请求Vsync信号请求给EventThread（APP），EventThread（APP）再向DispSyncThread线程发送请求，DispSyncThread线程收到请求后，延迟offset1后将VSync-APP发送给EventThread（APP）并唤醒，EventThread（APP）在收到Vsync-APP后唤醒APP，APP开始产生新界面（dequeueBuffer操作）。
APP将产生的界面提交Buffer时会调用queueBuffer的操作，最后会通知SF合成界面。SF需要合成界面的时候，发送请求Vsync信号请求给EventThread（SF），EventThread（SF）再向DispSyncThread线程发送请求，DispSyncThread线程收到请求后，延迟offset2后将Vsync-SF发送给EventThread（SF）并唤醒，EventThread（SF）在收到Vsync-SF后唤醒SF，SF开始合成新界面。
2.2 GraphicBuffer
2.2.1 BufferQueue
BufferQueue类将可生成图形数据缓冲区的组件（producers）连接到接受数据以便进行显示或进一步处理的组件（consumers）。几乎所有在系统中移动图形数据缓冲区的内容都依赖于BufferQueue。
Android显示系统是一个典型的生产者——消费者模式，其中，上层（APP）负责生产，而下层负责消费，而在其中每一层之间所传递的数据载体就是GraphicBuffer（图形缓冲区），而GraphicBuffer通过Gralloc进行申请。BufferQueue在Android图形组件之间起到粘合作用。这是一对queue，用于实现从生产者到消费者的恒定循环。一旦生产者交出buffer，SurfaceFlinger就承担起将所有东西合成到display的责任。下图是BufferQueue的通信过程。
 
图2.2.1 BufferQueue的通信过程
BufferQueue包含把图像流生产者和图像流消费者绑定到一起的逻辑。一些图像生产者的例子是camera预览，是由camera HAL或者OpenGL ES产生的。一些图像消费者的例子是SurfaceFlinger或者某些显示OpenGL ES流的app，例如camera app显示camera取景器（viewfinder）。BufferQueue是一个将buffer池（pool）和一个队列组合在一起的数据结构，并且使用Binder IPC来传递buffer。BufferQueue通常用于渲染到Surface、使用GL Consumer来消耗等其他任务。下图为Android的图形数据流，左侧是渲染器生产的graphics buffer，例如home界面、状态栏和system UI。SurfaceFlinger是排版者，HWC是合成者。
 
图2.2.2 Android的图形数据流
2.2.2 Fence
Fence（栅栏）机制是一种资源同步机制，主要用于处理跨硬件场景，如CPU、GPU和HWC之间的buffer资源同步，即图形系统中GraphicBuffer的同步。可以将fence理解为一种资源锁。一组线程能够使用fence来集体进行相互同步。本质上每个线程在到达某种状态时调用fence的wait()方法，阻塞等待其他全部参与线程调用wait()方法表明它们也到达了这个状态。一旦全部的线程都到达fence，它们就会集体解除阻塞，并一起继续运行。Fence可以实现在producer和consumer对buffer处理的过程中是协调同步从而保证buffer内容准确且不会被篡改。从下图可以发现，一个buffer的状态变化是：Free—Dequeued—Queued—Acquired—Free。
 
图2.2.3 Buffer状态变化
接下来对Buffer的四种状态进行说明：
	Buffer处于FREE状态时，producer并不能直接申请它，此时需要等一个signal（NO_FENCE信号），由于有可能上一次申请的buffer正在被consumer消费中，所以要等待consumer发出finish信号，此时FREE状态下的buffer就好像被栅栏fence拦住了，这里是用Fence中wait()或者waitForever()方法。等一个NO_FENCCE信号，栅栏fence就会打开。进入到下一流程。
	Buffer处于DEQUEUED状态时，即producer申请的buffer从队列中出来但还没有进入队列或者取消buffer。这个状态下producer想向Buffer填入UI数据时，必须等一个NO_FENCE信号，因为有可能其它owner正在对它进行操作。当信号到达时poducer就能够对其进行操作，操作完毕后再发出一个NO_FENCE信号。
	Buffer处于QUEUED状态时，即把buffer加入队列，在这个操作前必须等dequeueBuffer完毕之后发的NO_FENCE信号，收到信号后才进行入队列操作或者取消buffer操作。这个时候它的owner就变成BufferQueue了。
	Buffer处于ACQUIRED状态时，即producer已经对buffer填充完毕，也要等一个NO_FENCE信号，然后consumer才干对其进行操作。操作完毕后会释放buffer，然后发出一个NO_FENCE信号。
2.3 Hardware Composer
2.3.1 Overview
Composer作为一个独立的关键进程，主要作用是接收SurfaceFlinger传递过来的数据，采用合适的方式进行合成，并将framebuffer数据通过显示框架（如：DRM，ADF，FB等）送屏显示输出。与Gralloc一样，Composer作为HAL层，在Android版本升级时同样对它有严格的要求，需要升级到指定的版本，否则会导致XTS fail。
HWC（HWComposer）是Android中进行窗口（Layer）合成和显示的HAL层模块（注意：不是SurfaceFlinger代码中HWComposer这个类），通常由显示设备制造商（OEM）实现并完成，为SurfaceFlinger服务提供硬件支持。SurfaceFlinger可以使用OpenGL ES合成Layer，这需要占用并消耗GPU资源。大多数GPU都没有针对图层合成进行优化，因此当SurfaceFlinger通过GPU合成图层时，应用程序无法使用GPU进行自己的渲染。而HWC通过硬件设备进行图层合成，可以减轻GPU的合成压力。
2.3.2 HWC pipeline
显示设备的能力千差万别，很难直接用API表示硬件设备支持合成的Layer数量，Layer是否可以进行旋转和混合模式操作，以及对图层定位和硬件合成的限制等。因此HWC描述上述信息的流程是这样的：
	SurfaceFlinger向HWC提供所有Layer的完整列表，让HWC根据其硬件能力，决定如何处理这些Layer。
	HWC会为每个Layer标注合成方式，表明是通过GPU还是通过HWC合成。
	SurfaceFlinger负责先把所有注明GPU合成的Layer合成到一个输出Buffer，然后把这个输出Buffer和其他Layer（注明HWC合成的Layer）一起交给HWC，让HWC完成剩余Layer的合成和显示。
2.3.3 HWC function
每个HWC硬件模块都应该支持以下能力：
	至少支持4个叠加层：状态栏、系统栏、应用本身和壁纸或者背景。
	叠加层可以大于显示屏，例如：壁纸
	同时支持预乘每像素（per-pixel）Alpha混合和每平面（per-plane）Alpha混合。
	为了支持受保护的内容，必须提供受保护视频播放的硬件路径。
	RGBA packing order, YUV formats, and tiling, swizzling, and stride properties
	部分专业词汇说明：
	Tiling：可以把Image切割成MxN个小块，最后渲染时，再将这些小块拼接起来，就像铺瓷砖一样。
	Swizzling：一种拌和技术，表示向量单元可以被任意地重排或重复。
2.4 HWUI
HWUI硬件加速绘制模型，主要就是采用OpenGL来进行GPU硬件绘图，以此来提升绘图的速率，从而提高图形显示的性能。
3 OpenGL&Vulkan
OpenGL（Open Graphics Library）是用于渲染2D、3D矢量图形的跨语言、跨平台的应用程序编程接口（API）。
Vulkan也是一个跨平台的2D和3D绘图应用程序接口（API）。相对于OpenGL，Vulkan可以节约大量的CPU开销，提升应用的性能。
4 Debug
	开发者选项中“停用HW叠加层”
	dump SurfaceFlinger
	截屏/录屏
	dump layer和framebuffer
	停用HWUI
	抓取systrace
	抓取Gapid/RenderDoc
	SDK HierarchyView
 
5 Graphics architecture
5.1 Low-level components
5.1.1 BufferQueue and gralloc
BufferQueue将生成图形数据缓冲区的东西（生产者）连接到接受数据以供显示或进一步处理的东西（消费者）。缓冲区分配是通过gralloc内存分配器执行的，后者通过vendor-specific的HAL接口实现。
5.1.2 SurfaceFlinger, Hardware Composer, and virtual displays
SurfaceFlinger接受来自多个来源的数据缓冲区，将它们合成，并将它们发送给display。HWC确定可用硬件组合缓冲区的最有效方法，virtual displays使系统内的合成输出可用（记录屏幕或通过网络发送屏幕）。
5.1.3 Surface, canvas, and SurfaceHolder
Surface产生一个缓冲区队列，该队列通常由SurfaceFlinger消耗。当渲染到一个Surface上时，结果最终在一个缓冲区中被传送给consumer。Canvas API提供了一个软件实现（带有硬件加速支持），可以直接在表面上绘图（OpenGL ES的低级替代方案）。任何与视图有关的东西都涉及到一个SurfaceHolder，它的API允许获取和设置表面参数，比如大小和格式。
5.1.4 EGLSurface and OpenGL ES
OpenGL ES (GLES)定义了一个图形渲染API，旨在与EGL相结合，EGL是一个可以通过操作系统创建和访问窗口的库（绘制纹理多边形，使用GLES调用；在屏幕上呈现，使用EGL call）。
5.1.5 Vulkan
Vulkan是一个低开销、跨平台的高性能3D图形API。与OpenGL ES一样，Vulkan提供了在应用程序中创建高质量实时图形的工具。Vulkan的优点包括减少CPU开销和支持SPIR-V二进制中间语言。
5.2 High-level components
5.2.1 SurfaceView and GLSurfaceView
SurfaceView结合了surface和view。SurfaceView的视图组件由SurfaceFlinger（而不是应用程序）合成，使渲染从一个单独的线程/进程和应用程序UI渲染隔离。GLSurfaceView提供了辅助类来管理EGL上下文、线程间通信和与活动生命周期的交互（但不需要使用GLES）。
5.2.2 SurfaceTexture
SurfaceTexture结合了surface和GLES texture来创建BufferQueue，app是它的消费者。当一个生产者将一个新的缓冲区放入队列时，它会通知app，然后app释放之前持有的缓冲区，从队列中获取新的缓冲区，并使EGL调用使缓冲区作为外部texture可供GLES使用。
5.2.3 TextureView
TextureView结合了view和SurfaceTexture。TextureView封装了一个SurfaceTexture，负责响应回调和获取新的缓冲区。绘图时，TextureView使用最近收到的缓冲区的内容作为数据源，在view状态指示的任何地方和方式呈现。视图组合总是使用GLES执行的，这意味着对内容的更新可能会导致其他视图元素也被重新绘制。
Reference
https://source.android.com/docs/core/graphics
https://blog.csdn.net/vviccc/article/details/104860616
















































