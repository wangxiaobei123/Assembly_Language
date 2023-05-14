# 第5章	[BX]和loop指令

#### 1)[bx]和内存单元的描述

mov	ax,[0]

将一个内存单元的内容送入ax，这个内存单元的长度为2字节（字单元），存放一个字，偏移地址为0，段地址在ds中



要完成描述一个内存单元，需要两种信息：内存单元的地址和内存单元的长度（类型）



[bx]同样也表示一个内存单元，它的偏移地址在bx中

mov	ax,[bx]

将一个内存单元的内容送入ax中，这个内存单元的长度为2字节（字单元），存放一个字，偏移地址在bx中，段地址在ds中



#### 2）loop

循环

#### 3)我们定义的描述性符号：“（）”

“（）”表示一个寄存器或一个内存单元中的内容

（ax）表示ax中的内容，（al）表示al中的内容

（）中的元素可以由三种类型：寄存器名；段寄存器名；内存物理地址（一个20位数据）。



#### 4）约定符号idata表示常量



## 5.1	[BX]

（1）int bx的含义使bx中的内容加1

（2）mov	ax, [bx]

​	bx中存放的数据作为一个偏移地址EA，段地址SA默认在DS中，将SA:EA处的数据放入ax中

即 （ax）= ((ds)*16 + (bx))



## 5.2 loop指令

（1）loop指令的格式是：loop 标号

（2）CPU执行loop指令的时候，要进行两步操作：

​		（cx）=(cx)-1;

​			判断cx中的值，不为零则转至标号处执行程序，如果为0则向下执行

（3）loop指令实现循环功能，cx中存放循环次数



```
assume	cs:code
code	segment
	mov	ax,2
	
	mov	cx,11
s:	add ax,ax
	loop	s
	
	mov ax,4c00h
	int 21h
	
code	ends
end
```



标号：s



## 5.3	在Debug中跟踪用loop指令实现的循环程序

#### 1) 计算结果

（1）计算ffff:0006 单元中的数乘以3，结果存在dx中

```
assume cs:code
code segment
	mov ax,0ffffh
	mov	ds,ax
	mov	bx,6		;以上，设置ds:bx指向ffff:6
	
	mov al,[bx]
	mov ah,0		;以上，设置（al）=((ds*16)+(bx)),(ah) = 0
	
	mov dx,0		;累加寄存器清0
	
	mov	cx,3		;循环3次
s:	add dx,dx
	loop s			;以上累加计算（ax）*3
	
	mov ax,4x00h
	int	21h			;程序返回
	
code ends
end
```

（2）在汇编源程序中，数据不能以字母开头

​			9138还可以写作“9138h”

​			A000h 应该写作“0A00h”

(3)debug模式下输入指令：g 0012

​		从当前的CS:IP指向的指令执行，一直到（IP）= 0012H为止

p指令：

​		遇到loop指令时，使用p命令来执行，debug就会自动重复执行循环中的指令，直到（cx） = 0



## 5.4 Debug和汇编编译器masm对指令的不同处理

（1）在汇编源程序中，如果用指令访问一个内存单元，则在指令中必须用“[...]”来表示内存单元，如果在“[]”里用一个常量idata直接给出内存单元的偏移地址，就要在“[]”的前面显式的给出段地址所在的段寄存器。

（2）如果在“[]”中用寄存器，比如bx，间接给出内存单元的偏移地址，则默认段地址在ds中。当然，也可以显式的给出段地址所在的段寄存器。



## 5.5 loop和[bx]的联合应用

#### 计算ffff:0~ffff:b单元中的数据的和，结果存储在dx中

（1）分析：

​		不会超出dx范围

​		不能将数据直接累加到dx中，因为其数据是8位，而dx是16位

​		不能将数据累加到dl中，因为可能会造成进位丢失



​		总结问题：类型的匹配和结果的不超界

（2）解决方法：用一个16位寄存器做中介



在指令中，不能用常量来表示偏移地址，可以将偏移地址放到bx中，用[bx]的方式访问内存单元。

```
assume cs:code
code segment
	
	mov ax,0ffffh
	mov ds,ax
	mov bx,0			;初始化ds:bx指向ffff:0
	
	mov	dx,0			;初始化累加器dx，（dx）= 0
	
	mov cx,12			;初始化循环计数寄存器cx，（cx）=12
	
s:	mov al,[bx]
	mov ah,0
	add dx,ax			;间接向dx中加上（（ds）*16+(bx)）单元的数值
	inc bx				;ds:bx指向下一个单元
	loop s
	
	mov ax,4c00h
	int 21h
code ends
end
```



## 5.6 段前缀

（1）指令“mov ax,[bx]”中，内存单元的偏移地址由bx给出，而段地址默认在ds中。

在访问内存单元的指令中显式地给出内存单元地段地址所在地段寄存器

如：

​	mov ax, ds:[bx] 

​	mov ax, cs:[bx]

​	mov ax, ss:[bx]

​	mov ax, es:[bx]

将一个内存单元的内容放入ax中，内存的那元长度位2字节，存放一个字，偏移地址为0，段地址为ds/cs/ss/es

（2）出现在访问内存单元的指令中，用于显示地指明内存单元的段地址的“ds:”“cs:”"ss:""es:"，在汇编语言中称为段前缀。



## 5.7 一段安全的空间

（1）我们需要直接向一段中写入内容

（2）这段内存空间不应该存放西戎或者其他程序的数据或者代码，否则写入操作很可能引发错误；

（3）DOS方式下，一般情况，0：200~0：2ff空间中没有系统或者其他程序的数据或者代码

（4）在写入之前，可以先用debug查看0：200~0：2ff单元的内容是否为0、



## 5.8 段前缀的使用

#### 问题：将内存ffff:0~ffff:b的内容复制到0：200~0：20b单元中

程序如下：

```
assume cs:code
code segment
	mov bx,0		;(bx)=0,偏移地址从0开始
	mov cx,12		;(cx)=12,循环12次
	
s:	mov ax,0ffffh
	mov ds,ax		;(ds) = 0ffffh
	mov dl,[bx]		;(dl)=((ds)*16+(bx)),将ffff:bx中的数据送入dl中
	
	mov ax,0020h
	mov ds,ax		;(ds)=0020h
	mov [bx],dl		;((ds)*16+(bx))=(dl),将中dl的数据送入0020：bx
	
	int bx			;(bx)=(bx)+1
	loop s
	
	mov ax,4c00h
	int 21h
code ends
end
```

改进：使用两个寄存器分别存放源始单元ffff:x和目标单元0020：x的段地址，这样可以省略循环中需要重复做12次的设置ds的程序段

```
assume cs:code
code segment
	mov ax,0ffffh
	mov ds,ax		;(ds)=0ffffh
	
	mov ax,0020h
	mov es,ax		;(es)=0020h
	
	mov bx,0		;(bx)=0,此时ds:bx指向ffff:0,es:bx指向0020：0
	
	mov cx,12
	
s:	mov dl,[bx]
	mov es:[bx],dl
	inc bx
	loop s
	
	mov ax,4c00h
	int 21h
code ends
end
```

