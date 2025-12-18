---
title: Qemu run a oem i2c device
date: 2025-12-12 16:51:30 +0800
categories: [Linux, QEMU]
tags: [openbmc, qemu, i2c]     # TAG names should always be lowercase
author: baiqing
description: Qemu 中虚拟一个 自定义的 i2c 设备并在openbmc中进行交互
toc: true
mermaid: true
pin: false
---

OpenBMC 开发时会遇到新项目的板子还没出来，但是有些功能需要验证 的情况。尤其时对研究OpenBmc 的初学者，没有板子的情况下，实际读写一个设备 会对理解BMC 的逻辑有很大帮助。

在上面种种情况下，我们可以在qemu中写一个可以进行通信的 i2c 设备，在这个设备上我们配置数据把他变成DEV，也可以搭配其他sensor变成一个背板或者其他小卡。方便我们进行学习和工程前期的debug。

## 定义前期需求

直接进入正题，我们以虚拟一个小板为例：

小板中一般包含一些自定义功能的设备 来获取小板上 的硬件状态：

1. 温度sensor
2. EEprom 存储静态信息
3. Smbus 设备来反馈小板上的信息

## 创建一个QEMU 驱动

- hw/i2c/Kconfig 中添加一个驱动：

```c
config SMBUS_DEV
    bool
    select SMBUS
```

- hw/i2c/meson.build 中关联到驱动文件：

```c
i2c_ss.add(when: 'CONFIG_SMBUS_DEV', if_true: files('i2c_oem_dev.c'))
```

- Kconfig 里面选择上面创建的驱动：

> config ASPEED_SOC
    bool
    default y
    depends on TCG && ARM
    ...
    select SMBUS_DEV

- 创建一个自定义的驱动：hw/i2c/i2c_oem_dev.c

## 在创建的 dev 驱动中实现功能

### i2c设备自定义的必要结构

这里参考eeprom dev 驱动继承 SMBUS 的父属性：

```c
static const TypeInfo smbus_dev_types[] = {
    {
        .name          = TYPE_SMBUS_DEV,
        .parent        = TYPE_SMBUS_DEVICE,
        .instance_init = smbus_dev_initfn,        
        .instance_size = sizeof(SMBusDEVDevice),
        .class_init    = smbus_dev_class_initfn,
    },
};
```

SMBUS 的设备需要实现的方法 `dev_receive_byte` 和 `dev_write_data` , 分别对应收到一个master 的写数据 和 进行slave 的写数据：

```c
static void smbus_dev_class_initfn(ObjectClass *klass,const void *data)
{
    DeviceClass *dc = DEVICE_CLASS(klass);
    SMBusDeviceClass *sc = SMBUS_DEVICE_CLASS(klass);

    dc->realize = smbus_dev_realize;
    device_class_set_legacy_reset(dc, smbus_dev_reset);
    sc->receive_byte = dev_receive_byte;
    sc->write_data = dev_write_data;
    dc->vmsd = &vmstate_smbus_dev;
    /* Reason: init_data */
    dc->user_creatable = false;
}
```

需要实现的方法弄清楚后进行整体设计，我想要的 dev 是按照SMBUS 协议，进行字节读取的设备（block读取这个SMBUS实现不了），在收到master 的第一个字节当作命令，把命令对应的数据返回给i2c bus.

### 一个命令列表

实现如下的结构体数组，分别放入命令和命令对应的数据

```c
SMBusDEVReg DEV_Init_default[] = {
    // Cmd code     rw flag     data len     data len reserve +    data buf
    {  0x00,        RO,          1 ,        {0x00, 0x21}  },
    ......
    {  0xff,        RW,          1,         {0xff}  }, //END Cmd
    {}
};
```

### 接收命令

在qemu 驱动接收到数据时会执行 `dev_write_data` ，此时按照命令格式，把第一个字节当作命令，去列表中查找，找到对应的index后记录到节点信息`（SMBusDEVReg *data_all）`中.

如果第一个字节后面还有数据，那就表示这些数据时需要写进设备里面的，我们就把这些数据放到命令对应的缓存里面：这里code 中可以按照设备的逻辑做一些特殊处理，计算checksum之类的，此处忽略：

```c
static int dev_write_data(SMBusDevice *dev, uint8_t *buf, uint8_t len)
{
    SMBusDEVDevice *dev = SMBUS_DEV(dev);
    SMBusDEVReg *data_all = dev->data_all;
    int index = 0;
    dev->data_ptr = &data_all[0];

    dev->accessed = true;
#ifdef DEBUG
    printf("dev_write_byte: addr=0x%02x cmd=0x%02x type:%d\n",
           dev->i2c.address, buf[0],dev->devType);
#endif
    if(dev->devType == DEV_TYPE_BP){
        for(index = 0;index < dev->init_data_size; index++){
            if(data_all[index].cmd_code == buf[0] ){
                dev->data_ptr = &data_all[index];
                if(len > 1 && data_all[index].rw_flag == RW && 
                    len - 1 == data_all[index].data_len){
                    memcpy(&data_all[index].data_buf[1], &buf[1], data_all[index].data_len);
                }
                data_all[index].data_buf[data_all[index].data_len + 1] = cal_checksum(&data_all[index].data_buf[0], data_all[index].data_len + 1);
            }
            if(data_all[index].cmd_code == 0xff){
                break;
            }
        }
    }

    return 0;
}
```

### 返回数据

上面接受命令时已经记录了master 要进行读写的命令，在开始向i2c写数据的时候会执行.

dev_receive_byte  在这个函数中，我们根据记录到的命令，找到对应的缓存区，将缓存区中的数据作为返回值返回即可.

```c
static uint8_t dev_receive_byte(SMBusDevice *dev)
{
    SMBusDEVDevice *dev = SMBUS_DEV(dev);
    SMBusDEVReg *data_ptr = dev->data_ptr;
    uint8_t val = 0;

    dev->accessed = true;
    if(data_ptr){
        if(dev->buf_offset < data_ptr->data_len + 3){
            val = data_ptr->data_buf[dev->buf_offset++];
        }
    }
#ifdef DEBUG
    printf("dev_receive_byte: addr=0x%02x val=0x%02x\n",
           dev->i2c.address, val);
#endif
    return val;
}
```

> 注意 这个函数每次只返回一个字节，因此记录命令的同时还要记录返回到第几个字节，下次继续用.
{: .prompt-warning }

## QEMU注册设备

有了上面的驱动，我们在 自己的平台里面 就可以添加这个设备了.

以2700 为例, 在`static void ast2700_oem_i2c_init(AspeedMachineState *bmc)` 中添加设备：

```c
dev_mux = i2c_slave_create_simple(aspeed_i2c_get_bus(&soc->i2c, 9), "pca9548", 0x70);
dev = DEVICE(smbus_dev_init_one(pca954x_i2c_get_bus(dev_mux, 0), 0x30));
```

同时我们还可以搭配eeprom 和 温度sensor 直接模拟一个背板：

```c
object_property_set_int(OBJECT(dev), "temperature", 28000, &error_abort);
smbus_eeprom_init_one(pca954x_i2c_get_bus(dev_mux, 0), 0x50, oem_bmc_fruid);
```

这个背板上硬盘，假设是NVME, 它也是有SMBUS 协议通讯的，也可以通过自定义的数据来“假装”一个：

```c
smbus_eeprom_init_one(pca954x_i2c_get_bus(nvme_mux, 0), 0x53, oem_bmc_nvme_disk_VPD );
```

这样一套完整的 包含背板和硬盘的仿真就完成了，

## 注册oem 2700

把上面用到的AST2700 平台初始化的代码注册到Mechine 里面：

```c
static void aspeed_machine_ast2700_oem_class_init(ObjectClass *oc, const void *data)
{
    MachineClass *mc = MACHINE_CLASS(oc);
    AspeedMachineClass *amc = ASPEED_MACHINE_CLASS(oc);

    mc->desc = "Aspeed AST2700 EVB (Cortex-A35) -OEM";
    amc->soc_name  = "ast2700-a0";
    amc->hw_strap1 = AST2700_EVB_HW_STRAP1;
    amc->hw_strap2 = AST2700_EVB_HW_STRAP2;
    amc->fmc_model = "w25q01jvq";
    amc->spi_model = "w25q512jv";
    amc->num_cs    = 2;
    amc->macs_mask = ASPEED_MAC0_ON | ASPEED_MAC1_ON | ASPEED_MAC2_ON;
    amc->uart_default = ASPEED_DEV_UART12;
    amc->i2c_init  = ast2700_oem_i2c_init;
    mc->default_ram_size = 2 * GiB;
    aspeed_machine_class_init_cpus_defaults(mc);
}
......
{
        .name          = MACHINE_TYPE_NAME("ast2700-oem"),
        .parent        = TYPE_ASPEED_MACHINE,
        .class_init    = aspeed_machine_ast2700_oem_class_init,
},
```

接下来运行自己的openbmc代码。就可以愉快的在没有板子/背板/硬盘的情况下根据需求调试OB代码了。
