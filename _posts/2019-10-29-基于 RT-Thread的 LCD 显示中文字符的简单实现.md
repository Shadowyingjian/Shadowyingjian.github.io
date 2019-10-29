# 基于 RT-Thread 系统的 LCD 显示中文字符的简单实现

  根据上一篇文章，学会了利用基于 RT-Thread 的 PIN 框架实现模拟 SPI 的功能，利用模拟的 SPI 驱动 LCD 屏，在 RT-Thread 的官网中有相关的文档介绍，这里不做详细说明。

  接下来的文章是个人参考 “零死角玩转 STM32 -MINI版本”中的 LCD 触摸画板的例程中实现的在 LCD 屏上显示中文字符。

## 简述 

  中文字符的显示需要有中文字库，一般的 LCD 屏都没有带中文字库，如果需要在 LCD 屏上显示中文就需要在 MCU 的 flash 中放置字库，在显示中文时，从 flash 的字库文件中找到对应的字模，显示到  LCD 屏上。

  在本文中参考“零死角玩转 STM32 -MINI版本”中的 LCD 触摸画板的例程，并作适当的修改，会使用到 SD 卡，在 SD 卡中放置中文字库。



### 实现流程

```
#include <rtthread.h>
#include <rtdevice.h>
//添加文件操作使用需要的头文件
#include <dfs_posix.h>
//添加 lcd 显示使用的头文件
#include "drv_lcd.h"
#include "ascii.h"

#if defined RT_USING_LCD_TEST
#define macWIDTH_EN_CHAR 8   //英文字符宽度
#define macHEIGHT_EN_CHAR 16 //英文字符高度

#define macWIDTH_CH_CHAR 16  //中文字符宽度
#define macHEIGHT_CH_CHAR 16 //中文字符高度

#define mac_DispWindow_X_Star 0 //  起始点的 X 坐标
#define mac_DispWindow_Y_Star 0 //  起始点的 Y 坐标

#define mac_DispWindow_COLUMN 240                   // 总列数
#define mac_DispWindow_PAGE 240                     //总行数
static const char *file_location = "/sd/HZLIB.bin"; //字库文件的路径
/**
 *  LCD 上显示英文字符
 * usX : 在特定扫描的方向下字符的起始 X 坐标
 * usY : 在特定扫描的方向下字符的起始 Y 坐标
 * cChar : 要显示的英文字符
 * usColor_Background : 选择英文字符的背景色
 * usColor_Foreground : 选择英文字符的前景色
 * 返回值 ： 无
 */
void lcd_disp_char_en(uint16_t usX, uint16_t usY, const char cChar, uint16_t usColor_Background, uint16_t usColor_Foreground)
{
    uint8_t ucTemp, ucRelativePosition, ucPage, ucColum;

    ucRelativePosition = cChar - ' ';

    lcd_open_window(usX, usY, macWIDTH_EN_CHAR, macHEIGHT_EN_CHAR);

    for (ucPage = 0; ucPage < macHEIGHT_EN_CHAR; ucPage++)
    {
        ucTemp = ucAscii_1608[ucRelativePosition][ucPage];

        for (ucColum = 0; ucColum < macWIDTH_EN_CHAR; ucColum++)
        {
            if (ucTemp & 0x01)
            {
                lcd_write_half_word(usColor_Foreground);
            }
            else
            {
                lcd_write_half_word(usColor_Background);
            }
            ucTemp >>= 1;

        } /*写完一行 */
    }
    /*全部写完 */
}

/**
 * 在 LCD 上显示英文字符串
 * usX : 在特定扫描方向下字符串的起始 X 坐标
 * usY : 在特定扫描方向下字符串的起始 Y 坐标
 * pStr : 要显示的英文字符串的首地址
 * usColor_Background : 选择英文字符串的背景色
 * usColor_Foreground : 选择英文字符串的前景色
 * 返回值 : 无
 */

void lcd_disp_str_en(uint16_t usX, uint16_t usY, const char *pStr, uint16_t usColor_Background, uint16_t usColor_Foreground)
{
    while (*pStr != '\0')
    {
        if ((usX - mac_DispWindow_X_Star + macWIDTH_EN_CHAR) > mac_DispWindow_COLUMN)
        {
            usX = mac_DispWindow_X_Star;
            usY += macHEIGHT_EN_CHAR;
        }

        if ((usY - mac_DispWindow_Y_Star + macHEIGHT_EN_CHAR) > mac_DispWindow_PAGE)
        {
            usX = mac_DispWindow_X_Star;
            usY = mac_DispWindow_Y_Star;
        }

        lcd_disp_char_en(usX, usY, *pStr, usColor_Background, usColor_Foreground);
        pStr++;
        usX += macWIDTH_EN_CHAR;
    }
}

/**
 * 从 SD 卡中获取中文字库
 * *pBuffer : 数据指针 
 * c : 读取的字符
 */

int GetGBKCode_from_sd(uint8_t *pBuffer, uint16_t c)
{
    unsigned char Hight8bit, Low8bit;
    unsigned int pos;
    int fd, size, ret;
    int result = 0;

    Hight8bit = c >> 8;   /* 取高8位数据 */
    Low8bit = c & 0x00FF; /* 取低8位数据 */

    //rt_kprintf("%d,%d\r\n",Hight8bit,Low8bit);
    //rt_kprintf("%x,%x\r\n",Hight8bit,Low8bit);

    pos = ((Hight8bit - 0xa0 - 16) * 94 + Low8bit - 0xa0 - 1) * 2 * 16;

    /* 以只读模式打开 字库文件 */
    fd = open(file_location, O_RDONLY);
    if (fd >= 0)
    {
        //移动指针
        ret = lseek(fd, pos, 0);
        //rt_kprintf("lseek  = %d \n",ret);
        size = read(fd, pBuffer, 32);
        close(fd);
        if (size < 0)
        {
            rt_kprintf("read %s  file error!!!\n", file_location);
        }

        result = 0;
    }
    else
    {
        rt_kprintf("open %s file error!!!\n", file_location);
        result = -1;
    }
    return result;
}

/**
 * 在 lcd 上显示中文字符
 * x : 在特定扫描方向下字符串的起始 X 坐标
 * y : 在特定扫描方向下字符串的起始 Y 坐标
 * usChar : 要显示的字符
 * usColor_Background : 选择字符串的背景色
 * usColor_Foreground : 选择字符串的前景色
 * 
 */
void lcd_disp_char_ch(uint16_t x, uint16_t y, uint16_t usChar, uint16_t usColor_Background, uint16_t usColor_Foreground)
{
    uint8_t ucPage, ucColum;
    uint8_t ucBuffer[32];
    uint16_t us_Temp;

    lcd_open_window(x, y, macWIDTH_CH_CHAR, macHEIGHT_CH_CHAR);
    GetGBKCode_from_sd(ucBuffer, usChar); //取字模数据

    for (ucPage = 0; ucPage < macHEIGHT_CH_CHAR; ucPage++)
    {
        //取出两个字节的数据，在 LCD 上即是一个汉字的一行
        us_Temp = ucBuffer[ucPage * 2];
        us_Temp = (us_Temp << 8);
        us_Temp |= ucBuffer[ucPage * 2 + 1];

        for (ucColum = 0; ucColum < macWIDTH_CH_CHAR; ucColum++)
        {
            if (us_Temp & (0x01 << 15)) //高位在前
            {
                lcd_write_half_word(usColor_Foreground);
            }
            else
            {
                lcd_write_half_word(usColor_Background);
            }
            us_Temp <<= 1;
        }
    }
}

/*
 * LCD 显示中文字符串
 * usX : 在特定扫描方向下字符串的起始 X 坐标
 * usY : 在特定扫描方向下字符串的起始 Y 坐标
 * pStr : 要显示的字符串首地址
 * usColor_Background : 选择字符串的背景色
 * usColor_Foreground : 选择字符串的前景色
 */
void lcd_disp_str_ch(uint16_t usX, uint16_t usY, const uint8_t *pStr, uint16_t usColor_Background, uint16_t usColor_Foreground)
{
    rt_uint16_t usCh = 0;
    rt_uint8_t usCh_low = 0;
    rt_uint8_t usCh_high = 0;
    rt_uint16_t usCh_temp = 0xffff;

    while (*pStr != '\0')
    {
        if ((usX - mac_DispWindow_X_Star + macWIDTH_CH_CHAR) > mac_DispWindow_COLUMN)
        {
            usX = mac_DispWindow_X_Star;
            usY += macHEIGHT_CH_CHAR;
        }

        if ((usY - mac_DispWindow_Y_Star + macHEIGHT_CH_CHAR) > mac_DispWindow_PAGE)
        {
            usX = mac_DispWindow_X_Star;
            usY = mac_DispWindow_Y_Star;
        }
        usCh_low = *pStr;
        usCh_high = *(pStr + 1);
        usCh_temp = 0xffff & usCh_low;
        usCh_temp = usCh_temp << 8;
        usCh_temp = usCh_temp | usCh_high;

        lcd_disp_char_ch(usX, usY, usCh_temp, usColor_Background, usColor_Foreground);
        usX += macWIDTH_CH_CHAR;
        pStr += 2; //一个汉字两个字节
    }
}

/**
 * 显示中英文字符串
 * usX : 在特定扫描方向下字符串的起始 X 坐标
 * usY : 在特定扫描方向下字符串的起始 Y 坐标
 * pStr : 要显示的字符串首地址
 * usColor_Background : 选择字符串的背景色
 * usColor_Foreground : 选择字符串的前景色
 * 
 * 
*/
void lcd_disp_str_en_ch(uint16_t usX, uint16_t usY, const uint8_t *pStr, uint16_t usColor_Background, uint16_t usColor_Foreground)
{
    uint16_t usCh;
    rt_uint8_t usCh_low = 0;
    rt_uint8_t usCh_high = 0;
    rt_uint16_t usCh_temp = 0xffff;
    while (*pStr != '\0')
    {
        if (*pStr <= 126) //英文字符
        {
            if ((usX - mac_DispWindow_X_Star + macWIDTH_EN_CHAR) > mac_DispWindow_COLUMN)
            {
                usX = mac_DispWindow_X_Star;
                usY += macHEIGHT_EN_CHAR;
            }

            if ((usY - mac_DispWindow_Y_Star + macHEIGHT_EN_CHAR) > mac_DispWindow_PAGE)
            {
                usX = mac_DispWindow_X_Star;
                usY = mac_DispWindow_Y_Star;
            }

            lcd_disp_char_en(usX, usY, *pStr, usColor_Background, usColor_Foreground);

            usX += macWIDTH_EN_CHAR;

            pStr++;
        }
        else
        {
            if ((usX - mac_DispWindow_X_Star + macWIDTH_CH_CHAR) > mac_DispWindow_COLUMN)
            {
                usX = mac_DispWindow_X_Star;
                usY += macHEIGHT_CH_CHAR;
            }

            if ((usY - mac_DispWindow_Y_Star + macHEIGHT_CH_CHAR) > mac_DispWindow_PAGE)
            {
                usX = mac_DispWindow_X_Star;
                usY = mac_DispWindow_Y_Star;
            }

            usCh_low = *pStr;
            usCh_high = *(pStr + 1);
            usCh_temp = 0xffff & usCh_low;
            usCh_temp = usCh_temp << 8;
            usCh_temp = usCh_temp | usCh_high;

            lcd_disp_char_ch(usX, usY, usCh_temp, usColor_Background, usColor_Foreground);
            usX += macWIDTH_CH_CHAR;
            pStr += 2; //一个汉字两个字节
        }
    }
}

int test_lcd_ch(int argc, char *argv[])
{
    rt_err_t ret = RT_EOK;
    lcd_disp_str_ch(0, 50, "小而美的", WHITE, BLUE);
    lcd_disp_str_ch(0, 100, "物联网操作系统", BLACK, BLUE);
    lcd_disp_str_ch(0, 150, "小而美的物联网操作系统", GREEN, BLUE);
    lcd_disp_str_ch(100, 200, "一二三四五六七八九十", BLACK, WHITE);
    lcd_disp_str_en_ch(0, 170, "Hello,你好,小而美的物联网操作系统.", WHITE, BLACK);

    return ret;
}

MSH_CMD_EXPORT(test_lcd_ch, test_lcd_ch);

#endif
```

## 总结

  参考上面的代码实现过程，可以在 LCD 屏上显示中英文字符，并且实现了从裸机程序移植 RT-Thread 是多么的方便。