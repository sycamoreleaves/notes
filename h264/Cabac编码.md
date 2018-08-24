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

## Cabac编码总结

> 下面列出了`Cabac`编码的总体描述

- 编码器的初始化
- 编码决定
- 编码完`slice`中所有宏块的语法元素之后，写入`end_of_slice_flag`标志，然后进行字节压缩，在编码完一帧图像所有元素之后，所有输出的二进制都会进行封装，成为适合`NAL`层的传输单位。
- 对近似均匀分别的元素使用旁路编码