## Windows 编译 Android SDK 

### 1. 安装 Android SDK

首先到官网下载 Android Studio 进行安装，安装完成后会提示安装 SDK。安装SDK过程中最好先开启代理，否则下载可能会很慢。

SDK安装完成后，设置系统环境变量： `ANDROID_HOME`， 对应的是 SDK 实际存放的目录

### 2. 安装 NDK 

`The NDK version tested by the OpenIM team was r20b. To build an AAR on Mac, gomobile, Android Studio, and the NDK version 20.0.5594570 must be installed`

Makefile 已经说明 NDK 需要安装的版本，故需下载指定版本号的 [NDK r20b](https://github.com/android/ndk/wiki/Unsupported-Downloads). 

下载完成后同样需要设置环境变量：`ANDROID_NDK_HOME`

### 3. 安装 gomobile

首先到 github 下载相关源码 [mobile](https://github.com/golang/mobile/) ，并将源码放到 GOPATH 下的目录 `src/golang.org/x` 中, 随后在 `mobile` 目录下执行编译命令

```go
E:\go\src\golang.org\x\mobile> go build golang.org/x/mobile/cmd/gomobile
E:\go\src\golang.org\x\mobile> go build golang.org/x/mobile/cmd/gobind
```

编译完成后会在 mobile 目录下生成两个执行文件 gomobile.exe 以及 gobind.exe. 最后将两个执行文件`放到GoPath下面的bin目录`即可，如 E:\go\bin

### 2. 编译

该项目已自带 Makefile 文件，故可以直接使用 make 命令来编译。但使用 make 工具需要通过  Cygwin 等类似工具来安装 make 工具。当然也可直接使用自定义脚本来执行编译

#### 2.1 使用脚本直接编译

Windows 编译脚本：

```bat
set GOARCH=amd64
gomobile bind -v -trimpath -ldflags="-s -w" -o ./open_im_sdk.aar -target=android ./open_im_sdk/ ./open_im_sdk_callback/
```

Linux 编译脚本：

```sh
export GOARCH=amd64
gomobile bind -v -trimpath -ldflags="-s -w" -o ./open_im_sdk.aar -target=android ./open_im_sdk/ ./open_im_sdk_callback/
```

#### 2.2 使用make工具编译

到官网 [Cygwin](https://cygwin.com/install.html) 下载其安装包进行安装，安装过程中程序会提示`选择软件包`：

1. 在`视图选项`选择`完整`

2. 在`搜索框` 输入 `make`

3. 随后在搜索结果中找到 `make` ，并选择最新版本 `4.4.1-2`

4. 最后点击下一页进行安装即可

5. 安装完成后，通过 Cygwin 桌面图标进入命令行，随后定位到sdk源码位置执行命令即可编译：

   ```sh
   make android
   ```

#### 2.3 编译结果

编译完成后，会在根目录生成两个文件：

```
open_im_sdk.aar
open_im_sdk-sources.jar
```

## 2. Windows 编译 H5 SDK

由于 H5 编译得到的文件较大，故需对文件进行 压缩处理，以便提升 H5 加载速度。所以在编译H5 SDK 前，请先安装 [gzip](https://gnuwin32.sourceforge.net/packages/gzip.htm) 工具 ， 并`将其路径加入到环境变量 Path` 中. 随后直接执行脚本：

```go
./build-wasm.bat
```

执行完成后会生成文件 `openIM.wasm.gzip`，该文件即是 H5 端需要使用的文件，存放于前端H5项目根目录的 public 文件夹中