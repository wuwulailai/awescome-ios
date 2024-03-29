# iOS可执行文件瘦身方法

Appstore安装包是由资源和可执行文件两部分组成的。瘦身也从这两部分入手

## 资源瘦身
资源瘦身主要是去掉无用资源和压缩资源，资源包括图片、音视频文件、配置文件以及多语言wording。

#### 无用资源指资源在工程文件里，但没有代码引用
检查：用资源关键字搜索，手工过滤
#### 资源压缩指对png进行无损压缩，用ImageOptim工具和compress命令

ImageOptim
https://imageoptim.com/howto.html
缩减安装包大小，首先会对资源文件下手，压缩图片/音频，去除不必要的资源。
还可以对可执行文件进行瘦身，项目越大，执行文件的体积越大。
AppStore对执行文件加密，导致执行文件压缩率低，压缩后可执行文件体积占比80%-90%。值得优化

研究过程使用了linkmap。

iOS APP可执行文件的组成：
资源文件 + 可执行文件 
引入的库多了，可执行文件很大。

可执行文件构成是怎样的，里面的内容是什么，占用空间。

### 编译选项
1. 编译器优化级别
Build Settings->Optimization Level
2. 去除符号信息
Strip Linked Product / Deployment Postprocessing / Symbols Hidden by Default 在release版本应该设为yes，可以去除不必要的调试符号

### 第三方库统计

### 无用代码

### 类/方法名长度
混淆，压缩

### 冗余字符串
Log非常多，静态字符串抽离为静态资源文件，压缩率会比可执行文件高很多。

## Checklist

资源优化

编译优化

可执行文件优化
