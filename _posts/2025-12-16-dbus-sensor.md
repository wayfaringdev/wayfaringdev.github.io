---
title: dbus-sensor
date: 2025-12-15 16:51:30 +0800
categories: [Linux, OpenBMC]
tags: [openbmc, sensor]     # TAG names should always be lowercase
author: baiqing
description: OpenBMC 中 dbus-sensor 的运行原理
toc: true
mermaid: true
pin: false
---


## dbus Sensor 的工作原理

Dbus-Sensor 是提供 xyz.openbmc_project 的传感器应用程序接口的集合。它们从 hwmon、 d-bus 或直接驱动访问读取传感器值以提供数据。还支持一些高级的非传感器特性，如风扇存在、 pwm 控制和自动 CPU 检测 (x86)。
这里是 dbus sensor git仓库的介绍<https://github.com/openbmc/dbus-sensors>

另外 这个service 工作还涉及到Entitymanager 的协同，这个service 的工作原理，参考这篇文章 //TBD
下面以HwMon 举例，看一下这些硬件监控服务的工作原理。

### HwMonTempMain

这是dbus sensor 集合中的一个，负责监控硬件设备上报告上来的温度，并放到debus 上

#### HwMon总体框架

服务入口：

dbus-sensors/src/hwmon-temp/HwmonTempMain.cpp

```cpp
boost::asio::io_context io;
......
boost::container::flat_map<std::string, std::shared_ptr<HwmonTempSensor>>
    sensors;
auto sensorsChanged =
    std::make_shared<boost::container::flat_set<std::string>>();

auto powerCallBack = [&sensors, &io, &objectServer,
                      &systemBus](PowerState type, bool state) {
    powerStateChanged(type, state, sensors, io, objectServer, systemBus);
};
setupPowerMatchCallback(systemBus, powerCallBack);

boost::asio::post(io, [&]() {
    createSensors(io, objectServer, sensors, systemBus, nullptr, false);
});

boost::asio::steady_timer filterTimer(io);
std::function<void(sdbusplus::message_t&)> eventHandler =
    [&](sdbusplus::message_t& message) {
        TRACE_EVENT_INSTANT("service", "Inventory_changed_eventHandler");
        .......
        sensorsChanged->insert(message.get_path());
        // this implicitly cancels the timer
        filterTimer.expires_after(std::chrono::seconds(1));

        filterTimer.async_wait([&](const boost::system::error_code& ec) {
            ......
            createSensors(io, objectServer, sensors, systemBus,
                          sensorsChanged, false);
        });
    };

std::vector<std::unique_ptr<sdbusplus::bus::match_t>> matches =
    setupPropertiesChangedMatches(*systemBus, sensorTypes, eventHandler);
```

上面代码中，关键操作如下：

setupPowerMatchCallback
: 注册了一个 主机电源状态改变后的回调函数；

setupPropertiesChangedMatches
: 注册了一个 回调函数 eventHandler: 当dbus 上 有inventory 信息的改变时（Entity-manager 加载或者修改某个组件）会触发eventHandler来处理消息来源，将消息中的path 插入到 sensorsChanged 交给createSensors 处理。插入到 sensorsChanged 中的path 即为 /xyz/openbmc_project/inventory/system/X/Y/TempXXX，表示一个硬件传感器设备。

Main的最后使用 interfaceRemoved 注册一个回调函数：当硬件移除时，hwmontempsensor 移除相关sensor。

> 此处 一个path 即代表了一个硬件传感器，由dts配置或者根据硬件自动加载，自动加载主要来源于Enity-manager。
{: .prompt-info }

例如 Entitymanager 加载如下配置：

```config
        {
            "Address": "0x4c",
            "Bus": "$sbus",
            "Name": "TempXXX",
            "Label": "temp2",
            "Type": "EMC1412"
        }
```

Entity-manager 会 在dbus object：

```shell
# busctl introspect xyz.openbmc_project.EntityManager /xyz/openbmc_project/inventory/system/X/Y/TempXXX
NAME                                                  TYPE      SIGNATURE RESULT/VALUE            FLAGS
org.freedesktop.DBus.Introspectable                   interface -         -                       -
.Introspect                                           method    -         s                       -
org.freedesktop.DBus.Peer                             interface -         -                       -
.GetMachineId                                         method    -         s                       -
.Ping                                                 method    -         -                       -
org.freedesktop.DBus.Properties                       interface -         -                       -
.Get                                                  method    ss        v                       -
.GetAll                                               method    s         a{sv}                   -
.Set                                                  method    ssv       -                       -
.PropertiesChanged                                    signal    sa{sv}as  -                       -
xyz.openbmc_project.Configuration.EMC1412             interface -         -                       -
.Address                                              property  t         xx                     emits-change
.Bus                                                  property  t         xx                     emits-change
.Label                                                property  s         "temp2"                 emits-change
.Name                                                 property  s         "TempXXX"       emits-change
.Type                                                 property  s         "EMC1412"               emits-change
```

#### createSensors

创建 sensor 的主要任务在这里执行，上面已经看到这个任务在进程启动后和每次资产信息改变时执行createSensors 的任务：

```cpp
void createSensors(
    boost::asio::io_context& io, sdbusplus::asio::object_server& objectServer,
    boost::container::flat_map<std::string, std::shared_ptr<HwmonTempSensor>>&
        sensors,
    std::shared_ptr<sdbusplus::asio::connection>& dbusConnection,
    const std::shared_ptr<boost::container::flat_set<std::string>>&
        sensorsChanged,
    bool activateOnly)
{
    TRACE_EVENT_BEGIN("service", "create_sensors", "activateOnly",
                      activateOnly);
    auto getter = std::make_shared<GetSensorConfiguration>(
        dbusConnection,
        [&io, &objectServer, &sensors, &dbusConnection, sensorsChanged,
         activateOnly](const ManagedObjectType& sensorConfigurations) {  ....Lamada callback... });
    std::vector<std::string> types(sensorTypes.size());
    for (const auto& [type, dt] : sensorTypes)
    {
        types.push_back(type);
    }
    getter->getConfiguration(types);
    TRACE_EVENT_END("service");
}
```

函数中 实例化了一个 GetSensorConfiguration，Lamada callback  是实例化时传进去的回调函数，在对象析构时调用。
通过调用 getConfiguration 方法遍历dbus 上 所有的可监控的硬件，GitHub 上对此这样描述：
> The dbus-sensors suite of daemons each run searching for a specific type of sensor. In this case the hwmon temperature sensor daemon will recognize the configuration interface: xyz.openbmc_project.Configuration.TMP441.
[My First Sensors](https://github.com/openbmc/entity-manager/blob/master/docs/my_first_sensors.md)

对每个可监控的dbus object ，获取dbus上的bus/address等属性上述获取的属性 数据会 传入到 callback 中。
Lamada 表达式捕获的参数中, sensorsChanged 主要作用是差异化变更，对状态有变动的sensor 做单独处理。

#### Lamada callback

上面的 GetSensorConfiguration 对象析构时会调用实例化时传进来的 lamada 表达式，这个回调函数可以简化为下面的代码：

```cpp
SensorConfigMap configMap =  buildSensorConfigMap(sensorConfigurations);
......
findFiles(fs::path("/sys/class/hwmon"), R"(temp\d+_input)", paths);
......
for (auto& path : paths)
    if (!getDeviceBusAddr(deviceName, bus, addr))
    ......
    auto findSensorCfg = configMap.find({bus, addr});
    ......
    if (!parseThresholdsFromConfig(sensorData, sensorThresholds,nullptr, &index))
    ......
    sensor->activate(*hwmonFile, i2cDev);
    ......
    sensor = std::make_shared<HwmonTempSensor>
    sensor->setupRead();
    ......
    while (true)
        ......
```

buildSensorConfigMap
: 获取dbus 所有 object 中的 path->config 组成 configMap .

findFiles-for
: 遍历所有 hwmon 下的所有路径，对每个可监控的 hwmon 都要监控

getDeviceBusAddr
: 获取 hwmon 路径中硬件设备的bus 和 addr，方便 dbus 上分辨这个设备指向哪里。

findSensorCfg
: 在 configMap 中查找对应bus 和 address 的sensor配置

parseThresholdsFromConfig
: 对查找到的sensor，从config 中获取阈值相关的属性

对于首次添加进来的sensor , 使用上面获取的 config 实例化 一个 HwmonTempSensor 进行监控, 对于已经开始监控的sensor，执行 activate 方法激活监控.

最后面的 循环 是遍历 config 中多个Name对应 hwmon 中多个输入的情况（cofig配置name、name1 name2，对应 hwmon 中temp1_input temp2_input temp3_input etc.）

#### HwmonTempSensor

HwmonTempSensor 主要进行硬件传感器信息监控，构造时会创建 sensor 各种属性对应的interface：

```cpp
{
    sensorInterface = objectServer.add_interface(
        "/xyz/openbmc_project/sensors/" + thisSensorParameters.typeName + "/" + name,  "xyz.openbmc_project.Sensor.Value");

    for (const auto& threshold : thresholds)
    {
        std::string interface = thresholds::getInterface(threshold.level);
        thresholdInterfaces[static_cast<size_t>(threshold.level)] =
            objectServer.add_interface( 
            "/xyz/openbmc_project/sensors/" + thisSensorParameters.typeName + "/" + name, interface);
    }
    setInitialProperties(thisSensorParameters.units);
    association = objectServer.add_interface( 
        "/xyz/openbmc_project/sensors/" + thisSensorParameters.typeName + "/" + name,  association::interface);
        ......
}
```

通过调用对象的 setupRead() 方法开始监控，从 path 中获取sensor value 的任务，并将 value 更新到dbus 上。

```cpp
void HwmonTempSensor::setupRead()
{
    if (!readingStateGood())
    {
        markAvailable(false);
        updateValue(std::numeric_limits<double>::quiet_NaN());
        TRACE_EVENT_INSTANT(
            "sensor", "hwmon_temp_sensor_status", "sensorName", name,
            "available", false, "reason",
            "sensor is not ready for reading since current state is not as expected",
            "expected_state", readState);
        restartRead();
        return;
    }

    std::weak_ptr<HwmonTempSensor> weakRef = weak_from_this();
    inputDev.async_read_some_at(
        0, boost::asio::buffer(readBuf),
        [weakRef](const boost::system::error_code& ec, std::size_t bytesRead) {
            std::shared_ptr<HwmonTempSensor> self = weakRef.lock();
            if (self)
            {
                self->handleResponse(ec, bytesRead);
            }
        });
}
```

经过上面的一系列操作，已经将传感器读值从硬件中读出来了，接下来通过下面这个链路实现定时读取：
setupRead -> handleResponse ->restartRead -> setupRead

初学OpenBMC 记录学习过程，请各位读者大佬斧正。
