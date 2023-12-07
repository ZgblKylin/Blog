![QOI徽标](C:\git\Blog\Develop\QOI\Translation\github\qoi-logo.svg)

# [QOI——“还可以图像(Quick OK Image)”格式，用于快速无损压缩](https://github.com/phoboslab/qoi/tree/master)

单文件MIT协议库，使用C/C++。

文档和格式规范详见[qoi.h](https://github.com/phoboslab/qoi/blob/master/qoi.h)

更多信息详见https://qoiformat.org


## 为什么？

相比于stb_image和stb_image_write，QOI提供20-50倍速度的编码，3-4倍速度的解码和20%的额外压缩率。它同样简单到愚蠢，包含大约300行C代码。


## 使用范例

- [qoiconv.c](https://github.com/phoboslab/qoi/blob/master/qoiconv.c)
png和qoi之间的转换
 - [qoibench.c](https://github.com/phoboslab/qoi/blob/master/qoibench.c)
stbi、libpng和qoi的简易对比测试


## MIME类别，文件扩展名

QOI图像的推荐MIME类别是`image/qoi`。尽管QOI尚未在IANA注册，我认为QOI足够避免与未来的图像格式重名，因此几乎不可能出现MIME类别冲突（详见#167](https://github.com/phoboslab/qoi/issues/167)）。

QOI图像的推荐扩展名为.qoi`


## 限制

QOI文件格式允许高达18E像素的图像。流式编/解码器可以以最低的内存使用处理它们，假设存储空间足够。

然而，当前QOI实现将图像限制为最大4亿像素，它会安全地拒绝编/解码任何大于该尺寸的内容。这不是一个流式编/解码器，它会将整张图像加载到内存中，然后才执行后续操作。同样，它并没有面向性能进行深度优化（尽管依然很快）。

如果以上限制影响到了您的使用场景，请查询下方列举的其他实现。


## 改进，新版本和参与贡献

QOI格式已经为终版。文件头中刻意**不**包含版本号。若您今天正致力于QOI实现，您可以确信它与明天的所有QOI文件兼容。

QOI的继任者有很多有趣的想法，但都不会在这里被实现。这并不意味着您不该参与贡献QOI，而是请注意，任何修改文件格式的拉取请求(PR, pull request)都不会被通过。

同样，提升性能的拉取请求也可能不被接受，因为这个“参考实现”致力于尽可能地便于阅读。

## 工具

译者注：随项目维护更新，详见[原项目页面](https://github.com/phoboslab/qoi/tree/master)。

## QOI的实现/语言绑定Implementations & Bindings of QOI

译者注：随项目维护更新，详见[原项目页面](https://github.com/phoboslab/qoi/tree/master)。

## 其他软件的QOI支持

译者注：随项目维护更新，详见[原项目页面](https://github.com/phoboslab/qoi/tree/master)。

## 包管理

译者注：随项目维护更新，详见[原项目页面](https://github.com/phoboslab/qoi/tree/master)。

- [AUR](https://aur.archlinux.org/pkgbase/qoi-git/)——系统全局qoi.h，qoiconv和qoibench，使用分离的不同软件包安装。
- [Debian](https://packages.debian.org/bookworm/source/qoi)——包含二进制和qoi.h的软件包
- [Ubuntu](https://launchpad.net/ubuntu/+source/qoi)——包含二进制和qoi.h的软件包

其他系统的软件包[在Repology进行跟踪](https://repology.org/project/qoi/versions)。