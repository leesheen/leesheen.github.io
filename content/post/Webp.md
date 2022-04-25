---
title: "批量转换 WebP"
date: 2022-03-01T18:02:38+08:00
draft: false
description: " "
tags: ["Mac", "Blog"]
categories: ["技术"]
---

> 竟然 Google 不到免费又好用的 WebP 转换工具？既然 Google 推广 WebP，一定会有的！

## 背景

前些时间域名到期了没有续费，Word Press 的博客就不再更新了。最近把之前的文章迁移到了 GitHub Pages，想把图片转换为 WebP 的格式，提高访问速度。Google 了半天，发现 Mac 图形界面工具大都是付费产品。

## Google 官方工具

[cwebp](https://developers.google.com/speed/webp/) 是 Google 官方的开源命令后工具，支持将 PNG、JPEG 等格式的图片转换为 WebP。

### 安装

在 Mac 下，可以使用 `brew install webp` 安装。

### 转换

转换的方式也很简单，使用默认配置：

```bash
cwebp temp.png -o temp.webp
```

输出结果：

```
Saving file 'temp.webp'
File:      temp.png
Dimension: 1524 x 1278 (with alpha)
Output:    63686 bytes Y-U-V-All-PSNR 49.33 55.07 55.96   50.59 dB
           (0.26 bpp)
block count:  intra4:        566  (7.37%)
              intra16:      7114  (92.63%)
              skipped:      7018  (91.38%)
bytes used:  header:            325  (0.5%)
             mode-partition:   5674  (8.9%)
             transparency:    39119 (99.0 dB)
 Residuals bytes  |segment 1|segment 2|segment 3|segment 4|  total
    macroblocks:  |       1%|       1%|      10%|      88%|    7680
      quantizer:  |      36 |      36 |      30 |      25 |
   filter level:  |      11 |       8 |       5 |       4 |
Lossless-alpha compressed size: 39118 bytes
  * Header size: 361 bytes, image data size: 38757
  * Precision Bits: histogram=5 transform=5 cache=0
  * Palette size:   129
```

上述是使用 Mac 部分截图功能生成的图片，转换后文件大小从 363K 减小到 62K，压缩比非常可观。

### 配置参数

使用 `cwebp --help` 可以看到 `cwebp` 所有支持的参数，这里就不展开了，需要的时候都可以 Google 到例子。

```
Usage:
 cwebp [-preset <...>] [options] in_file [-o out_file]

If input size (-s) for an image is not specified, it is
assumed to be a PNG, JPEG, TIFF or WebP file.
Note: Animated PNG and WebP files are not supported.

Options:
  -h / -help ............. short help
  -H / -longhelp ......... long help
  -q <float> ............. quality factor (0:small..100:big), default=75
  -alpha_q <int> ......... transparency-compression quality (0..100),
                           default=100
  -preset <string> ....... preset setting, one of:
                            default, photo, picture,
                            drawing, icon, text
     -preset must come first, as it overwrites other parameters
  -z <int> ............... activates lossless preset with given
                           level in [0:fast, ..., 9:slowest]

  -m <int> ............... compression method (0=fast, 6=slowest), default=4
  -segments <int> ........ number of segments to use (1..4), default=4
  -size <int> ............ target size (in bytes)
  -psnr <float> .......... target PSNR (in dB. typically: 42)

  -s <int> <int> ......... input size (width x height) for YUV
  -sns <int> ............. spatial noise shaping (0:off, 100:max), default=50
  -f <int> ............... filter strength (0=off..100), default=60
  -sharpness <int> ....... filter sharpness (0:most .. 7:least sharp), default=0
  -strong ................ use strong filter instead of simple (default)
  -nostrong .............. use simple filter instead of strong
  -sharp_yuv ............. use sharper (and slower) RGB->YUV conversion
  -partition_limit <int> . limit quality to fit the 512k limit on
                           the first partition (0=no degradation ... 100=full)
  -pass <int> ............ analysis pass number (1..10)
  -qrange <min> <max> .... specifies the permissible quality range
                           (default: 0 100)
  -crop <x> <y> <w> <h> .. crop picture with the given rectangle
  -resize <w> <h> ........ resize picture (after any cropping)
  -mt .................... use multi-threading if available
  -low_memory ............ reduce memory usage (slower encoding)
  -map <int> ............. print map of extra info
  -print_psnr ............ prints averaged PSNR distortion
  -print_ssim ............ prints averaged SSIM distortion
  -print_lsim ............ prints local-similarity distortion
  -d <file.pgm> .......... dump the compressed output (PGM file)
  -alpha_method <int> .... transparency-compression method (0..1), default=1
  -alpha_filter <string> . predictive filtering for alpha plane,
                           one of: none, fast (default) or best
  -exact ................. preserve RGB values in transparent area, default=off
  -blend_alpha <hex> ..... blend colors against background color
                           expressed as RGB values written in
                           hexadecimal, e.g. 0xc0e0d0 for red=0xc0
                           green=0xe0 and blue=0xd0
  -noalpha ............... discard any transparency information
  -lossless .............. encode image losslessly, default=off
  -near_lossless <int> ... use near-lossless image
                           preprocessing (0..100=off), default=100
  -hint <string> ......... specify image characteristics hint,
                           one of: photo, picture or graph

  -metadata <string> ..... comma separated list of metadata to
                           copy from the input to the output if present.
                           Valid values: all, none (default), exif, icc, xmp

  -short ................. condense printed message
  -quiet ................. don't print anything
  -version ............... print version number and exit
  -noasm ................. disable all assembly optimizations
  -v ..................... verbose, e.g. print encoding/decoding times
  -progress .............. report encoding progress

Experimental Options:
  -jpeg_like ............. roughly match expected JPEG size
  -af .................... auto-adjust filter strength
  -pre <int> ............. pre-processing filter

```

## 批量转换

因为要迁移很多文章，需要批量转换图片。看了半天 help 没找到对批量转换的支持，而且 `cwebp` 命令必须指定 `-o ` 参数，心想需要上 Shell 脚本了。

`xargs` 好像也行？又看了下 `xargs` 的 help，发现了 `-I {}` 参数的功能合适，**批量转换命令如下**：

```bash
find ./ -name '*.png' | awk -F '.png' '{print $1}' | xargs -I {} cwebp {}.png -o {}.webp
```

可以按需要定制这行命令。

### xargs -I

`xargs` 默认可以接受一个输入。比如 `find ./ -name need-touch | touch`

当需要使用多次输入的时候，就可以添加 `-I {}` 参数，然后在后面的命令使用 `{}` 作为替代符。如前面的批量命令使用的那样。 
