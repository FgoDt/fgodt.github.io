## h264 CAVLC编码
一个残差块(residual_block)它的值为下表
```c
+---+---+---+---+
| 0 | 3 | -1| 0 |
+---+---+---+---+
| 0 | -1| 1 | 0 |
+---+---+---+---+
| 1 | 0 | 0 | 0 |
+---+---+---+---+
| 0 | 0 | 0 | 0 |
+---+---+---+---+
```
对其进行zigzag排序后得到数据队列 N=[0, 3, 0, 1, -1, -1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0]
### 编码TotalCoeff和TrailingOnes
>这里有几个概念要先说明一下
1. TotalCoeff：非零系数的数目（也就是N队列里不为零的数据个数和，N队列这个值为5）
2. TrailingOnes：脱尾系数数目（N队列里重后向先前值为1/-1的个数和）
     1. TrailingOnes 的取值范围[0..3]，如果队列N里的1/-1个数超过3个，也应该只取最后3位为脱尾系数

以上得到TotalCoeff=5，得到TrailingOnes=3

查找coeff_token对应表9-5得到编码，假设nC为0，我们查到TotalCoeff==1&&TrailingOnes==1对应的二进制编码为01
，将值写入bit流中则完成了coeff_token编码

如果TotalCoeff为0直接退出，我们就完成了整个CAVLC编码

如果TotalCoeff不为0按以下步骤继续编码

### 编码去拖尾非零系数
先看levelCode的标准描述
>9.2.2 Parsing process for level information
>
>Inputs to this process are bits from slice data, the number of non-zero transform coefficient levels
TotalCoeff( coeff_token ), and the number of trailing one transform coefficient levels TrailingOnes( coeff_token ).
>
>Output of this process is a list with name levelVal containing transform coefficient levels.
>
>Initially an index i is set equal to 0. Then, when TrailingOnes( coeff_token ) is not equal to 0, the following ordered steps are applied TrailingOnes( coeff_token ) times to decode the trailing one transform coefficient levels:
>1. A 1-bit syntax element trailing_ones_sign_flag is decoded and evaluated as follows:
>>
>>– If trailing_ones_sign_flag is equal to 0, levelVal[ i ] is set equal to 1.
>>
>>– Otherwise (trailing_ones_sign_flag is equal to 1), levelVal[ i ] is set equal to −1.
>2. The index i is incremented by 1.
>
>Then, the variable suffixLength is initialised as follows:
>
>– If TotalCoeff( coeff_token ) is greater than 10 and TrailingOnes( coeff_token ) is less than 3, suffixLength is set equal to 1.
>– Otherwise (TotalCoeff( coeff_token ) is less than or equal to 10 or TrailingOnes( coeff_token ) is equal to 3), suffixLength is set equal to 0.
>
>Then, when TotalCoeff( coeff_token ) − TrailingOnes( coeff_token ) is not equal to 0, the following ordered steps are applied TotalCoeff( coeff_token ) − TrailingOnes( coeff_token ) times to decode the remaining non-zero level values:
>
>1. The syntax element level_prefix is decoded as specified in clause 9.2.2.1.
>2. The variable levelSuffixSize is set as follows:
>>– If level_prefix is equal to 14 and suffixLength is equal to 0, levelSuffixSize is set equal to 4.
>>
>>– Otherwise, if level_prefix is greater than or equal to 15, levelSuffixSize is set equal to level_prefix − 3.
>>
>>– Otherwise, levelSuffixSize is set equal to suffixLength.
>3. The syntax element level_suffix is decoded as follows:
>>– If levelSuffixSize is greater than 0, the syntax element level_suffix is decoded as unsigned integer
representation u(v) with levelSuffixSize bits.
>>
>>– Otherwise (levelSuffixSize is equal to 0), the syntax element level_suffix is inferred to be equal to 0.
>4. The variable levelCode is set equal to ( Min( 15, level_prefix ) << suffixLength ) + level_suffix.
>5. When level_prefix is greater than or equal to 15 and suffixLength is equal to 0, levelCode is incremented by 15.
>6. When level_prefix is greater than or equal to 16, levelCode is incremented by (1<<( level_prefix − 3 )) − 4096.
>7. When the index i is equal to TrailingOnes( coeff_token ) and TrailingOnes( coeff_token ) is less than 3, levelCode is incremented by 2.
>8. The variable levelVal[ i ] is derived as follows:
>>– If levelCode is an even number, levelVal[ i ] is set equal to ( levelCode + 2 ) >> 1
>>
>>– Otherwise (levelCode is an odd number), levelVal[ i ] is set equal to ( −levelCode − 1) >> 1.
>9. When suffixLength is equal to 0, suffixLength is set equal to 1.
>10. When the absolute value of levelVal[ i ] is greater than ( 3 << ( suffixLength − 1 ) ) and suffixLength is less than 6,suffixLength is incremented by 1.
>11. The index i is incremented by 1

一个去拖尾非零系数值按以下方式编码, 
```c
+---------------+--------------+
| level_prefix  | level_suffix |
+---------------+--------------+
````
1. 计算除去拖尾数据的非零系数levelCode信息
   1. 逆序顺序依次找出需要计算的数num
   2. 更具上述文档第8条解码levelCode计算方法来反推编码levelCode计算方法，
      1. 先看看奇偶数更具上述标准8解码后得到的值，带入数据 y=[2, 4, 6, 8] 得到 x=[2, 3, 4, 5], 带入数据y=[3, 5, 7, 9] 得到 y=[-2, -3, -4, -5]，可以发现levelCode编码就是将正负数全部按正数编码，且偶数表示正数奇数表示负数，和有符号哥伦布编码类似
      2. 接下来按解码标准推导编码方式，我们先用偶数来推导，设解码levelCode值为X，编码的levelCode为Y 我们有一个等式： (y+2) >> 1 = x 则能推导出 y = (x << 1) - 2, 同理可以推导出奇数的编码计算方式 （-y-1)>>1 = x -> y = -(x<<1)-1。这里就完成了编码的推导，不过大部分编码是按下面计算levelCode的，效果相同
   3. 如果num是正数levelCode = (abs(num) - 1)<<1
   4. 如果num是负数levelCode = (abs(num) - 1)<<1+1
   5. 如果TrailingOnes小于3那么第一个非零非拖尾levelCode -= 2
      1. 小于3那么说明拖尾最多两个-1/+1数据，倒数第三个非零数据不可能出现abs(num) == 1的情况，也就保证了levelCode在减掉2后为正数
   7. 队列N中需要计算的数有[1，3], 计算结果为levelCode(1)=0 由于TrailingOnes == 3 所以不用 -2,levelCode(3)=4, 
   
3. 根据
