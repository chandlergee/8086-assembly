1. call和ret指令都是转移指令，它们都修改IP，或同时修改CS和IP。它们经常被共同用来实现子程序的设计。

2. ret指令用栈中的数据，修改IP的内容，从而实现近转移。

   retf指令用栈中的数据，修改CS和IP的内容，从而实现远转移。

   CPU执行ret指令时，进行两步操作：

   1. (IP)=((ss) * 16 + (sp))
   2. (sp)=(sp)+2

   CPU执行retf指令时，进行四步操作：

   1. (IP)=((ss) * 16 + (sp))
   2. (sp)=(sp)+2
   3. (CS)=((ss) * 16 + (sp))
   4. (sp)=(sp) + 2

   CPU执行ret指令时，相当于进行：

   ```assembly
   pop IP
   ```

   CPU执行retf指令时，相当于执行:

   ```assembly
   pop IP
   pop CS
   ```

   ```assembly
   assume cs:code
   
   stack segment
   	db 16 dup(0)
   stack ends
   
   code segment
   	mov ax,4c00H
   	int 21H
   	
   start:mov ax,stack
   	  mov ss,ax
   	  mov sp,16
   	  mov ax,0
   	  push ax
   	  mov bx,0
   	  ret
   code ends
   end start
   ```

3. CPU执行call指令时，进行两个步骤：

   1. 将当前的IP或CS和IP压入栈中；
   2. 转移

   call不能实现短转移，除此之外，call指令实现转移的方法和jmp指令的原理相同。

4. 依据位移进行转移的call指令

   `call 标号（将当前的IP入栈后，转到标号处执行指令）`

   CPU执行此格式的call指令后，进行如下操作：

   1. (sp)=(sp)-2
   2. ((ss) * 16 + (sp)) = (IP)
   3. (IP) =(IP)+16位位移

   16位位移=标号处的地址-call指令后的第一个字节的地址

   16位位移的范围是-32768~32767，用补码表示

   16位位移由编译程序在编译时算出

   CPU执行“call 标号”时，相当于进行：

   ```assembly
   push IP
   jmp near ptr 标号
   ```

5. 转移的目的地址在指令中的call指令

   `call far ptr 标号`实现的是段间转移

   CPU执行此格式的call指令时，进行如下操作：

   1. (sp)=(sp)-2
   2. ((ss) * 16 + (sp)) = (CS)
   3. (sp)=(sp)-2
   4. ((ss) * 16 + (sp))=(IP)
   5. (CS)=标号所在段的段地址
   6. (IP)=标号在段中的偏移地址

   CPU执行此call指令时，相当于：

   ```assembly
   push CS
   push IP
   jmp far ptr 标号
   ```

6. 转移地址在寄存器中的call指令

   指令格式：`call 16位reg`

   功能：

   1. (sp)=(sp)-2
   2. ((ss)*16+(sp))=IP
   3. (IP)=(16位reg)

   此时，相当于：

   ```assembly
   push IP
   jmp 16位reg
   ```

7. 转移地址在内存中的call指令

   1. `call word ptr 内存单元地址`

      用汇编表示为：

      ```assembly
      ;push IP
      ;jmp word ptr 内存单元地址
      
      ;例子
      mov sp,10H
      mov ax,123H
      mov ds:[0],ax
      call word ptr ds:[0]
      ```

   2. `call dword ptr 内存单元地址`

      用汇编表示：

      ```assembly
      ;push CS
      ;push IP
      ;jmp dword ptr 内存单元地址
      
      mov sp,10H
      mov ax,0123H
      mov ds:[0],ax
      mov word ptr ds:[2],0
      call dword ptr ds:[0]
      ```

8. call和ret的配合使用

   ```assembly
   assume cs:code
   code segment
   start:mov ax,1
   	  mov cx,3
   	  call s
   	  mov bx,ax
   	  mov ax,4c00H
   	  int 21H
   	s:add ax,ax
   	  loop s
   	  ret
   code ends
   end start
   ```

   1. CPU将`call s`指令的机器码读入，IP指向`call s`后的指令`mov bx,ax`，然后，CPU指向`call s`指令，将当前IP的值压栈，并且将IP的值改变为标号s处的偏移地址。
   2. CPU从标号s处开始执行指令，loop循环完毕后，(ax)=8。
   3. CPU将ret指令的机器码读入，IP指向了ret指令后的内存单元，然后CPU执行ret指令，从栈中弹出一个值，送入IP中，则CS:IP指向指令`mov bx,ax`。
   4. CPU从`mov bx,ax`开始执行指令，直至完成。

9. mul指令：乘法指令，需要注意两点：

   1. 两个相乘的数，要么都是8位，要么都是16位。如果是8位，一个默认放在AL中，另一个放在8位reg或内存字节单元中；如果是16位，一个默认放在AX中，另一个放在16位reg或内存字单元中。
   2. 结果：如果是8位乘法，结果默认放在AX中，如果是16位乘法，结果高位默认放在DX中，低位放在AX中。

   格式如下：

   ```assembly
   mul reg
   mul 内存单元
   ```

   内存单元可以用不同的寻址方式给出：比如

   `mul byte ptr ds:[0]`

   含义：(ax)=(al)\*((ds)\*16+0)

   `mul word ptr [bx+si+8]`

   含义：(ax)=(ax)\*((ds)\*16+(bx)+(si)+8)结果的低16位

   ​	       (dx)=(ax)\*((ds)\*16+(bx)+(si)+8)结果的高16位

10. call和ret指令共同支持了汇编语言编程中的模块化设计。

11. 子程序参数和结果传递

    1. 用寄存器来存储参数和结果

       ```assembly
       ;计算data段中的第一组数据的3次方
       ;结果存放在后一组dword单元中
       
       assume cs:code,ds:data
       
       data segment
       	dw 1,2,3,4,5,6,7,8
       	dd 0,0,0,0,0,0,0,0
       data ends
       
       code segment
       start:mov ax,data
       	  mov ds,ax
       	  mov si,0
       	  mov di,16
       	  
       	  mov cx,8
       	s:mov bx,[si]
       	  call cube
       	  mov [di],ax
       	  mov [di].2,dx
       	  add si,2
       	  add di,4
       	  loop s
       	  
       	  mov ax,4c00H
       	  int 21H
        cube:mov ax,bx
             mul bx
             mul bx
             ret
       code ends
       end start
       ```

    2. 将数据放在内存中，然后将内存空间的首地址放在寄存器中。

       ```assembly
       assume cs:code,ds:data
       
       data segment
       	db 'conversation'
       data ends
       
       code segment
       start:mov ax,data
       	  mov ds,ax
       	  mov si,0
       	  mov cx,12
       	  call capital
       	  
       	  mov ax,4c00H
       	  int 21H
       	  
       capital:and byte ptr [si],11011111b
               inc si
               loop capital
               ret
       code ends
       end start
       ```

    3. 用栈来传递参数

12. 解决寄存器冲突问题：在子程序的开始将子程序中所用到的寄存器中的内容都保存起来，在子程序返回前再恢复。

    编写子程序的标准框架：

    ```assembly
    子程序开始:子程序中使用的寄存器入栈
    		 子程序内容
    		 子程序中使用的寄存器出栈
    		 返回(ret、retf)
    ```

    























   

   

