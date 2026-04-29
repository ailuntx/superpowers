# Go Fractals CLI - 设计文档

## 概述

一个生成ASCII艺术分形的命令行工具。支持两种分形类型，并具有可配置的输出选项。

## 使用方法

```bash
# 谢尔宾斯基三角形
fractals sierpinski --size 32 --depth 5

# 曼德博集合
fractals mandelbrot --width 80 --height 24 --iterations 100

# 自定义字符
fractals sierpinski --size 16 --char '#'

# 帮助信息
fractals --help
fractals sierpinski --help
```

## 命令说明

### `sierpinski`

使用递归细分算法生成谢尔宾斯基三角形。

参数：
- `--size` (默认值: 32) - 三角形底边的字符宽度
- `--depth` (默认值: 5) - 递归深度
- `--char` (默认值: '*') - 用于填充点的字符

输出：三角形打印到标准输出，每行对应三角形的一行。

### `mandelbrot`

将曼德博集合渲染为ASCII艺术。将迭代次数映射到字符。

参数：
- `--width` (默认值: 80) - 输出宽度（字符数）
- `--height` (默认值: 24) - 输出高度（字符数）
- `--iterations` (默认值: 100) - 逃逸计算的最大迭代次数
- `--char` (默认值: gradient) - 单个字符，或省略以使用渐变字符集 " .:-=+*#%@"

输出：矩形区域打印到标准输出。

## 架构设计

```
cmd/
  fractals/
    main.go           # 程序入口点，CLI设置
internal/
  sierpinski/
    sierpinski.go     # 算法实现
    sierpinski_test.go
  mandelbrot/
    mandelbrot.go     # 算法实现
    mandelbrot_test.go
  cli/
    root.go           # 根命令，帮助信息
    sierpinski.go     # 谢尔宾斯基于命令
    mandelbrot.go     # 曼德博子命令
```

## 依赖项

- Go 1.21+
- `github.com/spf13/cobra` 用于CLI功能

## 验收标准

1. `fractals --help` 显示使用说明
2. `fractals sierpinski` 输出可识别的三角形
3. `fractals mandelbrot` 输出可识别的曼德博集合
4. `--size`、`--width`、`--height`、`--depth`、`--iterations` 参数正常工作
5. `--char` 可自定义输出字符
6. 无效输入会产生清晰的错误信息
7. 所有测试通过
