# YODAOS 灯光服务架构及客制化指引

## 概述

灯光分为2部分，一是上层的 lightd 应用框架部分，二是底层的渲染部分，也即是这个文档将要描述的部分。
上层应用框架将操作灯光的 api 进行了抽象，用户只需要知道有多少个灯，然后对灯进行操作就行了。默认每个灯都是 RGB 格式。
但是硬件这部分，除了需要知道有多少个灯，还要知道灯的格式。比如在 Me 上面是 RGB 灯，在 NABOO 上面是单色的 PWM 灯。

这部分文档需要描述的问题主要有以下几个

- 如何配置系统内置灯效
- 如何接收灯光应用框架传递过来的帧数据
- 如何将应用框架传递过来的帧数据转换为实际灯的格式，然后渲染。

## 配置系统灯效

系统内置的灯效文件默认存放在 /opt/light/ 下面，可在 lightd 服务中配置。

系统内置灯效文件路径配置方法：

1. 编辑文件: `/usr/yoda/services/lightd/service.js`。
2. 配置此字段的值 `var LIGHT_SOURCE = '/opt/light/'` 注意最后面需要一个斜杠 `/`。

从配置文件上，灯效文件可以分为2种:

1. 系统灯效文件。
2. 用户灯效文件。

### 系统灯效文件定义

系统灯效在 `${LIGHT_SOURCE}/config.json` 中配置。此文件的内容是json对象。key保存系统灯效的相对路径，以`config.json`文件为相对路径，值为该灯光的优先级。系统灯效的优先级数字越小，表示优先级越高，也就是0最高。

系统灯效维护着一个layer层，layer层是一个数组，里面保存了所有要恢复的灯光。每层只能有一个灯效，层的数字表示优先级。即一个系统灯光要恢复，会把该灯光保存到系统layer里。恢复的时候从0层开始查找。层的最大层数由config.json文件中的最大数字决定。

下面是一个 `config.json` 文件内容的例子:

```json
{
  "setVolume.js": 0,
  "setMuted.js": 1,
  "awake.js": 2,
  "longPressMic.js": 2,
  "loading.js": 3,
  "setPickup.js": 3,
  "setSpeaking.js": 4,
  "bluetoothOpen.js": 3,
  "bluetoothConnect.js": 3
}
```

只有在此文件中声明了的灯效文件才被定义为 `系统灯效`。

> 系统灯效的优先级为什么0是最大？

假设一下，如果是数字越大优先级越高。预先定义了10个灯效，如果现在要增加一个更低优先级的灯效，那么要把全部的数字往上加1。
系统定义的优先级在运行时是不会被改变的，一般在定义阶段，最高优先级的先定义，所以很自然的，从0开始，1234比较符合顺序。

### 用户灯效文件定义

所有非系统灯效都被定义为用户灯效。也就是说，即使你的文件在系统目录下，但是不在`config.json`文件中声明，它还是用户灯效。

用户灯效无需定义，也没有预先声明的优先级，所以它的优先级是运行时，由调用方动态指定的。

用户灯效同样也有一个layer层。只不过这个层是数字越小，优先级越低，默认是0。恢复的时候，从最大层开始查找，直到0层。最大层数由lightd服务里面配置。如果调用的时候超出了范围，则会自动限定在范围内。比如超出了最大范围，则默认为最大层数。

用户layer层的最大层数配置方法:

1. 编辑文件: `/usr/yoda/services/lightd/service.js`
2. 配置此字段的值 `var maxUserspaceLayers = 3` 注意需要一个 `大于0的整数`

用户灯效的优先级数字越大，优先级越高。

> 用户灯效的优先级为什么0是最小?

假设一下，如果0是最大，那我调用一个用户灯效的时候，优先级应该传多少？
用户灯效的优先级是运行时动态指定的，想要调用一个优先级更高的灯效，自然而然数字就会加1，所以是从0开始。

## lightd 应用层工作流程

### lightd 架构

lightd 整体分为2部分：

 - 一是上层 JavaScript 应用框架，负责抽象硬件 api 和管理资源。
 - 二是底层硬件渲染库，负责将上层应用设置的灯效最终渲染到硬件上。

### led数据结构

在 JavaScript 应用框架的角度看，现在默认都是 RGB 数据格式，即使是单色 PWM 灯，也当做 RGB 操作，只是在将数据流传递给硬件渲染的时候，底层渲染库会自动把 RGB 转换为单色值。数据格式按照 RGB RGB... 顺序排列。每个通道的值为 8位整数。即每个通道的值在 0-255 之间。

下面以拥有 5 个 RGB 灯的硬件为例，看下真实的数据在 lightd 中是如何表示的。

5个灯，每个灯是 RGB 通道，所以 `frame` 帧的长度为 5 * 3 = 15，即 15 个字节，所以真实的数据结构是：

```c++
char* frame = new char[5 * 3]
```

> 名词解释：frame
> frame 表示刷新所有灯光一帧所需要的数据。

## 硬件层工作流程

####LED HAL参数配置

- LED驱动属性分类
	make menuconfig -> rokid -> Hardware Layer Solutions -> 
	这里有三种接口的led驱动：PWM格式，MCU侧I2C，ARM侧I2C(以kamino18为例)
- LED灯的数量
	make menuconfig -> rokid -> Hardware Layer Solutions ->
	由`BOARD_LED_NUMS`可以控制led灯的个数

-------

####LED HAL代码逻辑
- 标准的android hardware layer架构,代码目录：`kamino18/hardware/modules/leds/`
- HAL层代码基本格式
       static struct hw_module_methods_t led_module_methods = {
           .open = led_dev_open,
       };

       struct hw_module_t HAL_MODULE_INFO_SYM = {
           .tag = HARDWARE_MODULE_TAG,
           .module_api_version = LEDS_API_VERSION,
           .hal_api_version = HARDWARE_HAL_API_VERSION,
           .id = LED_ARRAY_HW_ID,
           .name = "ROKID PWM LEDS HAL",
           .methods = &led_module_methods,
       };
- HAL层的调用方法：
		hw_get_module(LED_ARRAT_HW_ID, (const struct hw_module_t **)&module);
- 处理应用层发过RGB数据
	第一入口函数`led_draw()`，主要实现以下功能：
	+ 如果是RGB格式的三色灯，直接按照对应顺序刷写进底层驱动
	+ 如果是PWM控制的单色灯，会响应把RGB数据通过`rgb_to_gray()`转换成灰度值发到驱动层
	+ 以上各色灯自动丢弃透明度值

