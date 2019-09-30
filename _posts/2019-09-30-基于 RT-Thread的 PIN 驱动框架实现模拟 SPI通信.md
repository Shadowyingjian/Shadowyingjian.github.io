# 基于 RT-Thread 的 PIN 驱动框架实现模拟 SPI 通信（一）

## 简述

  由于最近的工作中需要使用到 SPI 驱动一块型号为 ST7789 的 LCD 屏，参考 RT-Thread 的官方文档中 SPI 的使用方式，通过简单的几步配置就可以使用硬件 SPI 驱动 LCD 屏了，不得不说 RT-Thread 这款嵌入式实时操作系统真的是为嵌入式开发者提供了很多造好的轮子，大大提高了开发者的开发过程。本来以为通过使用硬件 SPI 驱动 LCD 屏就可以完成任务的了，可是可是....被告知由于板子的硬件 SPI 被用作驱动外部 flash 了，所以需要使用 GPIO 模拟 SPI 驱动 LCD 屏。（此时的我有点凌乱了，貌似使用 GPIO 模拟 SPI 驱动 LCD 我还没有真正尝试过，心底里有点虚，但是任务安排了下来，只能硬着头皮埋头干就对了），于是便有了，接下来的过程。

## SPI 通信方式的知识点

### SPI 总述

  SPI 是一种允许一个主设备启动一个与从设备的同步通讯的协议，从而完成数据的交换。也就是说，SPI是一种规定好的通讯方式。这种通信方式的优点是占用端口较少，一般4根就够基本通讯了。同时传输速度也很高。一般来说要求主设备要有SPI控制器（但可用模拟方式），就可以与基于SPI的芯片通讯了。
  常见的SPI外围设备包括外部FLASH、PSRAM、网络控制器、LCD显示驱动器、A/D转换器和MCU等。
  SPI 的通信原理很简单，它需要至少4根线，事实上3根也可以。也是所有基于SPI的设备共有的，它们是SDI（数据输入），SDO（数据输出），SCK（时钟），CS（片选）。其中CS是控制芯片是否被选中的，也就是说只有片选信号为预先规定的使能信号时（高电位或低电位），对此芯片的操作才有效。这就允许在同一总线上连接多个SPI设备成为可能。
  接下来就负责通讯的3根线了。通讯是通过数据交换完成的，这里先要知道SPI是串行通讯协议，也就是说数据是一位一位的传输的。这就是SCK时钟线存在的原 因，由SCK提供时钟脉冲，SDI，SDO则基于此脉冲完成数据传输。数据输出通过SDO线，数据在时钟上沿或下沿时改变，在紧接着的下沿或上沿被读取。 完成一位数据传输，输入也使用同样原理。这样，在至少8次时钟信号的改变（上沿和下沿为一次），就可以完成8位数据的传输。

**值得注意的是：SCK 信号线能由主设备控制，从设备不能控制信号线。在一个基于 SPI 的设备中，至少有一个主控设备，这样的控制方式不适用于多处理器的无主控通信。**

**特点：**
* 1.与普通的串行通信不同，普通的串行通信一次连续传送至少8位数据，而 SPI 允许数据一位一位的传送，甚至允许暂停。因为 SCK 时钟线由主控设备控制，当没有时钟跳变时，从设备不采集或传送数据，即主设备通过对 SCK 时钟线的控制可以完成对通信的控制。
* 2.SPI 是一个数据交换协议：因为 SPI 的数据输入和输出线独立，所以允许同时完成数据的输入和输出。
* 3.不同的SPI设备的实现方式不尽相同，主要是数据改变和采集的时间不同，在时钟信号上沿或下沿采集有不同定义，需要参考相关器件的文档。

### SPI 的信号线

  SPI一共有4根信号线，再回顾下作用：

| 引脚名称标识 | 说明                                         |
| ------------ | -------------------------------------------- |
| SCLK         | Serial Clock，（串行）时钟                   |
| SDI（MISO）  | Master In Slave Out，主设备输入，从设备输出  |
| SDO（MOSI）  | Master Out  Slave In，主设备输出，从设备输入 |
| CS           | Chip Select，选中从设备，片选                |

### SPI 的相位和极性

专业术语说明：

CKPOL (Clock Polarity) = CPOL = POL = Polarity = （时钟）极性

CKPHA (Clock Phase) = CPHA = PHA = Phase = （时钟）相位

1、CPOL  极性

先解释下什么是SCLK时钟的空闲。SCLK空闲就是当SCLK在数发送8个bit比特数据之前和之后的状态，于此对应的，SCLK在发送数据的时候，就是正常的工作的时候，有效active的时刻了。    
简单的说，SPI的CPOL，表示当SCLK空闲的时候，其电平的值是低电平0还是高电平。
CPOL=0，时钟空闲idle时候的电平是低电平，所以当SCLK有效的时候，就是高电平，就是所谓的active-high 
CPOL=1，时钟空闲idle时候的电平是高电平，所以当SCLK有效的时候，就是低电平，就是所谓的active-low



2、CPHA  相位

相位，对应着数据采样是在第几个边沿（edge），是第一个边沿还是第二个边沿，0对应着第一个边沿，1对应着第二个边沿。 

CPHA=0，表示第一个边沿： 
对于CPOL=0，idle时候的是低电平，第一个边沿就是从低变到高，所以是上升沿； 
对于CPOL=1，idle时候的是高电平，第一个边沿就是从高变到低，所以是下降沿； 
CPHA=1，表示第二个边沿： 
对于CPOL=0，idle时候的是低电平，第二个边沿就是从高变到低，所以是下降沿； 
对于CPOL=1，idle时候的是高电平，第一个边沿就是从低变到高，所以是上升沿；

如上所述，CPOL和CPHA可构成4种组合，这就是常说的SPI四种传输模式——

CPOL=0, CPHA=0 
CPOL=0, CPHA=1
CPOL=1, CPHA=0 
CPOL=1, CPHA=1

可以参考下图的说明进行理解：

![SPI 总线数据传输时序](D:\shadowliang\blog\Shadowyingjian.github.io\img\post-image\SPI-01.png)

##  基于 RT-Thread 的 PIN 驱动框架模拟 SPI 驱动

Talk is chep,Let me show you the code ....

```
/*
利用 IO 模拟 SPI 注册到总线
 */

#include <rtthread.h>
#include <rthw.h>
#include <rtdevice.h>
#include <stdio.h>
#include <string.h>

//定义模拟 SPI 的 GPIO 引脚,根据不同板子进行设置 GPIO 
#define SOFT_SPI_MISO   (4)
#define SOFT_SPI_MOSI   (5)
#define SOFT_SPI_CS     (3)
#define SOFT_SPI_SCLK   (2)

struct soft_spi_dev
{
    struct rt_spi_bus *spi_bus;
};

static struct soft_spi_dev *spi_dev;

/* 设置 SPI 端口初始化 */
static void soft_spi_init()
{
    //设置为输出模式
    rt_pin_mode(SOFT_SPI_CS,PIN_MODE_OUTPUT);
    rt_pin_mode(SOFT_SPI_SCLK,PIN_MODE_OUTPUT);
    rt_pin_mode(SOFT_SPI_MOSI,PIN_MODE_OUTPUT);
    rt_pin_mode(SOFT_SPI_MISO,PIN_MODE_INPUT);

    //设置CPOL = 0
    rt_pin_write(SOFT_SPI_SCLK,PIN_LOW);
    rt_pin_write(SOFT_SPI_MOSI,PIN_LOW);
    rt_pin_write(SOFT_SPI_CS,PIN_HIGH);
}

/*设置从设备使能 */
static void soft_spi_cs_enable(int enable)
{
    //设置为输出模式
    rt_pin_mode(SOFT_SPI_CS,PIN_MODE_OUTPUT);
    if(enable)
    {
        rt_pin_write(SOFT_SPI_CS,PIN_LOW);
    }
    else
    {
        rt_pin_write(SOFT_SPI_CS,PIN_HIGH);
    }
}

/* 写一个字节 */
static void soft_spi_WriteByte(uint8 byte)
{
    int i;
     for(i=8;i>0;i--)
    {
        rt_pin_write(SOFT_SPI_SCLK,PIN_LOW);
        if(byte&0x80)
        {
            rt_pin_write(SOFT_SPI_MOSI,PIN_HIGH);
        }
        else
        {
            rt_pin_write(SOFT_SPI_MOSI,PIN_LOW);
        }
        byte<<=1;
        rt_pin_write(SOFT_SPI_SCLK,PIN_HIGH);
    }
    
    //rt_kprintf("write %d bytes...\r\n",count);
    rt_pin_write(SOFT_SPI_SCLK,PIN_LOW);
}

/* 读写一个字节 */
static uint8 soft_spi_ReadWriteByte(uint8 byte)
{
    uint8 rdata = 0;
    uint8 i = 0;
    

    for(i=8;i>0;i--)
    {
        rt_pin_write(SOFT_SPI_SCLK,PIN_LOW);
        if(byte&0x80)
        {
            rt_pin_write(SOFT_SPI_MOSI,PIN_HIGH);
        }
        else
        {
            rt_pin_write(SOFT_SPI_MOSI,PIN_LOW);
        }
        byte<<=1;
        rdata = rdata<<1;
        if(rt_pin_read(SOFT_SPI_MISO))        //读取数据
        {
            rdata |= 0x01;
        }

        rt_pin_write(SOFT_SPI_SCLK,PIN_HIGH);
        
    }
    rt_pin_write(SOFT_SPI_SCLK,PIN_LOW);  //空闲时把 SCK 拉低
    //rt_kprintf("rdata:0x%08X \n",rdata);
    

    return rdata;
}



/* 模拟SPI 配置 */
rt_err_t _soft_spi_configure(struct rt_spi_device *dev,struct rt_spi_configuration *cfg)
{
    return RT_EOK;
}
/* 模拟 SPI 传输数据  */
rt_uint32_t _soft_spi_xfer(struct rt_spi_device* device, struct rt_spi_message* message)
{
   
    struct rt_spi_configuration * config = &device->config;
    rt_uint32_t size = message->length;
    /* 设置 CS  */
    if(message->cs_take)
    {
        soft_spi_cs_enable(1);

    }
   
    const rt_uint8_t * send_ptr =  message->send_buf;
    rt_uint8_t * recv_ptr =  message->recv_buf;
    while(size--)
    {
        rt_uint8_t data = 0xFF;

        if(send_ptr != RT_NULL)
        {
            data = *send_ptr++;
            //rt_kprintf("send_ptr:%02x\n",data);
             // 发送数据 
            soft_spi_ReadWriteByte(data);
        }

       
        // 接收数据
        if(recv_ptr != RT_NULL)
        {
             // 发送数据 
            data = soft_spi_ReadWriteByte(data);
            *recv_ptr++ = data;
            //rt_kprintf("recv_ptr:%02x\n",data);
        }
    }
    /* 设置 release CS   */
    if(message->cs_release)
    {
        soft_spi_cs_enable(0);
    }
    //rt_kprintf("len=%d \n",message->length);
    return message->length;

}


static struct rt_spi_ops soft_spi_ops = 
{
    .configure = _soft_spi_configure,
    .xfer = _soft_spi_xfer
};

int rt_soft_spi_bus_register(char *name)
{
    int result = RT_EOK;
    struct rt_spi_bus *spi_bus = RT_NULL;

    if(spi_dev)
    {
       return RT_EOK; 
    }

    spi_dev = rt_malloc(sizeof(struct soft_spi_dev));
    if(!spi_dev)
    {
        rt_kprintf("[soft spi]:malloc memory for spi_dev failed\n");
        result = -RT_ENOMEM;
        goto _exit;
    }
    memset(spi_dev,0,sizeof(struct soft_spi_dev));

    spi_bus = rt_malloc(sizeof(struct rt_spi_bus));
    if(!spi_bus)
    {
        rt_kprintf("[soft spi]:malloc memory for spi_bus failed\n");
        result = -RT_ENOMEM;
        goto _exit;
    }
    memset(spi_bus,0,sizeof(struct rt_spi_bus));

    spi_bus->parent.user_data = spi_dev;
    rt_spi_bus_register(spi_bus, name, &soft_spi_ops);
    
    return result;



_exit:
    if (spi_dev)
    {
        rt_free(spi_dev);
        spi_dev = RT_NULL;
    }

    if (spi_bus)
    {
        rt_free(spi_bus);
        spi_bus = RT_NULL;
    }
    return result;
}

static struct rt_spi_device *soft_spi_device = RT_NULL;
int rt_soft_spi_device_init(void)
{
    int result = RT_EOK;

    rt_kprintf("[soft spi]:rt_soft_spi_device_init \n");

    if(soft_spi_device)
    {
        return RT_EOK;
    }
    soft_spi_device = rt_malloc(sizeof(struct rt_spi_device));
    if(!soft_spi_device)
    {
        rt_kprintf("[soft spi]:malloc memory for soft spi_device failed\n");
        result = -RT_ENOMEM;
    }
    memset(soft_spi_device,0,sizeof(struct rt_spi_device));

    /* 注册 SPI BUS */
    result = rt_soft_spi_bus_register("soft_spi");
    if(result != RT_EOK)
    {
        rt_kprintf("[soft spi]:register soft spi bus error : %d !!!\n",result);
        goto _exit;
    }

    /* 绑定 CS */
    result = rt_spi_bus_attach_device(soft_spi_device,"spi3","soft_spi",NULL);
    if(result != RT_EOK)
    {
        rt_kprintf("[soft spi]:attact spi bus error :%d !!!\n",result);
        goto _exit;
    }
    rt_kprintf("[soft spi]:rt_soft_spi_device init ok\n");
    return RT_EOK;

_exit:
    if(soft_spi_device)
    {
        rt_free(soft_spi_device);
        soft_spi_device = RT_NULL;
    }
    return result;
}
INIT_PREV_EXPORT(rt_soft_spi_device_init);
INIT_DEVICE_EXPORT(soft_spi_init);
MSH_CMD_EXPORT(soft_spi_init,soft_spi_init);
```

通过上面的代码，就可以实现利用 RT-Thread 的 PIN 设备驱动框架模拟 SPI ，并把模拟的 SPI 设备注册到 系统的 SPI bus 上，示例中为注册成`soft_spi`bus,并且把`spi3`绑定到 `soft_spi`bus上。

## 总结

  通过本文档，回顾了 SPI 是通信方式和实现了利用 RT-Thread 的 PIN 设备驱动框架实现模拟 SPI 的通信方式，欢迎感兴趣的朋友尝试一下上面的方法，如有不对，请留言更正。