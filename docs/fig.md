---
title: 终端最美工具fig，自定义cli补全
show_first: true
---

## 效果预览图

![图片](https://camo.githubusercontent.com/74884a83c88c19b27b1a451b7e618fe773c6a48f6cb209a02d3fe70381a4f9d9/68747470733a2f2f6669672e696f2f676966732f64656d6f2d776974682d6865616465722e676966)

## 安装

- macOS:
    - DMG下载: [fig.io](fig.io)
    - Homebrew: `brew install fig`
- Windows/Linux:
    - 访问 [waitlist](https://withfig.typeform.com/linux)

**下载完成后，启动Fig，根据指引配置终端**

## 自定义cli补全

1. 生成ts文件，参考：[https://fig.io/docs/guides/integrating/integrations](https://fig.io/docs/guides/integrating/integrations)。例如cli工具使用go cobra开发，那么可以通过命令自动生成ts文件，详细见：[https://fig.io/docs/guides/integrating/integrations/cobra](https://fig.io/docs/guides/integrating/integrations/cobra)
2. 下载autocomplete源码，[https://github.com/withfig/autocomplete](https://github.com/withfig/autocomplete)
3. 将生成的ts放到src目录
4. 运行 `npm run dev`
5. 新开启一个终端，试试自动补全是否生效

## 扩展阅读

- [zsh 补全脚本介绍](http://chuquan.me/2020/11/28/how-to-write-a-zsh-completion-script/)

注：cobra提供了自动生成不全脚本的命令，会给命令自动添加一个`completion`子命令