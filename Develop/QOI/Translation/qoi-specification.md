![QOI](qoi-specification/qoi-logo-black-framed.svg)

# [还可以图像（QUITE OK IMAGE）格式](https://qoiformat.org/qoi-specification.pdf)

规范**版本1.0**，2022.01.05——[qoiformat.org](qoiformat.org)——[Dominic Szablewski](http://twitter.com/phoboslab)



AOI文件包含14字节文件头，后随任意数量的数据“块”，以及8字节结束标识。

```c
 qoi_header {
    char     magic[4];   // 魔法字节"qoif"
    uint32_t width;      // 图像宽度像素数（大端序）
    uint32_t height;     // 图像高度像素数（大端序）
    uint8_t  channels;   // 3 = RGB, 4 = RGBA
    uint8_t  colorspace; // 0 = sRGB，包含线性alpha通道
                         // 1 = 所有通道均为线性
 };
```

colorspace和channel字段仅作提示用，不影响数据块的编码方式。

图像编码顺序为逐行处理，从左到右，从上到下。解码器和编码器都从`{r: 0, g: 0, b: 0, a: 255}`开始，将其作为上一像素值。当`width * height`指定的所有像素均被覆盖，则该图像视作完整。图像按以下方式编码：

- 前一个像素的重复；
- 指向之前见过的像素列表的索引；
- 与之前像素rgb的差值；
- 完整的rgb或rgba值。

颜色通道假设为没有被提前乘以透明通道（“un-premultiptied alpha”）。

编码器/解码器维护一个动态的`array[64]`（初始填充零），包含之前见过的像素。编码器/解码器见过的每一个像素都被填入该数组，通过颜色的散列值来决定填写位置。对于编码器，如果该位置的像素值与当前像素相同，则将索引值以`QOI_OP_INDEX`写入数据流。索引值的散列函数为：

```c
index_position = (r * 3 + g * 5 + b * 7 + a * 11) % 64
```

每个块以2或8比特的标签起始，后续为一系列数据位。块的位数可被8整除，即所有块都按字节对齐。数据位中编码的所有数据都将最高有效位放在左边。8比特标签优先于2比特标签，解码器需要先检查8位标签是否存在。

字节流末尾用7个`0x00`字节和一个`0x01`字节标识。

可能出现的块包括：

```
┌─ QOI_OP_RGB ────────────┬─────────┬─────────┬─────────┐
│         Byte[0]         │ Byte[1] │ Byte[2] │ Byte[3] │
│  7  6  5  4  3  2  1  0 │ 7 .. 0  │ 7 .. 0  │ 7 .. 0  │
│─────────────────────────┼─────────┼─────────┼─────────│
│  1  1  1  1  1  1  1  0 │   red   │  green  │  blue   │
└─────────────────────────┴─────────┴─────────┴─────────┘
```

8-bit 标签 b11111110
8-bit 红色通道值
8-bit 绿色通道值
8-bit 蓝色通道值

alpha值与上一像素保持一致。

```
┌─ QOI_OP_RGBA ───────────┬─────────┬─────────┬─────────┬─────────┐
│         Byte[0]         │ Byte[1] │ Byte[2] │ Byte[3] │ Byte[4] │
│  7  6  5  4  3  2  1  0 │ 7 .. 0  │ 7 .. 0  │ 7 .. 0  │ 7 .. 0  │
│─────────────────────────┼─────────┼─────────┼─────────┼─────────│
│  1  1  1  1  1  1  1  1 │   red   │  green  │  blue   │  alpha  │
└─────────────────────────┴─────────┴─────────┴─────────┴─────────┘
```

8-bit 标签 b11111111
8-bit 红色通道值
8-bit 绿色通道值
8-bit 蓝色通道值
8-bit 透明通道值

```
┌─ QOI_OP_INDEX ──────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│───────┼─────────────────│
│  0  0 │     index       │
└───────┴─────────────────┘
```

2-bit 标签 b00
6-bit 颜色索引数组的下标：0..63

有效的编码器不应连续生成两个具有相同索引的`QOI_OP_INDEX`块，应改用`QOI_OP_RUN`。

```
┌─ QOI_OP_DIFF ───────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│───────┼─────┼─────┼─────│
│  0  1 │  dr │  dg │  db │
└───────┴─────┴─────┴─────┘
```

2-bit 标签 b01
2-bit 红色通道与上一像素的差值 -2..1
2-bit 绿色通道与上一像素的差值 -2..1
2-bit 蓝色通道与上一像素的差值 -2..1

差值应用到当前各通道时使用循环计数，即`1 - 2`结果为`255`，而`255 + 1`结果为`0`。

数值以偏移量为`2`的无符号整型方式存储，例如`-2`储存为`0 (b00)`，`1`储存为`3 (b11)`。

alpha值与上一像素保持一致。

```
┌─ QOI_OP_LUMA ───────────┬─────────────────────────┐
│         Byte[0]         │         Byte[1]         │
│  7  6  5  4  3  2  1  0 │  7  6  5  4  3  2  1  0 │
│───────┼─────────────────┼─────────────┼───────────│
│  1  0 │   diff green    │   dr - dg   │  db - dg  │
└───────┴─────────────────┴─────────────┴───────────┘
```

2-bit 标签 b10
6-bit 绿色通道与上一像素的差值 -32..31
4-bit 红色通道差值减去绿色通道差值 -8..7
4-bit 蓝色通道差值减去绿色通道差值 -8..7

绿色通道用于标识颜色变化的通用方向，使用`6`比特编码。红色和蓝色通道（dr和db）基于它们自己的差值减去绿色通道差值，即：

```c
    dr_dg = (cur_px.r - prev_px.r) - (cur_px.g - prev_px.g)
    db_dg = (cur_px.b - prev_px.b) - (cur_px.g - prev_px.g)
```

差值应用到当前通道时使用循环计数，即`10 - 13`结果为`253`，而`250 + 7`结果为`1`。

数据以无符号整型方式存储，绿色通道使用偏移量`32`，红色和蓝色通道使用偏移量`8`。

```
┌─ QOI_OP_RUN ────────────┐
│         Byte[0]         │
│  7  6  5  4  3  2  1  0 │
│───────┼─────────────────│
│  1  1 │       run       │
└───────┴─────────────────┘
```

2-bit 标签 b11
6-bit 行程长度，重复之前的像素：1..62

行程长度使用`-1`偏移量存储。注意，长度为`63`和`64`的行程（`b111110`和`b111111`）是非法的，它们被`QOI_OP_RGB`和`QOI_OP_RGBA`标签占用。