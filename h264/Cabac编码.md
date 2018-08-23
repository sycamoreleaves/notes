# Cabac编码

> Cabac: 全称`Context-based Adaptive Binary Arithmetic Coding`，基于上下文的自适应算术编码。是`h264`两种熵编码中的其中一种。由名字可以看出来，`Cabac`的核心是算术编码。

## Cabac编码总结
> 下面列出了`Cabac`编码的总体描述

- 编码器的初始化
- 编码决定
- 编码完`slice`中所有宏块的语法元素之后，写入`end_of_slice_flag`标志，然后进行字节压缩，在编码完一帧图像所有元素之后，所有输出的二进制都会进行封装，成为适合`NAL`层的传输单位。
- 对近似均匀分别的元素使用旁路编码