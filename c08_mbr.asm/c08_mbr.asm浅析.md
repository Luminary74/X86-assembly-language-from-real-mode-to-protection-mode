## c08_mbr.asm



```
         ;代码清单8-1
         ;文件名：c08_mbr.asm
         ;文件说明：硬盘主引导扇区代码（加载程序） 
         ;创建日期：2011-5-5 18:17
         
         app_lba_start equ 100           ;声明常数（用户程序起始逻辑扇区号）
                                         ;常数的声明不会占用汇编地址
                                    
SECTION mbr align=16 vstart=0x7c00                                     

         ;设置堆栈段和栈指针 
         mov ax,0      
         mov ss,ax
         mov sp,ax
      
         mov ax,[cs:phy_base]            ;计算用于加载用户程序的逻辑段地址 
         mov dx,[cs:phy_base+0x02]
         mov bx,16        
         div bx            
         mov ds,ax                       ;令DS和ES指向该段以进行操作
         mov es,ax                        
    
         ;以下读取程序的起始部分 
         xor di,di
         mov si,app_lba_start            ;程序在硬盘上的起始逻辑扇区号 
         xor bx,bx                       ;加载到DS:0x0000处 
         call read_hard_disk_0
      
         ;以下判断整个程序有多大
         mov dx,[2]                      ;曾经把dx写成了ds，花了二十分钟排错 
         mov ax,[0]
         mov bx,512                      ;512字节每扇区
         div bx
         cmp dx,0
         jnz @1                          ;未除尽，因此结果比实际扇区数少1 
         dec ax                          ;已经读了一个扇区，扇区总数减1 
   @1:
         cmp ax,0                        ;考虑实际长度小于等于512个字节的情况 
         jz direct
         
         ;读取剩余的扇区
         push ds                         ;以下要用到并改变DS寄存器 

         mov cx,ax                       ;循环次数（剩余扇区数）
   @2:
         mov ax,ds
         add ax,0x20                     ;得到下一个以512字节为边界的段地址
         mov ds,ax  
                              
         xor bx,bx                       ;每次读时，偏移地址始终为0x0000 
         inc si                          ;下一个逻辑扇区 
         call read_hard_disk_0
         loop @2                         ;循环读，直到读完整个功能程序 

         pop ds                          ;恢复数据段基址到用户程序头部段 
      
         ;计算入口点代码段基址 
   direct:
         mov dx,[0x08]
         mov ax,[0x06]
         call calc_segment_base
         mov [0x06],ax                   ;回填修正后的入口点代码段基址 
      
         ;开始处理段重定位表
         mov cx,[0x0a]                   ;需要重定位的项目数量
         mov bx,0x0c                     ;重定位表首地址
          
 realloc:
         mov dx,[bx+0x02]                ;32位地址的高16位 
         mov ax,[bx]
         call calc_segment_base
         mov [bx],ax                     ;回填段的基址
         add bx,4                        ;下一个重定位项（每项占4个字节） 
         loop realloc 
      
         jmp far [0x04]                  ;转移到用户程序  
 
;-------------------------------------------------------------------------------
read_hard_disk_0:                        ;从硬盘读取一个逻辑扇区
                                         ;输入：DI:SI=起始逻辑扇区号
                                         ;      DS:BX=目标缓冲区地址
         push ax
         push bx
         push cx
         push dx
      
         mov dx,0x1f2
         mov al,1
         out dx,al                       ;读取的扇区数

         inc dx                          ;0x1f3
         mov ax,si
         out dx,al                       ;LBA地址7~0

         inc dx                          ;0x1f4
         mov al,ah
         out dx,al                       ;LBA地址15~8

         inc dx                          ;0x1f5
         mov ax,di
         out dx,al                       ;LBA地址23~16

         inc dx                          ;0x1f6
         mov al,0xe0                     ;LBA28模式，主盘
         or al,ah                        ;LBA地址27~24
         out dx,al

         inc dx                          ;0x1f7
         mov al,0x20                     ;读命令
         out dx,al

  .waits:
         in al,dx
         and al,0x88
         cmp al,0x08
         jnz .waits                      ;不忙，且硬盘已准备好数据传输 

         mov cx,256                      ;总共要读取的字数
         mov dx,0x1f0
  .readw:
         in ax,dx
         mov [bx],ax
         add bx,2
         loop .readw

         pop dx
         pop cx
         pop bx
         pop ax
      
         ret

;-------------------------------------------------------------------------------
calc_segment_base:                       ;计算16位段地址
                                         ;输入：DX:AX=32位物理地址
                                         ;返回：AX=16位段基地址 
         push dx                          
         
         add ax,[cs:phy_base]
         adc dx,[cs:phy_base+0x02]
         shr ax,4
         ror dx,4
         and dx,0xf000
         or ax,dx
         
         pop dx
         
         ret

;-------------------------------------------------------------------------------
         phy_base dd 0x10000             ;用户程序被加载的物理起始地址
         
 times 510-($-$$) db 0
                  db 0x55,0xaa
```



### 程序实现功能

加载器：将位于磁盘上的代码加载到内存，并将执行流程转移到加载的代码。

首先我们需将磁盘上从指定扇区开始的若干个扇区内容拷贝并放置到从指定物理内存地址开始的对应尺寸内存；磁盘上的代码本身，在拷贝到物理内存后，代码自身可以知道自己所处的物理内存位置。

加载器工作流程：

① 读取用户程序的起始扇区，获取**用户程序头部信息**

② 根据用户程序头部信息，将整个**用户程序读入内存**

③ 计算段的物理地址和逻辑段地址，即**段的重定位**

④ 跳转到**用户程序执行**，即将处理器的控制权交给用户程序



### 确定用户程序在硬盘上的位置

```
 app_lba_start equ 100;声明常数（用户程序起始逻辑扇区号）					;常数的声明不会占用汇编地址
```

规定用户程序在硬盘上的起始逻辑扇区号为100，我们使用equ伪指令定义该常数。

使用equ伪指令声明的数值不占用任何汇编地址，也不在运行时占用任何内存单元，他仅仅代表一个数值，和C语言中的宏定义类似。



```
SECTION mbr align=16 vstart=0x7c00;定义一个mbr段，段是以16字节对齐，汇编地址开始的地址是0x7c00        
```

主引导扇区使用VSTART子句，将段内元素汇编地址的计算基准设置为0x7C00，因为BIOS将主引导扇区加载到[0x0000:0x7C00]处运行，实现了编译时汇编地址和运行时偏移地址的统一。

此时，主引导程序从磁盘固定位置移动到固定物理内存位置：0x0000：0x7c00，CS=0x0000

### 确定用户程序在内存中的加载位置

拓展：

可用的物理内存空间

![img](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210012208526.png)
      

1.如图，根据8086的memory map，0x0000到0x9ffff共640KB的空间用于访问DRAM

2.内存中的0x0000到0x0FFFF的范围，被用于加载主引导扇区，同时设置了主引导扇区程序所用的栈

3.因此，可以使用**0x10000到0x9FFFF**共576KB的空间**加载用户程序**



计算加载用户程序的逻辑段地址      

```
phy_base dd 0x10000   ;用户程序被加载的物理起始地址

   		;设置堆栈段和栈指针 
         mov ax,0      
         mov ss,ax
         mov sp,ax
      
         mov ax,[cs:phy_base]            ;计算用于加载用户程序的逻辑段地址 
         mov dx,[cs:phy_base+0x02]
         mov bx,16        
         div bx            
         mov ds,ax                       ;令DS和ES指向该段以进行操作
         mov es,ax                        
```

加载用户程序的物理地址确定为0x10000，该地址位于可用物理内存范围内，且以16B对齐，便于计算逻辑段地址。使用双字单元phy_base来存储该物理地址，因为只有32位的空间才能容纳20位的物理内存。

计算加载用户程序的逻辑段地址，就是将物理地址除以16，除法运算后，商在AX中，这个值就是加载用户程序的逻辑段地址，将DS&ES指向该段，我们就可以开始读取用户程序头部。

**为何不用equ伪指令声明加载用户程序的物理地址 ?**为什么不这样直接拿到用户程序的物理地址加载呢？

在声明应用程序在硬盘上的位置时，使用了equ伪指令，为何此处不使用equ伪指令，而是使用dd伪指令预留并初始化内存呢 ?

这是因为此处加载用户程序的物理地址超过了16位，如果使用equ伪指令声明，在实模式下无法用16位的寄存器处理



```
;以下读取程序的起始部分 
         xor di,di
         mov si,app_lba_start            ;程序在硬盘上的起始逻辑扇区号 
         xor bx,bx                       ;加载到DS:0x0000处 
         call read_hard_disk_0
```

``xor di,di`` 异或，把di置0

``app_lba_start equ 100`` 起始逻辑扇区号为100

``mov si,app_lba_start `` si=100

``xor bx,bx`` 异或，把bx置0

``di:si-->ds:bx`` 磁盘扇区：di:si 指向 物理内存地址：ds:bx

数据块在硬盘的起始位置，数据块在内存的起始位置



### 读取扇区

``call read_hard_disk_0`` 跳到这开始执行

```
read_hard_disk_0: ; di:si起始逻辑扇区 bx：要放入起始物理位置的段内偏移。
         push ax 
         push bx
         push cx
         push dx ; 函数会修改的寄存器先入栈保存
      
         mov dx,0x1f2 ; 逻辑扇区个数
         mov al,1
         out dx,al ; 
 
         inc dx    ; 起始逻辑扇区号28位。高4位是1110，表示LAB下逻辑扇区，位于主硬盘。
         mov ax,si 
         out dx,al ; 送入低8位
 
         inc dx 
         mov al,ah
         out dx,al ; 送入次低8位
 
         inc dx 
         mov ax,di 
         out dx,al ; 送入次次低8位
 
         inc dx 
         mov al,0xe0 
         or al,ah ; 设置最高4位&28位起始逻辑扇区的高4位
         out dx,al 
 
         inc dx 
         mov al,0x20 
         out dx,al ; 写入0x20，表示起始逻辑扇区，扇区个数用来指示读取这个位置的磁盘内容
```

设置读取的扇区数量：

```
 mov dx,0x1f2 ; 逻辑扇区个数
 mov al,1
 out dx,al ; 读取的扇区数
```

设置要读取扇区数的0x1f2端口为8位端口，因此只能读写255个扇区。需要注意的是，如果写入该端口的值为0，则表示要读写256个扇区

后续每读取一个扇区，该端口的数值减一，因此如果在读写过程中发生错误，该端口包含着尚未读取的扇区数

设置起始LBA扇区号：

假设DI:SI存储起始逻辑扇区号

```
inc dx    ; 起始逻辑扇区号28位。高4位是1110，表示LAB下逻辑扇区，位于主硬盘。
         mov ax,si 
         out dx,al ; 送入低8位
 
         inc dx 
         mov al,ah
         out dx,al ; 送入次低8位
 
         inc dx 
         mov ax,di 
         out dx,al ; 送入次次低8位
 
         inc dx 
         mov al,0xe0 
         or al,ah ; 设置最高4位&28位起始逻辑扇区的高4位
         out dx,al 
```

设置起始LBA扇区号的端口均为8位端口，因此LBA28模式需要4个端口设置，其中最后一个端口多余4位，在实现中用于指定硬盘及其访问模式，如下图所示

![img](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210012301582.png)

之所以要指定主硬盘 / 从硬盘，是因为每个PATA/SATA接口允许挂接2块硬盘，分别是主盘（Master）和从盘（Slave）

发送读硬盘命令：

```
         inc dx 
         mov al,0x20 
         out dx,al ; 写入0x20，表示起始逻辑扇区，扇区个数用来指示读取这个位置的磁盘内容
```

等待硬盘操作完成：

```
  .waits:
         in al,dx
         and al,0x88
         cmp al,0x08
         jnz .waits                      ;不忙，且硬盘已准备好数据传输 

         mov cx,256                      ;总共要读取的字数
         mov dx,0x1f0
```

0x1f7端口既是命令端口，又是状态端口，在通过该端口发送读写命令之后，就可以轮询该端口，确定硬盘操作是否完成，该端口状态位如下图所示，

![img](https://img-blog.csdnimg.cn/20210523141109353.png)

连续取出数据：

假设DS : BX存储目标缓冲区地址

```
mov cx,256                      ;总共要读取的字数
         mov dx,0x1f0
  .readw:
         in ax,dx
         mov [bx],ax
         add bx,2
         loop .readw
```

0x1f0为硬盘的数据端口，是一个16位端口，因此每次读取2B，共读取256次，即可读取一个扇区

最后还有一个0x1f1端口，该端口为错误寄存器，包含硬盘驱动器最后一次指令命令后的状态（错误原因）



### 过程调用和保护现场

``read_hard_disk_0``这个过程在程序很多地方被调用，可以理解为高级语言中的函数。

call 函数  就是调用函数，调用时会把寄存器中的内容push保存到栈，保护现场；ret 函数，退出函数，推出时会把寄存器中的内容回复，也就是pop出栈。



### 用户程序的重定位

加载整个用户程序

之前仅仅读取了用户程序的第1个扇区，下面将根据第1个扇区中的头部信息解析用户程序的总长度，并将整个用户程序加载到内存中

```

         ;以下读取程序的起始部分 
         xor di,di
         mov si,app_lba_start            ;程序在硬盘上的起始逻辑扇区号 
         xor bx,bx                       ;加载到DS:0x0000处 
         call read_hard_disk_0
      
         ;以下判断整个程序有多大
         mov dx,[2]                      ;曾经把dx写成了ds，花了二十分钟排错 
         mov ax,[0]
         mov bx,512                      ;512字节每扇区
         div bx
         cmp dx,0
         jnz @1                          ;未除尽，因此结果比实际扇区数少1 
         dec ax                          ;已经读了一个扇区，扇区总数减1 
   @1:
         cmp ax,0                        ;考虑实际长度小于等于512个字节的情况 
         jz direct
         
         ;读取剩余的扇区
         push ds                         ;以下要用到并改变DS寄存器 

         mov cx,ax                       ;循环次数（剩余扇区数）
   @2:
         mov ax,ds
         add ax,0x20                     ;得到下一个以512字节为边界的段地址
         mov ds,ax  
                              
         xor bx,bx                       ;每次读时，偏移地址始终为0x0000 
         inc si                          ;下一个逻辑扇区 
         call read_hard_disk_0
         loop @2                         ;循环读，直到读完整个功能程序 

         pop ds                          ;恢复数据段基址到用户程序头部段 
      
         ;计算入口点代码段基址 
   direct:
         mov dx,[0x08]
         mov ax,[0x06]
         call calc_segment_base
         mov [0x06],ax                   ;回填修正后的入口点代码段基址 
```

扇区数：

```
mov bx,512 
div bx ; dx:ax / 512，商ax，余数dx。磁盘程序尺寸/512
```

计算还需要读取的扇区个数

因为之前已经读取了1个扇区，此处需要将其减去

```
dec ax ;余数是0会执行这一步。这样，ax就就代表了尚未拷贝到物理内存的存有磁盘程序的磁盘块个数
```

读取目标缓冲区地址的设置:

```
@1:
         cmp ax,0  
         jz direct ;
         push ds ; 表示尚有磁盘程序未拷贝到物理内存，先保存ds    
         mov cx,ax  ; cx代表剩余待拷贝到物理内存的磁盘块个数
   @2:
         mov ax,ds 
         add ax,0x20  ; 
         mov ds,ax  ; ds在原有基础上新增了一个磁盘块的偏移
         xor bx,bx  
         inc si   
         ; 循环不变式：
         ; di:si 下一个磁盘块的逻辑号
         ; ds:bx 下一个磁盘块所在的起始物理内存位置 
         call read_hard_disk_0 
         ; 重复
         loop @2     
         pop ds  
```

在read_hard_disk_0过程中，使用DS : BX表示读取扇区时的目标缓冲区地址，我们分析一下这2个寄存器在读取第1个扇区过程中的状态

① 读取第1个扇区之前

[DS : BX] = [0x1000 : 0x0000]


② 读取第1个扇区之后

[DS : BX] = [0x1000 : 0x0200]


在后续的读取过程中，如果保持DS寄存器的值不变，仅依靠递增BX寄存器的值是不可行的。因为BX可寻址的范围只有64KB，如果用户程序超过64KB，则无法索引

因此在实现中，在每次读取1个扇区之后，将DS寄存器的值加0x20，也就是将段地址递增512B，以此来索引目标缓冲区

而每次读取时，BX寄存器的值均为0

![img](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202210012323356.png)

在读取用户程序剩余扇区的前后，保存并恢复了DS寄存器的值，因此在加载整个用户程序之后，DS恢复为0x1000，仍指向物理地址0x10000的起始处



### 程序段的重定位

```
direct:
         mov dx,[0x08] 
         mov ax,[0x06] 
         ; dx:ax 代码段汇编地址，这里是入口点所在的代码段
         call calc_segment_base 
         ; 将4字节入口点代码段汇编地址的低2字节设置为该段在物理内存起始位置处的段地址
         mov [0x06],ax ; ax是重定位后的代码段地址
         mov cx,[0x0a] ; 重定位表的项数    
         mov bx,0x0c   ; 首个重定位项的偏移--距离磁盘程序起始位置的距离 
 
 realloc:
         ; 循环不变式
         ; bx指向待重定位项距离段首的偏移
         ; dx:ax，每个表项4个字节，代表某个段的汇编地址
         mov dx,[bx+0x02]     
         mov ax,[bx]          
         call calc_segment_base 
         ; 将4字节的低2字节设置为该段在物理内存起始位置处的段地址
         mov [bx],ax        
         add bx,4            
         loop realloc 
 
         ; 此时已经完成了所有重定位项的重定位
         jmp far [0x04]  ; 先取2字节的段内偏移，再取4字节的段地址，构成一个物理地址，此后运行流程转移到用户程序入口点处
```



计算段基址

```

calc_segment_base:     
         ; dx:ax->ax dx:ax是一个起始物理物质，这个过程是求取这个起始物理位置的段地址，将结果放入ax中                 
         push dx ; 保存dx               
         add ax,[cs:phy_base] 
         adc dx,[cs:phy_base+0x02] 
         ; dx:ax这是0x10000，dx是0x0001 ax是0x00000
         shr ax,4 ; ax是0x0000
         ror dx,4 ; dx是0x1000
         and dx,0xf000 ; dx是0x1000
         or ax,dx ; dx的高4位是dx原来的低4位，ax原来的
         pop dx
         ret 
```

段的重定位需要将段的汇编地址转换为逻辑段地址，需要完成如下步骤，

① 计算段的物理地址

② 将段的物理地址左移4位（除以16）形成逻辑段地址

③ 将逻辑段地址回填至重定位表中