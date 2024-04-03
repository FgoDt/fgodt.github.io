# NDK 编译 FFmpeg 最新版本的正确方式

NDK 编译 FFmpeg 真的是一个老生常谈的问题了，网络上无数的文章写了无数遍也被采集了无数遍，但是对新版本的NDK仿佛都没有一个正确的方式。大家翻来覆去的把一个配置抄了又抄改了又改，最后每个人因为环境问题还是编译出错。我也在每次需要编译安卓库时找好多个网站来解决问题，于是我就想着能不能找到一个正确的方法并把它记下来于是就有了这篇文章。

### NDK 的版本区分
之所以现在版本的编译FFmpeg困难，是因为现在版本的AndroidStudio下载的NDK已经和以前的NDK不一样了。而网络上很多教程都还在使用gcc 版本的 NDK 来编译（新的NDK使用的是clang工具链），这样的环境差距自然导致编译不通过。
还有一个原因是大部分的教程都使用NDK(side by side)的原始目录来编译，而各个版本又有细微的区别导致常见的几个问题就是 sysroot错误、找不到链接库、找不到编译工具等

### standalone toolchain
最好的办法是使用standalone toolchain来编译，虽然Google已经说明这个方式被弃用了，但是很多时候我们还是需要使用这种方式，目前最新的版本(25.2.9519653)仍然支持standalone方式构建。关于standalone toolchain的介绍我这里贴个Google的[网址](https://developer.android.com/ndk/guides/standalone_toolchain?hl=zh-cn)大家有兴趣自己查看


### 脚本
说了这么多还是上脚本
```bash
# 检查是否设置了NDK路径
if [ "$ANDROID_NDK_ROOT" ]
then
	echo "USE NDK: "$ANDROID_NDK_ROOT
else
	echo " use \"export ANDROID_NDK_ROOT=[ndk path]\" to export ndk path "
	exit -1
fi

# 声明arch version 等
ARCH="arm64-v8a"
NDK_TOOLCHAIN_ABI_VERSION=4.9
export PLATFORM_VERSION=android-21
# standalone toolchain 安装位置
export TOOLCHAIN_PREFIX=`pwd`/fftoolchain_${PLATFORM_VERSION}_abi_${NDK_TOOLCHAIN_ABI_VERSION}_armv8
# FFmpeg 库编译后安装地址
LIB_INSTALL_PREFIX=`pwd`/ffbuild_${PLATFORM_VERSION}_${ARCH}

# 安装standalone toolchain
if [ ! -d "$TOOLCHAIN_PREFIX" ]; then
	${ANDROID_NDK_ROOT}/build/tools/make-standalone-toolchain.sh \
		--toolchain=aarch64-linux-android-$NDK_TOOLCHAIN_ABI_VERSION \
		--platform=$PLATFORM_VERSION \
		--install-dir=${TOOLCHAIN_PREFIX}
fi

NDK_CROSS_PREFIX="aarch64-linux-android"
CROSS_PREFIX=${TOOLCHAIN_PREFIX}/bin/${NDK_CROSS_PREFIX}-
NDK_SYSROOT=${TOOLCHAIN_PREFIX}/sysroot

# 声明各个工具
export CC="${CROSS_PREFIX}gcc --sysroot=${NDK_SYSROOT}"
export LD="${CROSS_PREFIX}ld"
export RANLIB="${CROSS_PREFIX}ranlib"
export STRIP="${CROSS_PREFIX}strip"
export READELF="${CROSS_PREFIX}readelf"
export OBJDUMP="${CROSS_PREFIX}objdump"
export ADDR2LINE="${CROSS_PREFIX}addr2line"
export AR="${CROSS_PREFIX}ar"
export AS="${CROSS_PREFIX}as"
export CXX="${CROSS_PREFIX}g++"
export OBJCOPY="${CROSS_PREFIX}objcopy"
export ELFEDIT="${CROSS_PREFIX}elfedit"
export CPP="${CROSS_PREFIX}cpp"
export DWP="${CROSS_PREFIX}dwp"
export GCONV="${CROSS_PREFIX}gconv"
export GDP="${CROSS_PREFIX}gdb"
export GPROF="${CROSS_PREFIX}gprof"
export NM="${CROSS_PREFIX}nm"
export SIZE="${CROSS_PREFIX}size"
export STRINGS="${CROSS_PREFIX}strings"

LDFLAGS=" -lm"
CPPFLAGS=${CFLAGS}

# 配置FFmpeg 这里disable了encoder 和 decoder 可根据需要自行配置
./configure \
	--prefix=${LIB_INSTALL_PREFIX} \
	--target-os=android \
	--arch=aarch64 \
	--cpu=armv8-a \
	--enable-cross-compile \
	--cross-prefix=${CROSS_PREFIX} \
	--disable-shared \
	--disable-ffmpeg \
	--disable-ffplay \
	--disable-ffprobe \
	--disable-decoders \
	--disable-encoders \
	--disable-devices \
	--sysroot=${NDK_SYSROOT} \
	--extra-cflags="${CFLAGS}" \
	--extra-cxxflags="${CPPFLAGS}" \
	--extra-ldflags="${LDFAGS}"

make -j4

make install
```

这里只是一个特定的配置，只支持arm64的cpu编译的android版本是21你可以根据自己需求自己修改

<!--more-->

