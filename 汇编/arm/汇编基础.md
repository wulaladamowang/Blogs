# ARM64汇编
## 基本语法简介
带有c/c++表达式的内联汇编的格式为：
```c
#define __asm__ asm  //该条宏可能存在
#define __volatile__ volatile  //该条宏可能存在
__asm__　__volatile__("InSTructiON List" : Output : Input : Clobber/Modify);
```
1. __asm__ 或asm用来声明一个内联汇编表达式，任何内联汇编表达式都是以它开头，必不可少；
2. Instruction list是汇编指令序列，当Instruction list中有多条指令时，可以将多条指令放在一对引号中，用;或\n将它们分开，如过一条指令放一对引号中，可以每条指令一行：
   a. 每条指令都必须被双引号括起来;
   b. 两条指令必须用换行或分号分开；
3. __volatile__是GCC 关键字volatile 的宏定义;volatile 是可选的，假如用了它，则是向GCC 声明不答应对该内联汇编优化，否则当使用了优化选项(-O)进行编译时，GCC 将会根据自己的判定决定是否将这个内联汇编表达式中的指令优化掉;
4. input and output:格式如下 [xx]"xx"(xxxx)
   第一个[]中代表的是符号名，是asm code指令中要用到的，比如定义: [input1]"+r"(a)，然后在asm code中就可以直接用mov R0 %[input1], 当然，这个[]可以省略掉，当省略之后，asm code中要指代这个变量的时候，就要使用%num ， 如 mov r0 %1.
   第二个括号中的作用为限定，如下：
   |符号|作用|
   |---|---|
   | = |代表只写操作|
   | + |代表读写操作；|
   | & |占位操作，只能作为输出|
   | r |使用任何可用的寄存器|
   | m |使用变量的内存地址|
   | i |使用立即数|
   | x |操作符只能作为输出|
   1. Output 用来指定内联汇编语句的输出;
   2. Input Input 域的内容用来指定当前内联汇编语句的输进Input中;格式为形如“constraint”(variable)的列表（逗号分隔);
   ```c
   int add(int a, int b)
    {
        int c;  
    __asm__ __volatile__ (
        "MOV r0,%[input1] \n"
        "MOV r1,%[input2] \n"
        "ADD %[output1], r0, r1"
        :[output1]"+r"(c)
        :[input1]"+r"(a),[input2]"+r"(b)
        :r0,r1
        );
        return c;
    }
   ```
5. Clobber/Modify 有时候，你想通知GCC当前内联汇编语句可能会对某些寄存器或内存进行修改，希看GCC在编译时能够将这一点考虑进往。那么你就可以在Clobber/Modify域声明这些寄存器或内存。这种情况一般发生在一个寄存器出现在"Instruction List"，但却不是由Input/Output操纵表达式所指定的，也不是在一些Input/Output操纵表达式使用"r"约束时由GCC 为其选择的，同时此寄存器被"Instruction List"中的指令修改，而这个寄存器只是供当前内联汇编临时使用的情况;例如： asm ("mov R0, #0x34" : : : "R0"); 寄存器R0出现在"Instruction List中"，并且被mov指令修改，但却未被任何Input/Output操纵表达式指定，所以你需要在Clobber/Modify域指定"R0"，以让GCC知道这一点。 由于你在Input/Output操纵表达式所指定的寄存器，或当你为一些Input/Output操纵表达式使用"r"约束，让GCC为你选择一个寄存器时，GCC对这些寄存器是非常清楚的——它知道这些寄存器是被修改的，你根本不需要在Clobber/Modify域再声明它们。但除此之外， GCC对剩下的寄存器中哪些会被当前的内联汇编修改一无所知。所以假如你真的在当前内联汇编指令中修改了它们，那么就最好在Clobber/Modify 中声明它们，让GCC针对这些寄存器做相应的处理。否则有可能会造成寄存器的不一致，从而造成程序执行错误。
假如一个内联汇编语句的Clobber/Modify域存在"memory"，那么GCC会保证在此内联汇编之前，假如某个内存的内容被装进了寄存器，那么在这个内联汇编之后，假如需要使用这个内存处的内容，就会直接到这个内存处重新读取，而不是使用被存放在寄存器中的拷贝。由于这个 时候寄存器中的拷贝已经很可能和内存处的内容不一致了。 这只是使用"memory"时，GCC会保证做到的一点，但这并不是全部。由于使用"memory"是向GCC声明内存发生了变化，而内存发生变化带来的影响并不止这一点，举例：
    ```c
    int main(int __argc, char* __argv[])
    {
    int* __p = (int*)__argc;
    (*__p) = 9999;
    __asm__("":::"memory");
    if((*__p) == 9999)
        return 5;
    return (*__p);
    }
    ```
    本例中，假如没有那条内联汇编语句，那个if语句的判定条件就完全是一句空话。GCC在优化时会意识到这一点，而直接只天生return 5的汇编代码，而不会再天生if语句的相关代码，而不会天生return (__p)的相关代码。但你加上了这条内联汇编语句，它除了声明内存变化之外，什么都没有做。但GCC此时就不能简单的以为它不需要判定都知道 (__p)一定与9999相等，它只有老老实实天生这条if语句的汇编代码，一起相关的两个return语句相关代码。
    另外在linux内核中内存屏障也是基于它实现的include/asm/system.h中
    ```c
    # define barrier() *asm__volatile*("": : :"memory")
    ```
## 常用指令
|指令|含义|举例|解释|
|---|---|---|---|
|MRS|move to register from State register:将程序状态寄存器内容传送到通用寄存器|MRS R0,CPSR|传送CPSR的内容到R0|
|MSR|move to state register from register:将操作数的内容传送到程序状态寄存器的待定域中，操作数可以是通用寄存器或立即数|MSR CPSR, R0|将R0的内容传到CPSR|
|BSFL|扫描操作数op1，找到第一个非0bit位，将非0bit位的索引下标(从0开始计数)存入op2,扫描从低位到高位扫描(也就是从右->左)|bsfl %1, %0|找到%1第一个非0的bit位,注意，%0为0时，其没有意义|
## 常用寄存器
|寄存器|释义|作用|举例|
|---|---|---|---|
|DAIF|d:debugexception mask, a：serror interrupt mask, i：irq interrupt mask, f: FIQ interrut mask|||
|CPSR|当前程序状态寄存器|32位的程序状态寄存器可分为4个域；位[31：24]为条件标志位域，用f表示；位[23：16]为状态位域，用s表示；位[15：8]为扩展位域，用x表示；位[7：0]为控制位域，用c表示 |MSR CPSR_c，R0 传送R0的内容到SPSR，但仅仅修改CPSR中的控制位域|


