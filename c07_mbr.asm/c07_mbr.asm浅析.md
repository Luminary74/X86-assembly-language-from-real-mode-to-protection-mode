## c07_mbr.asm

```
         ;代码清单7-1
         ;文件名：c07_mbr.asm
         ;文件说明：硬盘主引导扇区代码
         ;创建日期：2011-4-13 18:02
         
         jmp near start
	
 message db '1+2+3+...+100='
        
 start:
         mov ax,0x7c0           ;设置数据段的段基地址 
         mov ds,ax

         mov ax,0xb800          ;设置附加段基址到显示缓冲区
         mov es,ax

         ;以下显示字符串 
         mov si,message          
         mov di,0
         mov cx,start-message
     @g:
         mov al,[si]
         mov [es:di],al
         inc di
         mov byte [es:di],0x07
         inc di
         inc si
         loop @g

         ;以下计算1到100的和 
         xor ax,ax
         mov cx,1
     @f:
         add ax,cx
         inc cx
         cmp cx,100
         jle @f

         ;以下计算累加和的每个数位 
         xor cx,cx              ;设置堆栈段的段基地址
         mov ss,cx
         mov sp,cx

         mov bx,10
         xor cx,cx
     @d:
         inc cx
         xor dx,dx
         div bx
         or dl,0x30
         push dx
         cmp ax,0
         jne @d

         ;以下显示各个数位 
     @a:
         pop dx
         mov [es:di],dl
         inc di
         mov byte [es:di],0x07
         inc di
         loop @a
       
         jmp near $ 
       

times 510-($-$$) db 0
                 db 0x55,0xaa
```



### 程序实现功能

从1加到100并显示结果



### 显示字符串

```
jmp near start ;越过数据区
	
 message db '1+2+3+...+100=' ;声明要显示的字符串
        
 start:
    mov ax,0x7c0 ;设置数据段的段基地址 
    mov ds,ax
 
    mov ax,0xb800 ;设置附加段基址到显示缓冲区
    mov es,ax
 
    ;以下显示字符串 
    mov si,message ; 源地址偏移为message汇编地址         
    mov di,0       ; 目的地址偏移为显存起始地址
    mov cx,start-message ; 循环次数为要显示的字节数
@g:
    mov al,[si]
    mov [es:di],al ;要显示的字符
    inc di ;递增显存索引
    mov byte [es:di],0x07 ;设置显示属性
    inc di  ;递增显存索引
    inc si  ;指向下一个要显示的字符
    loop @g ;构成循环
```



### 计算1到100的累加和

```
    ;以下计算1到100的和 
    xor ax,ax ;清空AX寄存器，用于保存累加和
    mov cx,1  ;加数
@f:
    add ax,cx
    inc cx
    cmp cx,100
    jle @f ;当CX <= 100，继续循环
```

这里需要注意，AX寄存器可以容纳的无符号数最大为65535，此处累加和为5050，所以不会越界


说明：如何求1到1000的累加和

1到1000的累加和为500500，超过了AX寄存器所能容纳的无符号数范围，所以需要使用DX : AX保存累加和，同时需要使用ADC指令，处理低16位累加可能产生的进位**



### 累加和数位的分解和显示

```
   ;以下计算累加和的每个数位 
    xor cx,cx ;设置堆栈段的段基地址
    mov ss,cx ;ss = 0x0
    mov sp,cx ;sp = 0x0
 
    mov bx,10 ;分解数位时使用的除数
    xor cx,cx ;保存分解的数位个数
@d:
    inc cx
    xor dx,dx  ;清空DX寄存器，AX构成被除数
    div bx
    or dl,0x30 ;将数位转换为数字字符
    push dx    ;将分解出的数位数字字符入栈
    cmp ax,0
    jne @d     ;如果商不为0，则继续分解；否则不再分解
 
    ;以下显示各个数位 
@a:
    pop dx                ;分解数位从高位出栈
    mov [es:di],dl        ;显示字符
    inc di
    mov byte [es:di],0x07 ;设置显示属性
    inc di
    loop @a ;CX中保存了分解的数位个数，正好作为循环控制变量
```

这里和上一章类似，不过借用的是栈。



![image-20220929190839022](https://luminary74-1312852271.cos.ap-beijing.myqcloud.com/202209291908482.png)