---
title: virtual-sensor
date: 2025-12-15 16:51:30 +0800
categories: [Linux, OpenBMC]
tags: [openbmc, sensor]     # TAG names should always be lowercase
author: baiqing
description: OpenBMC 中 virtual-sensor 的运行原理
toc: true
mermaid: true
pin: false
---

## OpenBMC : virtual Sensor 是什么

virtual Sensor 的主要工作任务是根据配置文件将几个物理上的sensor聚合成一个sensor放到dbus 上，提供给其他APP使用。

我们看 github 上是怎么描述这个模块的：

> <https://github.com/openbmc/phosphor-virtual-sensor?tab=readme-ov-file>
>
> phosphor-virtual-sensor reads the configuration file virtual_sensor_config.json from one of three locations
> By default the repository will install a sample config into (3).
> There are two types of data in this file.
>
> - virtual sensor configuration information
>
> - information to get a virtual sensor configuration from D-Bus
>

这里可以看到virtual Sensor 的配置文件路径和类型：

1. 直接来源于 virtual sensor 的配置文件
2. 来源于dbus，这一个配置最终来源于entity-manager 的配置文件，具体配置 git上也给出了 参考这里：<https://github.com/openbmc/entity-manager/blob/master/schemas/virtual_sensor.json>

首先明确：virtual Sensor 这个包是将几个sensor 取运算(均值/最小/最大/求和/中间值)后形成一个新的聚合sensor的服务

## virtual Sensor工作流程

### Main

从meson 中可以看到 main 文件是入口：

```cpp
    // Add the ObjectManager interface
    sdbusplus::server::manager_t objManager(bus,
                                            "/xyz/openbmc_project/sensors");

    // Create an virtual sensors object
    phosphor::virtual_sensor::VirtualSensors virtualSensors(bus);

    // Request service bus name
    bus.request_name("xyz.openbmc_project.VirtualSensor");

    // Run the dbus loop.
    bus.process_loop();
```

上面代码中 请求了service 的name 和 path，主要的工作任务在 virtualSensors 的构造函数中执行

### virtualSensors

virtual Sensor本身的dbus资源请求完，构造virtualSensors->createVirtualSensors
该任务在 createVirtualSensors  中执行：

1. 先读取配置文件，如果配置文件指向D-Bus那么从Dbus 上获取哪些sensor 需要处理，从而进行实例化处理
2. 如果配置文件不指向dbus，那么直接处理配置文件 中 需要处理的sensor name，实例化一个VirtualSensor

```cpp
auto data = parseConfigFile();
......
for (const auto& j : data)
    if (desc.value("Config", "") == "D-Bus")
        createVirtualSensorsFromDBus(intf);
    else
        auto virtualSensorPtr = std::make_unique<VirtualSensor>( bus, objPath.c_str(), j, name);
        virtualSensorPtr->updateVirtualSensor();
        virtualSensorPtr->emit_object_added();
    .......
```

VirtualSensor
: 参考dbus-interface 工作原理，通过实例化这个类，可以在dbus上注册对应的object，并将数据更新到dbus

updateVirtualSensor()
: 将不同的sensor 按照配置文件中规定的处理方式(各种运算)处理

emit_object_added()
: 将数据推送到dbus 上

### 构造VirtualSensor

在上一节中我们可以看到 构造VirtualSensor有两种方向：

1. createVirtualSensorsFromDBus 使用的构造函数，直接从dbus interface 上查找sensor
2. 在配置文件中根据sensor name，去 dbus 上获取 sensor 的值，下面以这种VirtualSensor构造为例：

#### 构造过程

```cpp
createThresholds(threshold, objPath);
ValueIface::maxValue(maxConf->get<double>());
ValueIface::minValue(minConf->get<double>());
associationIface->associations(assocs);
available(true);
functional(true);
.....
/*读取表达式*/
auto& ref = sensorConfig.at(exprKey);
......
/*读取配置文件中的固定参数*/
const auto& consParams = params.value("ConstParam", empty);
/*读取配置文件中的dbus 参数 这里就可以设置dbus 上的sensor*/
auto dbusParams = params.value("DbusParam", empty);
/*实例化一个指向dbus  上sensor 的对象*/
auto paramPtr = std::make_unique<SensorParam>(bus, path, *this);
```

上面与config 文件的交互就完成了，这里需要注意 SensorParam 构造时，还会构造成员DbusSensor，这个成员注册回调函数，在dbus 上的sensor value 等属性发生改变时触发 virtual sensor 的处理。

#### 表达式的解析

构造完VirtualSensor后，已经知道了要聚合哪些sensor，接下来要解析计算这些sensor的方法：

```cpp
/*使用ExptTk 库，注册运算符，*/
......
symbols.create_variable(paramName);// 变量符号表
symbols.add_function("minIgnoreNaN", funcMinIgnoreNaN); // 运算符号表
expression.register_symbol_table(symbols);
/*解析器解析表达式*/
if (!parser.compile(exprStr, expression))
```

这里使用了 [**ExptTk 库**](https://www.partow.net/programming/exprtk/index.html)，对配置文件中提供的表达式进行解析 。
ExptTk库运行和解析方法可以参考: [**ExprTK全面探索**](https://blog.csdn.net/weixin_49135598/article/details/144318015)

### 聚合sensor计算

根据debus 上是否包含运算表达式(calculationIfaces), 对表达式进行演算

```cpp
    auto val = (!calculationIfaces.contains(exprStr))
            /*从virtual sensor 配置文件读取配置的sensor 使用expression 计算*/
            ? expression.value()
            /*从entity-manager 上获取配置的sensor使用此函数计算*/
            : calculateValue(exprStr, paramMap);
```

### 聚合sensor value 更新

将演算好的值 推到dbus 上：

```cpp
setSensorValue(val);
```

至此virtual sensor这个配方目录的基本工作就完成了，通过dbus/config的方式将传感器聚合，有很多优点：

1. 屏蔽底层硬件差异，给用户更直观的显示服务器的信息。
2. 简化上层管理，可以综合一个部件的多个部位传感器，直接反应部件的状态。
3. 聚合sensor有更好的冗余和容错机制，某个部件缺失/损坏的情况下可以结合其他传感器推断整体状态。
4. 聚合sensor也减少了ipmi sensor的资源占用，提高ipmi的响应速度，减少负载。
