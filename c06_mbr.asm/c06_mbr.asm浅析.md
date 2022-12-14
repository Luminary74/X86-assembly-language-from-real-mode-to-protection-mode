## c06_mbr.asm

```
         ;代码清单6-1
         ;文件名：c06_mbr.asm
         ;文件说明：硬盘主引导扇区代码
         ;创建日期：2011-4-12 22:12 
      
         jmp near start
         
  mytext db 'L',0x07,'a',0x07,'b',0x07,'e',0x07,'l',0x07,' ',0x07,'o',0x07,\
            'f',0x07,'f',0x07,'s',0x07,'e',0x07,'t',0x07,':',0x07
  number db 0,0,0,0,0
  
  start:
         mov ax,0x7c0                  ;设置数据段基地址 
         mov ds,ax
         
         mov ax,0xb800                 ;设置附加段基地址 
         mov es,ax
         
         cld
         mov si,mytext                 
         mov di,0
         mov cx,(number-mytext)/2      ;实际上等于 13
         rep movsw
     
         ;得到标号所代表的偏移地址
         mov ax,number
         
         ;计算各个数位
         mov bx,ax
         mov cx,5                      ;循环次数 
         mov si,10                     ;除数 
  digit: 
         xor dx,dx
         div si
         mov [bx],dl                   ;保存数位
         inc bx 
         loop digit
         
         ;显示各个数位
         mov bx,number 
         mov si,4                      
   show:
         mov al,[bx+si]
         add al,0x30
         mov ah,0x04
         mov [es:di],ax
         add di,2
         dec si
         jns show
         
         mov word [es:di],0x0744

         jmp near $

  times 510-($-$$) db 0
                   db 0x55,0xaa
```



### 跳过非指令数据区

```
jmp near start ;跳过数据区
 
;要显示的字符串         
mytext db 'L',0x07,'a',0x07,'b',0x07,'e',0x07,'l',0x07,\
          ' ',0x07,'o',0x07,'f',0x07,'f',0x07,'s',0x07,\
          'e',0x07,'t',0x07,':',0x07
;存储分解数位
number db 0,0,0,0,0
  
start:
    ;代码部分
```

因为程序中没有区分数据段和代码段，这些后面会学到，所以该程序中包含了不可执行的普通数据。通过基于相对偏移量的近（near）跳转越过数据区，避免程序执行到这些非指令数据。

**单独定义数据区的好处：**

在c05_mbr.asm中，显示内容是以操作数的形式存在，这样不方便更改。要改必须重新编写指令。

通过单独定义数据区，将文本内容和实现显示它们的功能代码分开，更方便以后修改。

### 续行符  / 

一行写不下，可以用反斜杠续行。

也可以使用db伪指令

```
; 要显示的字符串         
mytext db 'L',0x07,'a',0x07,'b',0x07,'e',0x07,'l',0x07
       db ' ',0x07,'o',0x07,'f',0x07,'f',0x07,'s',0x07
       db 'e',0x07,'t',0x07,':',0x07
```

### 逻辑段地址的重新设定

```
start:
    mov ax,0x7c0 ;设置数据段基地址 
    mov ds,ax
    
    mov ax,0xb800 ;设置附加段基地址，指向显存地址
    mov es,ax
```

汇编地址和偏移地址的关系，前面已经了解过，这里加深印象。

在前一章，汇编地址和偏移地址不匹配进行的内存访问

```
 ;求十位上的数字
         xor dx,dx
         div bx
         mov [0x7c00+number+0x01],dl   ;保存十位上的数字
```

BIOS将MBR加载到[0x0000:0x7c00]处运行，但是这并不是加载到段的起始地址处，因此访问程序中的标号时需要加上一个固定的偏移量number。

如果我们要修改加载地址的段内的偏移，要修改指令，才能正确访问，很不方便，下面来介绍重置逻辑地址的方法。

为了使程序中标号的偏移地址与汇编地址一致，我们可以重置DS寄存器，这就体现出X86分段策略的灵活性，**同一段物理区域，可以用不同的[段地址 : 偏移地址]方式描述**。

### 程序的重定位

说明1：使得程序中的偏移地址与汇编地址一致，使得程序中对绝对地址的访问变得简单，不再存在加载地址偏移量。

基于相对地址的访问，其实不依赖于偏移地址与汇编地址一致。

这其实就反映出地址相关与地址无关指令的区别。


说明2：X86提供段机制的本意就是**为了实现程序重定位**，使得程序可以被加载到内存中的任意位置运行，所以根据加载地址合理设置段寄存器是应有之义。

因此，正确的理解方向应该是**程序加载到哪里，哪里就是逻辑段的起始地址**。之所以称之为逻辑段，就是因为对于同一个物理地址，可以有多种逻辑表达方式。



### 段之间的批量数据传送

```
cld                      ;设置正向传送模式
mov si,mytext            ;设置原始数据偏移地址
mov di,0                 ;设置目的数据偏移地址
mov cx,(number-mytext)/2 ;设置传送次数
rep movsw                ;重复传送数据  //按字传送
```

movsb & movsw指令

① movs（move string）指令实现数据串的传送，通常用于把数据从内存中的一个位置批量传送到另一个位置

② 根据传送数据的长度，分为不同的指令。movsb按字节传送，movsw指令按字传送

③ movs指令本身没有操作数，而是使用默认的寄存器，因此在调用之前需要做好准备工作

movs指令使用步骤：

1.设置原始数据地址：[DS:SI]

```
mov si,mytext  // [0x7c00:mytext]
```

2.设置目的数据地址：[ES:DI]

```
mov di,0   //  [0xB800:0x0000]
```

0xB800显存起始地址，因为我们要是实现显示数据，自然要送到显存去。

3.设置传送次数：CX

```
mov cx,(number-mytext)/2
```

这里因为程序中是以字为单位传送，字（显示字符+显示属性），因此要将算出来的字节数除以2、

4.设置传送方向：cld   

①movs指令有正向传送和反向传送2种工作模式，

**正向传送：源地址和目的地址都从低地址向高地址推进，每传送一次，SI & DI递增**

**反向传送：源地址和目的地址都从高地址向低地址推进，每传送一次，SI & DI递减**


② 传送方向由标志寄存器中的DF（Direction Flag）标志控制，①**

当DF为0时，表示正向传送

当DF为1时，表示反向传送

③ DF标志由cld & std指令设置，

cld将DF标志清零，即实现正向传送

std将DF标志置一，即实现反向传送

5.重复传送数据：rep

只要CX不为0，就一值重复。



### 使用循环分解数位

```
;得到标号所代表的偏移地址 / 汇编地址
    mov ax,number ;DX : AX构成被除数
         
    ;计算各个数位
    mov bx,ax ;bx指向保存数位的内存地址
    mov cx,5  ;设置循环次数 
    mov si,10 ;除数 
  digit: 
    xor dx,dx    ;dx会保存余数，所以每次除法运算前清空
    div si
    mov [bx],dl ;保存分解数位，即本次除法运算的余数
    inc bx      ;指向下一个保存位置
    loop digit
```

这里和上一章一样，DX保存余数，AX保存商

DX余数要清空，AX进入下一轮

loop指令

1.先将CX减一

2.如果CX不为0，跳转到指定位置继续执行，为0，往下执行



### 使用循环显示数位

```
    ;显示各个数位
    mov bx,number ;存储分解数位的基址
    mov si,4      ;存储分解数位的索引（下标）
show:
    mov al,[bx+si] ;从高位开始取出分解数位
    add al,0x30    ;转为为数字字符
    mov ah,0x04    ;设置显示属性
    mov [es:di],ax ; 写入显存
    add di,2       ;更新显存访问偏移量
    dec si        ;更新存储分解数位的索引
    jns show      ;当S标志未设置，则跳转到show标号处执行
         
    mov word [es:di],0x0744 ;显示字符'D'
```

基址变址寻址

```
mov al,[bx+si]
```

bx提供存储分解数位的内存地址，si作为索引，去一个一个的拿数字出来。

条件转移指令

jns指令实现跳转，该指令的跳转条件是标志寄存器中的符号位SF（Sign Flag）没有被置位

dec si指令在递减SI寄存器中的值时，会修改标志寄存器中的标志位，当运算到0 - 1 = 0xFFFF时，符号位SF会被置位，此时循环结束



```
jmp near $ ;进入无限循环
 
times 510-($-$$) db 0 ;填充0
db 0x55,0xaa
```

$和$$是NASM汇编器提供的记号，

$：当前这一行的汇编地址

$$：当前程序段的汇编地址