## c11_mbr.asm

```
         ;代码清单11-1
         ;文件名：c11_mbr.asm
         ;文件说明：硬盘主引导扇区代码 
         ;创建日期：2011-5-16 19:54

         ;设置堆栈段和栈指针 
         mov ax,cs      
         mov ss,ax
         mov sp,0x7c00
      
         ;计算GDT所在的逻辑段地址 
         mov ax,[cs:gdt_base+0x7c00]        ;低16位 
         mov dx,[cs:gdt_base+0x7c00+0x02]   ;高16位 
         mov bx,16        
         div bx            
         mov ds,ax                          ;令DS指向该段以进行操作
         mov bx,dx                          ;段内起始偏移地址 
      
         ;创建0#描述符，它是空描述符，这是处理器的要求
         mov dword [bx+0x00],0x00
         mov dword [bx+0x04],0x00  

         ;创建#1描述符，保护模式下的代码段描述符
         mov dword [bx+0x08],0x7c0001ff     
         mov dword [bx+0x0c],0x00409800     

         ;创建#2描述符，保护模式下的数据段描述符（文本模式下的显示缓冲区） 
         mov dword [bx+0x10],0x8000ffff     
         mov dword [bx+0x14],0x0040920b     

         ;创建#3描述符，保护模式下的堆栈段描述符
         mov dword [bx+0x18],0x00007a00
         mov dword [bx+0x1c],0x00409600

         ;初始化描述符表寄存器GDTR
         mov word [cs: gdt_size+0x7c00],31  ;描述符表的界限（总字节数减一）   
                                             
         lgdt [cs: gdt_size+0x7c00]
      
         in al,0x92                         ;南桥芯片内的端口 
         or al,0000_0010B
         out 0x92,al                        ;打开A20

         cli                                ;保护模式下中断机制尚未建立，应 
                                            ;禁止中断 
         mov eax,cr0
         or eax,1
         mov cr0,eax                        ;设置PE位
      
         ;以下进入保护模式... ...
         jmp dword 0x0008:flush             ;16位的描述符选择子：32位偏移
                                            ;清流水线并串行化处理器 
         [bits 32] 

    flush:
         mov cx,00000000000_10_000B         ;加载数据段选择子(0x10)
         mov ds,cx

         ;以下在屏幕上显示"Protect mode OK." 
         mov byte [0x00],'P'  
         mov byte [0x02],'r'
         mov byte [0x04],'o'
         mov byte [0x06],'t'
         mov byte [0x08],'e'
         mov byte [0x0a],'c'
         mov byte [0x0c],'t'
         mov byte [0x0e],' '
         mov byte [0x10],'m'
         mov byte [0x12],'o'
         mov byte [0x14],'d'
         mov byte [0x16],'e'
         mov byte [0x18],' '
         mov byte [0x1a],'O'
         mov byte [0x1c],'K'

         ;以下用简单的示例来帮助阐述32位保护模式下的堆栈操作 
         mov cx,00000000000_11_000B         ;加载堆栈段选择子
         mov ss,cx
         mov esp,0x7c00

         mov ebp,esp                        ;保存堆栈指针 
         push byte '.'                      ;压入立即数（字节）
         
         sub ebp,4
         cmp ebp,esp                        ;判断压入立即数时，ESP是否减4 
         jnz ghalt                          
         pop eax
         mov [0x1e],al                      ;显示句点 
      
  ghalt:     
         hlt                                ;已经禁止中断，将不会被唤醒 

;-------------------------------------------------------------------------------
     
         gdt_size         dw 0
         gdt_base         dd 0x00007e00     ;GDT的物理地址 
                             
         times 510-($-$$) db 0
                          db 0x55,0xaa
```

### 程序功能实现

进入保护模式例程



### 进入保护模式前的内存布局

```
gdt_size         dw 0
gdt_base         dd 0x00007e00     ;GDT的物理地址 
```

通过定义gdt_base标号，将GDT表部署在0x7E00处

```
;设置堆栈段和栈指针 
         mov ax,cs      
         mov ss,ax
         mov sp,0x7c00
```

进入MBR执行时，[CS:IP]的寄存器值为[0x0000:0x7c00]，因此此处将栈设置为[SS:SP]=[0x0000:0x7c00]



### 创建GDT表

计算GDT表逻辑地址：

```
;计算GDT所在的逻辑段地址 
         mov ax,[cs:gdt_base+0x7c00]        ;低16位 
         mov dx,[cs:gdt_base+0x7c00+0x02]   ;高16位 
         mov bx,16        
         div bx            
         mov ds,ax                          ;令DS指向该段以进行操作
         mov bx,dx                          ;段内起始偏移地址 
```

在gdt_base标号处存储的是GDT表的线性地址，而我们目前是在16位实模式下设置GDT表，所以需要先将线性地址转换为逻辑地址，也就是[段基址 : 偏移地址]的形式

设置GDT表项：

```
 ;创建0#描述符，它是空描述符，这是处理器的要求
         mov dword [bx+0x00],0x00
         mov dword [bx+0x04],0x00  

         ;创建#1描述符，保护模式下的代码段描述符
         mov dword [bx+0x08],0x7c0001ff     
         mov dword [bx+0x0c],0x00409800     

         ;创建#2描述符，保护模式下的数据段描述符（文本模式下的显示缓冲区） 
         mov dword [bx+0x10],0x8000ffff     
         mov dword [bx+0x14],0x0040920b     

         ;创建#3描述符，保护模式下的堆栈段描述符
         mov dword [bx+0x18],0x00007a00
         mov dword [bx+0x1c],0x00409600
```

设置了4个段描述符

设置GDT表大小：

```
 mov word [cs: gdt_size+0x7c00],31  ;描述符表的界限（总字节数减一）   
```

共设置了4个段描述符，共32B，因此GDT表的大小为（32 - 1 = 31）

加载GDTR：

```
lgdt [cs: gdt_size+0x7c00]
```

BIOS中设置了GDT，这是因为BIOS要检测内存1MB以上的内存信息。而且BIOS中进入过保护模式运行，并在将控制权交给MBR时重新进入了16位实模式



### 开启A20地址线

```
in al,0x92                         ;南桥芯片内的端口 
or al,0000_0010B
out 0x92,al                        ;打开A20
```

 8086只有20根地址线，只能访问1MB的内存，到了80386有32根地址线，这里就会出现一个问题，在8086时代，很多程序都会利用20位地址回绕特性（当物理地址超过0xFFFFF就会回绕到0x00000），而到了80286以后，由于地址线加多了，这个进位不会被丢弃，所以就会引发很多问题。

有两种开启A20的方法：

传统方法：早期，采用的是键盘的0x60端口和处理器的A20地址线相与，16位实模式下，只要将0x60端口的输出强制拉低，就可以确保进位被忽略，实现地址回绕；进入保护模式，就将键盘0x60端口输出为高电平，就可以开启A20，还是比较繁琐。

快速开启方法：后续的处理器增加了一个A20 mask引脚，用于控制A20地址线的开关。ICH中的0x92端口是一个8位端口，其中bit1连接在或门上，用于实现快速开启A20



### 进入保护模式

```
cli            ;保护模式下中断机制尚未建立，应 
               ;禁止中断 
mov eax,cr0
or eax,1
mov cr0,eax     ;设置PE位
```

CR0寄存器的bit 0为PE（Protection Enable）位，将该位置1，则处理器进入保护模式，开始按保护模式的规则开始运行

进入保护模式之前关闭中断，是因为保护模式下的中断机制和实模式不同，原有的中断向量表不再适用

同时需要注意的是，在保护模式下，BIOS中断也不能使用，因为他们是实模式下的代码



### 保护模式下的长跳转

```
         ;以下进入保护模式... ...
         jmp dword 0x0008:flush             ;16位的描述符选择子：32位偏移
                                            ;清流水线并串行化处理器 
         [bits 32] 

    flush:
         mov cx,00000000000_10_000B         ;加载数据段选择子(0x10)
         mov ds,cx
```

此处的jmp指令实现直接绝对远跳转，使用0x0008设置CS，使用flush的汇编地址设置EIP，dword关键字用于修饰偏移地址，要求使用32为的偏移量

设置到CS中的0x0008（0b1 0 00）为段选择符，对应段选择符的3个字段如下，

a. 描述符索引 = 1，选择第1个段描述符（从0开始）

b. TI = 0，描述符在GDT中

c. RPL = 0b00，表示最高特权级



跳转之后，会从flush标号处开始执行

同时实现，刷新描述符高速缓存器和刷新流水线



[bits 32]伪指令用于标识后续的指令均按32位模式编译



### 打印字符串

```
flush:
         mov cx,00000000000_10_000B         ;加载数据段选择子(0x10)
         mov ds,cx

         ;以下在屏幕上显示"Protect mode OK." 
         mov byte [0x00],'P'  
         mov byte [0x02],'r'
         mov byte [0x04],'o'
         mov byte [0x06],'t'
         mov byte [0x08],'e'
         mov byte [0x0a],'c'
         mov byte [0x0c],'t'
         mov byte [0x0e],' '
         mov byte [0x10],'m'
         mov byte [0x12],'o'
         mov byte [0x14],'d'
         mov byte [0x16],'e'
         mov byte [0x18],' '
         mov byte [0x1a],'O'
         mov byte [0x1c],'K'
```

此处设置到DS段选择器中的段选择符指向第2个段描述符，对应文本模式下的显示缓冲区

### 验证32位栈操作

```
;以下用简单的示例来帮助阐述32位保护模式下的堆栈操作 
         mov cx,00000000000_11_000B         ;加载堆栈段选择子
         mov ss,cx
         mov esp,0x7c00

         mov ebp,esp                        ;保存堆栈指针 
         push byte '.'                      ;压入立即数（字节）
         
         sub ebp,4
         cmp ebp,esp                        ;判断压入立即数时，ESP是否减4 
         jnz ghalt                          
         pop eax
         mov [0x1e],al                      ;显示句点 
      
  ghalt:     
         hlt                                ;已经禁止中断，将不会被唤醒 
```

栈段描述符分析：

```
  ;创建#3描述符，保护模式下的堆栈段描述符
         mov dword [bx+0x18],0x00007a00
         mov dword [bx+0x1c],0x00409600
```

段基地址=0x00000000

段界限=0x07A00

G=0，段界限以字节为单位

E=1，向下即低地址处扩展

D/B=1，32位的默认栈操作



根据对栈段描述符的分析，在这个栈段上的默认操作为32位的，此处验证的方式就是判断数据压栈后（即使只想压栈1B），ESP是否减4