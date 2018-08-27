---
  title: Cabac编码
  tags: H264
  notebook: 多媒体
---

# Cabac编码

> Cabac: 全称`Context-based Adaptive Binary Arithmetic Coding`，基于上下文的自适应算术编码。是`h264`两种熵编码中的其中一种。由名字可以看出来，`Cabac`的核心是算术编码。

## 二值化

> Cabac在进行算术编码前需要先将`slice data`中的非二进制的语法元素转换成二进制串。这个过程就叫做二值化。

二值化的方案一共有7种：
- 一元码(Unary)
- 截断一元码(TU, Truncated Unary)
- K阶指数哥伦布编码(kth order Exp-Golomb, EGk)
- mb_type与sub_mb_type特有的查表方式
- 定长编码(FL, Fixed-Length)
- TU与EGk的联合二值化方案(UEGk，Unary/kth order Exp-Golomb)

### 一元码(Unary)

> 对任意一个无符号整数`x`, 一元码方案会将其转换为`x`个`1`结尾加一个`0`的二进制串。

例如： x = 4 , 一元码之后： 11110

下面是JM中一元码的实现：
```c
static void unary_bin_encode(EncodingEnvironmentPtr eep_dp,
                             unsigned int symbol,
                             BiContextTypePtr ctx,
                             int ctx_offset)
{
    if (symbol==0) {
        biari_encode_symbol(eep_dp, 0, ctx );
        return;
    } else {
        biari_encode_symbol(eep_dp, 1, ctx );
        ctx += ctx_offset;
        while ((--symbol) > 0)
            biari_encode_symbol(eep_dp, 1, ctx);
        biari_encode_symbol(eep_dp, 0, ctx);
    }
}
```

### 截断一元码(TU, Truncated Unary)

> 一元码的变体，用在已知语法元素最大`cMax`的情况下，对于`0 <= x < cMax`的范围的语法元素，使用一元码进行二值化。对于`x = cMax`，其二值化串全部由一组成，长度为`cMax`。

例如：cMax=5 二值化结果：11111

下面是JM中截断一元码的实现：
```c
static void unary_bin_max_encode(EncodingEnvironmentPtr eep_dp,
                                 unsigned int symbol,
                                 BiContextTypePtr ctx,
                                 int ctx_offset,
                                 unsigned int max_symbol)
{
    if (symbol==0) {
        biari_encode_symbol(eep_dp, 0, ctx );
        return;
    } else {
        unsigned int l = symbol;
        biari_encode_symbol(eep_dp, 1, ctx );

        ctx += ctx_offset;
        while ((--l)>0)
            biari_encode_symbol(eep_dp, 1, ctx);
        if (symbol < max_symbol)
            biari_encode_symbol(eep_dp, 0, ctx);
    }
}
```

### K阶指数哥伦布编码(kth order Exp-Golomb, EGk)

> 指数哥伦布编码由前缀和后缀组成。其中前缀部分由$l(x) = [\log_{2}(2^{k}+1)]$的值所对应的一元码组成；后缀部分可以通过使用长度`k+l(x)`为的`x+2k(1-2l(k))`的二进制计算。码元结构和标准的指数哥伦布有些区别。

下面是JM中对于指数哥伦布二值化的实现
```c
static void exp_golomb_encode_eq_prob( EncodingEnvironmentPtr eep_dp,
                                       unsigned int symbol,
                                       int k)
{
    for(;;) {
        if (symbol >= (unsigned int)(1<<k)) {
            biari_encode_symbol_eq_prob(eep_dp, 1);   //first unary part
            symbol = symbol - (1<<k);
            k++;
        } else {
            biari_encode_symbol_eq_prob(eep_dp, 0);   //now terminated zero of unary part
            while (k--)                               //next binary part
                biari_encode_symbol_eq_prob(eep_dp, ((symbol>>k)&1));
            break;
        }
    }
}
```

### 定长编码(FL, Fixed-Length)

> 用定长编码二值化语法元素， 要求语法元素最大值已知。其编码的二进制串的长度为$\log_{2}(cMax+1)$，其中的值就是语法元素的二进制值，多用于近似均匀分布的语法元素的二值化。

### TU与EGk的联合二值化方案（UEGk，Unary/kth order Exp-Golomb）

> 用一元截断码来编前缀，后缀采用EGk来编码， 不同语法元素的值采用不同的截断值和阶数。

- 运动矢量差的绝对值采用截断值为9的UEG3
- `level`采用是截断值为14的UEG0

JM中对于`level`和`mv`编码采用的是这种方案，如下：
```c
/*!
 ************************************************************************
 * \brief
 *    Exp-Golomb for Level Encoding
*
************************************************************************/
static void unary_exp_golomb_level_encode( EncodingEnvironmentPtr eep_dp,
        unsigned int symbol,
        BiContextTypePtr ctx)
{
    if (symbol==0) {
        biari_encode_symbol(eep_dp, 0, ctx );
        return;
    } else {
        unsigned int l=symbol;
        unsigned int k = 1;

        biari_encode_symbol(eep_dp, 1, ctx );
        while (((--l)>0) && (++k <= 13))
            biari_encode_symbol(eep_dp, 1, ctx);
        if (symbol < 13)
            biari_encode_symbol(eep_dp, 0, ctx);
        else
            exp_golomb_encode_eq_prob(eep_dp,symbol - 13, 0);
    }
}


/*!
 ************************************************************************
 * \brief
 *    Exp-Golomb for MV Encoding
*
************************************************************************/
static void unary_exp_golomb_mv_encode(EncodingEnvironmentPtr eep_dp,
                                       unsigned int symbol,
                                       BiContextTypePtr ctx,
                                       unsigned int max_bin)
{
    if (symbol==0) {
        biari_encode_symbol(eep_dp, 0, ctx );
        return;
    } else {
        unsigned int bin = 1;
        unsigned int l = symbol, k = 1;
        biari_encode_symbol(eep_dp, 1, ctx++ );

        while (((--l)>0) && (++k <= 8)) {
            biari_encode_symbol(eep_dp, 1, ctx  );
            if ((++bin) == 2)
                ++ctx;
            if (bin == max_bin)
                ++ctx;
        }
        if (symbol < 8)
            biari_encode_symbol(eep_dp, 0, ctx);
        else
            exp_golomb_encode_eq_prob(eep_dp, symbol - 8, 3);
    }
}

```

## Cabac编码总结

> 下面列出了`Cabac`编码的总体描述

- 编码器的初始化
- 编码决定
- 编码完`slice`中所有宏块的语法元素之后，写入`end_of_slice_flag`标志，然后进行字节压缩，在编码完一帧图像所有元素之后，所有输出的二进制都会进行封装，成为适合`NAL`层的传输单位。
- 对近似均匀分别的元素使用旁路编码