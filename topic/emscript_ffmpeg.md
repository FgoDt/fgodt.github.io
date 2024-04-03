# Emscripten 编译 FFmpeg 6.0 并调用

WebAssembly 概念已经出来好多年了，市面上也有很多基于此技术的应用。最近突然想研究 web 上的视频播放器，目前有几个技术流派：
1. 使用 h5 的规范，mp4 hls
2. 使用 mse 扩展
3. 使用第三方库来解码渲染

前两个都很 web 的技术，第三个技术其实是我在移动端一直在做的，所以对开发一个这样的播放器很感兴趣。为了熟悉Emscripten我准备先从编译 FFmpeg 库开始。

## 预先准备
1. 安装Emscripten 很简单跟随[官网]("https://emscripten.org/docs/getting_started/downloads.html")教程安装
a. 拉取代码
```shell
# 获取emsdk
git clone https://github.com/emscripten-core/emsdk.git

# 进入目录
cd emsdk
```
b. 安装emsdk
```shell
#安装 latest 版本的 emsdk
./emsdk install latest
#激活 latest 版本的 emsdk
./emsdk activate latest
#将 emsdk 环境加入当前terminal
source ./emsdk_env.sh
```
2. 拉取 FFmpeg 代码

## 配置 FFmpeg
WebAssembly 的库需要 emsdk 来编译，具体需要的两个脚本是 emconfigure 和 emmake
1. 根据需求裁剪 FFmpeg，下面的配置是为了用于测试所以很多功能我都屏蔽了。配置时注意一定使用 emconfigure 
```shell
emconfigure ./configure \
--disable-pthreads \
--disable-encoders \
--disable-devices \
--disable-protocols \
--disable-asm \
--disable-muxers \
--disable-ffmpeg \
--disable-ffplay \
--disable-ffprobe \
--disable-demuxers \
--disable-hwaccels \
--disable-decoders \
--disable-bsfs \
--disable-parsers \
--disable-filters \
--disable-decoders \
--enable-decoder=aac \
--enable-decoder=h264 \
--enable-bsf=h264_metadata \
--enable-bsf=h264_mp4toannexb \
--enable-filter=aresample \
--prefix=./emffbuild \
--cc="emcc"  \
--ranlib="emranlib"  \
--enable-cross-compile \
--target-os=none \
--arch=x86_32 \
--nm="llvm-nm" \
--ar=emar \
--cxx=em++ \
--dep-cc=emcc
```
2. 编译 FFmpeg
编译和常规 make 的区别是 make 前加 emmake
```shell
emmake make -j 4 install
```

## 使用 FFmpeg 库
尝试调用 FFmpeg 库

先来个测试程序
```c
#include <libavcodec/avcodec.h>

int main(){
	const AVCodec* codec = avcodec_find_decoder(AV_CODEC_ID_AAC);
	if(codec == NULL){
		printf("can not find acc decoder\n");
		return -1;
	}
	printf("AVCodec ptr: %p\n", codec);
	return 0;
}
```

## 编译
```shell
emcc test.c \
-I/Users/xxx/Documents/xxx/emsdk/FFmpeg/emffbuild/include \
-L/Users/xxx/Documents/xxx/emsdk/FFmpeg/emffbuild/lib \
-lavcodec -lavformat -lavutil -o test.html 
```
编译时可能会出现下面的错误
```shell
wasm-ld: error: initial memory too small, xxx bytes needed
```
可以在编译后加入TOTAL_MEMORY来增加内存
```shell
-s TOTAL_MEMORY=32MB
```

最后效果
![2023-10-20T10:56:39.png][1]


  [1]: https://blog.nintendo-fans.com/usr/uploads/2023/10/2935766370.png
