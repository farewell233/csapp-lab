# 1.异或


1. （a&~b）&（~a&b）
	 a真b假 与a假b真 两种情况都符合
	 或者说 a和b补集的交集 并上 b和a补集的交集
2. ~(~a & ~b) & ~(a & b) 
	a b不同时为真 也不同时为假
# 3.isTmax
tmax=0111
tmax+1=1000
~(tmax+1)=0111

x^~(x+1) =0 说明x\==~（x+1）
再套个！
但是有特例
-1：1111
-1+1：0000
~（-1+1）：1111

所以加条件x！=-1进行限制

注意到-1+1=0  tmax+1!=0
而0可以理解为false
```c
int isTmax(int x) {
  return (!(x^~(x+1)))&(x+1)；
}
```
更好的答案
```c
return !((~(x+1)^x))&!!(x+1)
```
前后对应

# 4. 奇数位均为1
1010
1111也可
即不关心偶数位
不关心的位用0去遮盖
我直接 x&0xAAAAAAAA
这样偶数位全被0盖住了
我再看得到的结果和0xAAAAAAAA一样不一样

```c
int allOddBits(int x) {
    return !((x & (0xAAAAAAAA)) ^ 0xAAAAAAAA);
}
```

# 5. isAsciiDigit 
- return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
0x39       111001
0x30       110000

即比较给定x是否在此区间内
1. x的28个高位与0x30 0x39 是否一致
	(x>>4)^ 0x3
2. x的4个低位是否比0000大 比1001小
	1001+(-x)是否为负数 查看符号位即可
```c
int isAsciiDigit(int x) {
    int a = !((x >> 4) ^ 0x3);
    int b = 0xF & x;
    int c = ~b + 1;
    int d = 0x9 + c;
    return a & !(d >> 31);
}
```

# 6.conditional 条件句
```c
int conditional(int x, int y, int z) {
    int c = ~1 + 1;
    int a =y& ((!x) +c);
    int b = z &((!!x)+c);
    
    
    return  a ^ b;
}
```
思路 return的结果一定是
两个大式子 x满足条件时 运算结果分别为y和z 
中间由一个运算符衔接

注意到a^ 0\==0
中间的运算符就用^

又注意到a&0=0  a&-1=a  **掩码**
整体返回值需满足以下条件
x=1时 int a=y int b=0
x=0时 int a=0 int b=z

# 7. logical-neg
用其他符号完成实现!
0和非零数的一个区别
x非零 则x和~x+1的符号位不同
两符号位取或
右移后
要么是0 要么是-1 逻辑右移
加一即得答案
```c
int logicalNeg(int x) {
    int sign = (x | (~x + 1)) >> 31;
    return sign +1;
```

# 8. howManyBits
对于正数x 找最左边的1所在位 记为n 
x位数为n+1 加了个符号位 0
对于负数x 左边全是1 找0没意义 可以将其转化为正数
x的相反数和x占位不一定一样  但是!x和x的位数一样

**先统一转化成符号位为0的数**y
`int y = ((~flag)&x) | (flag&(~x));


**找最高位的1所在的第k位**

二分法最简便

共32位



太复杂了 看看仅8位的情况

高4位有1？
将y右移4位后规格化  为0则没有1 为1则有1
有-> k>4, k=4+i  y右移4位
	y高2位有1？y右移2位规格化
	 有->   k>4+2 ,k=4+2+i
	 没有->不管最高位是0是1 都需要1位来表示
没有->k<4 ,k>2? y直接右移2位

推广至32位情况
```c
int howManyBits(int x) {
    int flag = x >> 31;//符号位
    int y = ((~flag)&x) | (flag&(~x));//正数不变 负数按位取反

    int high_bit16, next_bit8, next_bit4, next_bit2, next_bit1,last_bit;

    high_bit16 = (!!(y >> 16)) << 4;//y的高16位是否有1
    y = y >> high_bit16;
    //不管低16位了 如果y的高16位有1 看y高16位的左起8位 
    //否则直接看y的高16+8位是否有1 
	//右移的巧妙之处在于衔接，判断位数区间内有没有1需要右移 
	//暂时忽略已判断的区间也需要右移
    next_bit8 = (!!(y >> 8)) << 3;
    y = y >> next_bit8;

    next_bit4 = (!!(y >> 4)) << 2;
    y = y >> next_bit4;

    next_bit2 = (!!(y >> 2)) << 1;
    y = y >> next_bit2;

    next_bit1 = (!!(y >> 1)) <<0;
    y = y >> next_bit1;

  

    return((high_bit16 + next_bit8 + next_bit4 + next_bit2 + next_bit1 + y) | 0) + 1;
}
```



# 9. floatScale2
临界没搞明白就开写 写半天一直不对

## 规格化值
exp!=0且exp!=-1
尾数不用动 只改阶码
仅需考虑是否溢出为无穷大

## 特殊值
exp=-1
返回变量本身即可

## 非规格化值
最复杂的一集
exp=0


### frac<<1是否溢出
**等价于frac最高位是否为1**
不溢出
	exp不变 frac=frac<<2
溢出
	exp=1 变为规格化值 E值没变 变的是M
	frac=frac<<1-8388608
### frac=0？
如果等于0 
exp=0 frac=0

ds给的 
```c
// 处理非规格化数
    // 非规格化数乘以2：尾数左移1位
    frac <<= 1;
    
    // 检查是否需要规格化（尾数溢出到指数位）
    if (frac & 0x00800000) {
        exp = 1;  // 最小规格化指数
        frac &= 0x007FFFFF;  // 清除溢出的位
    }
```
### 为什么清除溢出位即可得到新尾数
二进制视角下 
1.0101100
=0.10101100<<1
即原尾数右移后 整数部分（即溢出部分）就是个二进制1（最低位 等价于十进制1）抹除溢出部分 剩下的就是新尾数


```c
if (exp == 0) {
        int is_more = !!((frac+1) >> 22);
        int is_zero = !(frac ^ 0);
        if (is_more == 1) {
            exp = (exp+1);
            frac = (frac << 1) + (~8388608 + 1);
        }
```

十进制视角解 变为规格化值后 尾数表示的是小数部分 整数部分有个1 那么将小数视作分数 新的分子是之前分子乘二后减去分母值 即2^23


```c
unsigned floatScale2(unsigned uf) {
    unsigned flag = !!(uf >> 31);
    unsigned exp = (uf&0b01111111100000000000000000000000)>>23;
    unsigned frac = uf & 0b00000000011111111111111111111111;
    if ( exp == 0xff ) {
        return uf;
    }
    if (exp != 0) {
        exp += 1;
        if (exp >> 8) {
            exp = 0xff;
            frac = 0;
        }
    }
    if (exp == 0) {
        int is_more = !!((frac+1) >> 22);
        int is_zero = !(frac ^ 0);
        if (is_more == 1) {
            exp = (exp+1);
            frac = (frac << 1) + (~8388608 + 1);
        }//数学方式解 变为规格化值后 尾数表示的是小数部分 整数部分有个1 那么将小数视作分数 新的分子是之前分子乘二后减去分母值 即2^23
        if ((is_more == 0)&&(is_zero==0)) {
            frac =frac<< 1;

        }
        if (is_zero == 1) {
            exp = 0;
            frac = 0;
        }
    }
    //return ((exp | (flag<<8)) << 23) | frac; 不够清晰 
    return (flag << 31) | (exp << 23) | frac;
}
```

# 10. floatFloat2Int
依旧flag exp frac
转int是直接截断小数位 向0取整的
## 特殊值exp=0xff  
返回
## 规格化值 
exp!=0


E=exp-127
2的E次方即  1.frac << E

可能溢出为无穷大
## 非规格化值
exp=0


# 11. floatPower2 
Return bit-level equivalent of the expression 2.0^x for any 32-bit integer x.
### 表示2.0
### exp+x
正溢出 无穷大
负溢出 动frac 还溢出 0



