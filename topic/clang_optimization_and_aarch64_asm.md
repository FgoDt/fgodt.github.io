# clang 的优化引起的代码运行错误

事情的起因是我准备为播放器加入 flac 支持，这在 ffmpeg 开发的播放器看起来是一件非常简单的事情。我对 ffmpeg 库添加了-decoder=flac -demuxer=flac 编译选项，并成功编译了ffmpeg库。
一开始在 android 上播放器稳定且成功播放了 flac 格式的音频，这看起来很符合我的预期。但是当我将新的库放到 iOS 上时情况播放器就出了问题，在播放 flac 的时候进行 seek 操作就会崩溃，
而且是必现的崩溃，这就引起了我的注意，明明是同一个代码为什么在 android 没问题在iOS 有问题呢。

## -1 >= 0 ?

在对 iOS 平台库进行 debug 的时候我发现了一个崩溃点， 在 ff_seek_frame_binary 函数里

``` c
 if (index >= 0){
            e = &st->index_entries[index];
            av_assert1(e->timestamp >= target_ts);
            pos_max   = e->pos;
            ......
}

```

这里对 st->index_entries 取值时 index 为-1， 可是代码明明写了 index >= 0 时才会执行啊。 

我的第一直觉是 debug 可能读值不准确，可能在执行的时候 index 其实是大于0的，为了验证该观点我对该值进行了打印，结果也是 index 为-1。

这就很奇怪了我继续猜想，是不是多线程问题，虽然知道这个其实是局部变量不可能被多线程修改，我还是进行了线程打印发现没有多线程访问，所以也不是多线程问题。

接下来我就怀疑起了cpu 错误，我于是拿了另一台手机发现还是有问题，那就说明这个不是运行时运算错误，最大可能出现的问题就是编译器错了。

## 再看汇编

要了解编译器究竟干了什么就必须看运行时反汇编的代码

``` asm
    0x118ca1594 <+240>: bl     0x118ca8d68               ; OUTLINED_FUNCTION_80
    0x118ca1598 <+244>: ldr    w8, [x20, #0x1d0]
    0x118ca159c <+248>: cmp    w0, w8
    0x118ca15a0 <+252>: b.ge   0x118ca1690               ; <+492>
    0x118ca15a4 <+256>: ldr    x8, [x20, #0x1c8]
->  0x118ca15a8 <+260>: mov    w9, #0x18                 ; =24 
    0x118ca15ac <+264>: umaddl x8, w0, w9, x8
    0x118ca15b0 <+268>: ldp    x26, x27, [x8]
```
对应的 c 代码
``` c
  index = av_index_search_timestamp(st, target_ts,
                                          flags & ~AVSEEK_FLAG_BACKWARD);
  av_assert0(index < st->nb_index_entries);
  if (index >= 0) {
            e = &st->index_entries[index];
            av_assert1(e->timestamp >= target_ts);
            pos_max   = e->pos;
            ts_max    = e->timestamp;
            pos_limit = pos_max - e->min_distance;
            av_log(s, AV_LOG_TRACE, "using cached pos_max=0x%"PRIx64" pos_limit=0x%"PRIx64
                    " dts_max=%s\n", pos_max, pos_limit, av_ts2str(ts_max));
  }

```

这里对汇编代码一行一行解释
``` asm
    0x118ca1594 <+240>: bl     0x118ca8d68               ; OUTLINED_FUNCTION_80
```
调用 av_index_search_timestamp，并将结果存入 w0 寄存器

``` asm
    0x118ca1598 <+244>: ldr    w8, [x20, #0x1d0]
    0x118ca159c <+248>: cmp    w0, w8
    0x118ca15a0 <+252>: b.ge   0x118ca1690               ; <+492>
```
第一句：读取 x20 寄存器值加 #0x1d0 的内存地址保存到 w8，此时的 x20 是 AVStream* 结构体的地址，偏移 #0x1d0 地址是 nb_index_entries， 也就是说该句是将 w8 存入 AVStream—>nb_index_entries的值

第二句：比较 w0 和 w8 寄存器，w8 是 st->nb_index_entries, w0 是 index，也就是

第三句：如果 w0 >= w8 跳转到另一个地址

这三句也就是 c 代码的 av_assert0(index < st->nb_index_entries);

接下来就是重点了

``` asm
    0x118ca15a4 <+256>: ldr    x8, [x20, #0x1c8]
->  0x118ca15a8 <+260>: mov    w9, #0x18                 ; =24
    0x118ca15ac <+264>: umaddl x8, w0, w9, x8
    0x118ca15b0 <+268>: ldp    x26, x27, [x8]
```
第一句：类似上面的读取 nb_index_entries的值，不过这里地址偏移改变了，这里读取的是 index_entries 并保存到 x8 寄存器，

第二句：将 24 保存到 w9，为什么是24，因为 AVIndexEntry 这个结构体是24个字节

第三句：翻译过来就是 x8 = w0 * w9 + x8, 再代入各个表示的值可以表示成： x8 = index * sizeof(AVIndexEntry) + st->index_entries。
这一句相当于直接执行了 c 代码的e = &st->index_entries[index]。也就是说这一句直接进行了index地址计算根本没有检查index是否大于等于0

第四句：将x8 地址的两个64位内存保存到 x26、x27寄存器，也就是代码的 
``` c
pos_max   = e->pos; 
ts_max    = e->timestamp;
```
也就是在这里程序崩溃了，当然了这里没有检查index 而index为-1在寄存器里为0xffffffff，在通过x8 = w0 * w9 + x8后 x8 = 0xffffffff * 24 + x8，
这样的地址直接就非法了所以程序崩溃了


## clang -O2 做了什么


## 尝试解决

## 看似多余的代码

