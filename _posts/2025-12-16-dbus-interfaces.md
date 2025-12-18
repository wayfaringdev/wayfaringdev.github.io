---
title: dbus-interfaces
date: 2025-12-15 16:51:30 +0800
categories: [Linux, OpenBMC]
tags: [openbmc, dbus]     # TAG names should always be lowercase
author: baiqing
description: OpenBMC 中 dbus-interface 的运行原理
toc: true
mermaid: true
pin: false
---

## dbus interfaces 和 dbusplus 的协同工作

dbus interface 定义了一个dbus对象的具体内容格式，而 dbusplus 提供实现这些规范的开发工具。
在 yoctor 工程中，开发者可以使用 dbus interface 的 XML 文件描述服务接口，并通过 dbusplus 的工具生成代码框架。生成的代码可以直接集成到工程中，减少手动编写 D-Bus 相关代码的工作量。
dbusplus 还为信号和属性提供了事件驱动的编程模型，开发者可以注册回调函数处理信号或属性变化。这种机制在 yoctor 工程中可以用于实现模块间的异步通信和状态同步

简单来说，dbus interface 提供图纸，dbusplus 提供工具，yoctor 构建工程时 实用工具参考图纸生成建筑材料，当我们要构建一个service时，就可以直接使用建筑材料了。下面分析一下他们的工作流程：

## dbus-interfaces 提供dbus service的图纸

在openbmc 工程中解压phosphor-dbus-interfaces:
可以找到这个包下面 yaml目录，这里是yaml 文件开大会的地方，包括了很多常用的结构：

```yaml
./yaml/xyz/
└── openbmc_project
    ├── Association
    ├── Attestation
    ├── BIOSConfig
    ├── Certs
    ├── Channel
    ├── Chassis
    ├── Collection
    ├── Common
    ├── Condition
    ......
```

以phosphor-dbus-interfaces/yaml/xyz/openbmc_project/State/Chassis.interface.yaml 为例，

```yaml
description: Implement to provide the chassis power management

properties:
    - name: RequestedPowerTransition
      type: enum[self.Transition]
      default: "Off"
      description: >
          The desired power transition to start on this chassis. This will be
          preserved across AC power cycles of the BMC.
      errors:
          - xyz.openbmc_project.State.Chassis.Error.BMCNotReady
          - xyz.openbmc_project.Common.Error.Unavailable

    ......

enumerations:
    - name: Transition
      description: >
          The desired power transition for the chassis
      values:
          - name: "Off"
            description: >
                Chassis power should be off
          - name: "On"
            description: >
                Chassis power should be on
          - name: "PowerCycle"
            description: >
                Chassis power should be cycled from off to on. There will be a 5
                second delay between the off and the on.

    ......

paths:
    - namespace: /xyz/openbmc_project/state
      segments:
          - name: Chassis
            value: chassis
```

这个yaml 文件名中，%i.`interface`.yaml 声明了这是一个描述interface 的yaml

我们可以在这个文件中添加method/signal/property 等属性，还可以通过使用枚举对这里面的属性值进行限制。
(这个仓库的 README 描述的更详细些：[**README**](https://github.com/openbmc/phosphor-dbus-interfaces?tab=readme-ov-file#phosphor-dbus-interfaces) )
上面这个文件经过dbusplus处理后，在dbus上创建对象后 dbus 上的数据就是这样的：

```txt
Service：xxxx service
PATH：/xyz/openbmc_project/state/chassis0
interface：xyz.openbmc_project.State.Chassis
property：.CurrentPowerState/.LastStateChangeTime/.RequestedPowerTransition
```

通过busctrl可以获取对应的属性信息：

```bash
busctl introspect xyz.openbmc_project.Chassis.Buttons /xyz/openbmc_project/state/chassis0
xyz.openbmc_project.State.Chassis   interface -         -                                        -
.CurrentPowerState                  property  s         "xyz.openbmc_project.State.Chassis.Po... emits-change
.LastStateChangeTime                property  t         xxxxx                                    emits-change
.RequestedPowerTransition           property  s         "xyz.openbmc_project.State.Chassis.Tr... emits-change writable
```

## dbusplus 提供生成对象的工具

在openbmc 工程中，解压 dbusplus 这个配方文件，可以看到dbusplus的主要目录结构：

```yaml
.dbusplus 中主要目录
├── docs
├── example
├── include
├── meson.build
├── meson.options
├── src # 这里是sdbusplus 的源文件，向工程中提供dbus 的C++ API
├── test
└── tools # 这里 是工具，向dbus-interface 提供yaml 转换 的工具
    ├── meson.build
    ├── sdbus++
    ├── sdbus++-gen-meson
    ├── sdbusplus
    └── setup.py
```

src
: src目录中的API 包括dbus 的同步/异步调用，回调方法的实现。

tools
: tools目录中 sdbus++-gen-meson 是生成编译文件 meson 的工具，他会在 ./yaml/xyz/ 中每个目录下的文件生成一个meson.

下面是 sdbus++-gen-meson 生成的meson, 可以看到他的功能就是接收yaml， 生成C++类：

```bash
# Generated file; do not modify.
# phosphor-dbus-interfaces/gen/xyz/openbmc_project/State/Chassis/meson.build
sdbusplus_current_path = 'xyz/openbmc_project/State/Chassis'

generated_sources += custom_target(
    'xyz/openbmc_project/State/Chassis__cpp'.underscorify(),
    input: [
        '../../../../../yaml/xyz/openbmc_project/State/Chassis.errors.yaml',
        '../../../../../yaml/xyz/openbmc_project/State/Chassis.interface.yaml',
    ],
    output: [
        'error.cpp',
        'error.hpp',
        'common.hpp',
        'server.hpp',
        'server.cpp',
        'aserver.hpp',
        'client.hpp',
    ],
    depend_files: sdbusplusplus_depfiles,
    command: [
        sdbuspp_gen_meson_prog, ## 这里就是指定 sdbus++ 工具，生成从cpp文件，并且指定输出目录
        '--command',
        'cpp',
        '--output',
        meson.current_build_dir(),
        '--tool',
        sdbusplusplus_prog,
        '--directory',
        meson.current_source_dir() / '../../../../../yaml',
        'xyz/openbmc_project/State/Chassis',
    ],
    ......
)
```

## yaml 到 cpp 的蜕变

进行到现在，yaml 文件有了，meson 有了，该执行meson 去生成 C++ 类了：

meson 会执行sdbus++（python工具）把yaml 转换为cpp
可以在这里找到生成的文件：

```txt
phosphor-dbus-interfaces/oe-workdir/build/gen/xyz/openbmc_project/State/Chassis/：
.
├── aserver.hpp
├── client.hpp
├── common.hpp
├── error.cpp
├── error.hpp
├── server.cpp
└── server.hpp
```

common.hpp:
 定义了一个结构体，描述 Chassis的 interface, 实现一些基础的字符串转换功能。

server.hpp
: 定义 Chassis 类 和 包含其中的成员方法/变量等。

server.cpp
: 实现了 Chassis class 的虚函数，主要是获取/设置 interface中的属性的实现方法，这些虚函数需要派生类来实现。

## 使用构建好的dbus材料

上面 sdbus++ 已经生成了cpp 的类，我们可以在其他应用层中使用这些类：

假设我们现在已经有了一个service：xyz.openbmc_project.Chassis.Buttons

在这个service 中，我们创建一个 PowerButtons 类, 继承工具中的Chassis类，（这个类需要用 sdbusplus::server::object_t 模板套一下， 这个模板的主要作用是向dbus 注册interface 和 property），构造函数中使用service 和path 对积累进行构造。

举个例子：

```cpp
using ChassisInherit =
    sdbusplus::server::object_t<sdbusplus::server::xyz::openbmc_project::State::Chassis>;
......
class PowerButtons :
    public ChassisInherit 
{
......
    BackplaneInherit(*conn, Path.c_str()),
```

当我们实例化PowerButtons 的时候，也会调用基类的构造函数，object_t 模板中的类会注册interface 和 property，同时绑定基类中属性对应的设置/获取 方法。

此时dbus 的情况：

```bash
admin@SMBE327B001:~# busctl introspect xyz.openbmc_project.Chassis.Buttons /xyz/openbmc_project/state/chassis0
xyz.openbmc_project.State.Chassis   interface -         -                                        -
.CurrentPowerState                  property  s         "xyz.openbmc_project.State.Chassis.Po... emits-change
.LastStateChangeTime                property  t         xxxxx                                    emits-change
.RequestedPowerTransition           property  s         "xyz.openbmc_project.State.Chassis.Tr... emits-change writable
```

当我们通过dbus 调用 set/get property 时，会最终调用 Chassis类中 set/get property 的方法。

这里我们还可以在派生类中重写基类的方法，在重写时，还可以调用操作硬件的成员方法，再调用基类中的操作dbus的方法，就可以自动的将dbus 上的数据变动与硬件操作进行绑定。

## End

上面可以看出来，这套流程和我们自己注册interface，绑定属性的操作函数 逻辑上是一样的，dbus-interface 为我们提供了一个通用的 可扩展的流程，让程序员更少的关注Code，更多的关注业务逻辑实现。
