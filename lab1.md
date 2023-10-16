#mit lab1
##一、sleep test
这个实验是让系统休眠10s，代码如下：
```c
sleep.c
#include "kernel/types.h"
#include "user/user.h"

void sleep_test(uint32 secs)
{
    sleep(secs);
    printf("sleep %ds\n", secs);
}

int main(int argc, char *argv[])
{
    if (argc != 2) {
        printf("err:too many args\n");
        goto err_exit;
    }
    if (atoi(argv[1]) <= 0) {
        printf("err:bad time\n");
        goto err_exit;
    }
    sleep_test(atoi(argv[1]));

err_exit:
    exit(0);
}
```
在实验中，只能使用user.h里定义的api，包含system calls，ulib.c函数，代码很简单，不多赘余，重要的是如何跑起来，Makefile中修改参考fork_test，修改如下：
```Makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_sleep \

$U/_sleep:$U/sleep.o $(ULIB)
	$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $U/_sleep $U/sleep.o $U/ulib.o $U/printf.o $U/usys.o
	$(OBJDUMP) -S $U/_sleep > $U/sleep.asm 
```
首先在UPROGS中加入_sleep，之后添加_sleep编译测试，其中test.map没有生成，后续会继续修改，生成map文件。

测试指令为make qemu，sleep 10，或者./grade-lab-util sleep

二、pingpong测试
要求：父进程通过管道向子进程发送一个字节，子进程收到也向父进程发送一个字节，子进程打印pid与received ping，父进程收到后打印pid与received pong。
代码编写：
```c
pingpong.c

#include "kernel/types.h"
#include "user/user.h"
#define NULL ((void *)0)

void pingpong_test(void)
{
    int fd[2],pid;
    char buf[10];

    if(pipe(fd) == -1)
        printf("pipe error!\n");
    pid = fork();
    if (pid > 0) {
        write(fd[1],"ping",10);
        wait(0);
        read(fd[0],buf,10);
        printf("%d:received %s\n",getpid(),buf);
    } else if (pid == 0) {
        read(fd[0],buf,10);
        printf("%d:received %s\n",getpid(),buf);
        write(fd[1],buf,10);
        exit(0);
    } else {
        printf("fork error!\n");
    }
}

int main(int argc, char *argv[])
{
    pingpong_test();
    exit(0);
}
```

```Makefile
UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_sleep \
	$U/_pingpong

$U/_pingpong:$U/pingpong.o $(ULIB)
	$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $U/_pingpong $U/pingpong.o $U/ulib.o $U/printf.o $U/usys.o 
	$(OBJDUMP) -S $U/_pingpong > $U/pingpong.asm
```

##二、makefile讲解
```Makefile
#生成kernel
$K/kernel: $(OBJS) $(OBJS_KCSAN) $K/kernel.ld $U/initcode
	$(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS) $(OBJS_KCSAN)
	$(OBJDUMP) -S $K/kernel > $K/kernel.asm
	$(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym

#将kernel包含的.o通过c文件生成
$K/%.o: $K/%.c
	$(CC) $(CFLAGS) $(EXTRAFLAG) -c -o $@ $<

#initcode
$U/initcode: $U/initcode.S
	$(CC) $(CFLAGS) -march=rv64g -nostdinc -I. -Ikernel -c $U/initcode.S -o $U/initcode.o
	$(LD) $(LDFLAGS) -N -e start -Ttext 0 -o $U/initcode.out $U/initcode.o
	$(OBJCOPY) -S -O binary $U/initcode.out $U/initcode
	$(OBJDUMP) -S $U/initcode.o > $U/initcode.asm

$U/_sleep:$U/sleep.o $(ULIB)
	$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $U/_sleep $U/sleep.o $U/ulib.o $U/printf.o $U/usys.o
	$(OBJDUMP) -S $U/_sleep > $U/sleep.asm

UPROGS=\
	$U/_cat\
	$U/_echo\
	$U/_forktest\
	$U/_grep\
	$U/_init\
	$U/_kill\
	$U/_ln\
	$U/_ls\
	$U/_mkdir\
	$U/_rm\
	$U/_sh\
	$U/_stressfs\
	$U/_usertests\
	$U/_grind\
	$U/_wc\
	$U/_zombie\
	$U/_sleep \
	$U/_pingpong
```
目前并未解析qemu，留待以后。
首先是kernel那段代码，

```
$K/kernel: $(OBJS) $(OBJS_KCSAN) $K/kernel.ld $U/initcode：

这是一个规则头部，它表明了要生成的目标文件是 $K/kernel。
这个目标文件依赖于四个组件：$(OBJS)、$(OBJS_KCSAN)、$K/kernel.ld 和 $U/initcode。这表示在构建 $K/kernel 之前，需要确保这些依赖项都是最新的。
$(LD) $(LDFLAGS) -T $K/kernel.ld -o $K/kernel $(OBJS) $(OBJS_KCSAN)：

这是构建目标 $K/kernel 的命令部分。
$(LD)：这个变量通常代表链接器的名称，比如 ld。
$(LDFLAGS)：这是包含了链接器选项的变量，用于传递给链接器的参数。
-T $K/kernel.ld：这个选项指定了链接脚本文件的路径，它规定了链接器如何将不同的目标文件组合在一起。
-o $K/kernel：这个选项指定了输出文件的名字，其中 $K 代表了输出文件所在的目录。
$(OBJS) 和 $(OBJS_KCSAN)：这是目标文件的列表，它们将被链接到一起以生成可执行文件。
$(OBJDUMP) -S $K/kernel > $K/kernel.asm：

这行命令使用 $(OBJDUMP) 工具来生成汇编代码，并将其重定向到文件 $K/kernel.asm 中。
$(OBJDUMP) -t $K/kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $K/kernel.sym：

这行命令使用 $(OBJDUMP) 工具来生成目标文件的符号表，然后使用 sed 命令进行一些文本处理，最后将结果重定向到文件 $K/kernel.sym 中。
```

initcode
```
这段代码是用于生成一个名为 $U/initcode 的可执行文件。让我逐行解释其含义：

$U/initcode: $U/initcode.S：

这是一个规则头部，它表明了要生成的目标文件是 $U/initcode。
这个目标文件依赖于一个汇编文件 $U/initcode.S。这表示在构建 $U/initcode 之前，需要确保汇编文件是最新的。
$(CC) $(CFLAGS) -march=rv64g -nostdinc -I. -Ikernel -c $U/initcode.S -o $U/initcode.o：

这行命令使用 $(CC) 变量来调用 C 编译器。
$(CFLAGS)：这是包含了编译器选项的变量，用于传递给编译器的参数。
-march=rv64g：这个选项指定了目标架构为 RISC-V 64位通用指令集。
-nostdinc：这个选项告诉编译器不要包含标准库的头文件。
-I. 和 -Ikernel：这两个选项指定了额外的包含路径，以便编译器能够找到所需的头文件。
-c $U/initcode.S -o $U/initcode.o：这部分指定了要编译的源文件和输出的目标文件。
$(LD) $(LDFLAGS) -N -e start -Ttext 0 -o $U/initcode.out $U/initcode.o：

这行命令使用 $(LD) 变量来调用链接器。
$(LDFLAGS)：这是包含了链接器选项的变量，用于传递给链接器的参数。
-N：这个选项告诉链接器生成一个没有初始化数据段的可执行文件。
-e start：这个选项指定了程序的入口点。
-Ttext 0：这个选项告诉链接器将程序的文本段从地址0开始加载。
-o $U/initcode.out $U/initcode.o：这部分指定了要生成的输出文件。
$(OBJCOPY) -S -O binary $U/initcode.out $U/initcode：

这行命令使用 $(OBJCOPY) 工具将目标文件转换为二进制格式。
-S：这个选项告诉 objcopy 在复制过程中保留所有符号信息。
-O binary：这个选项指定了输出文件的格式为二进制。
$U/initcode.out $U/initcode：这部分指定了输入和输出的文件名。
$(OBJDUMP) -S $U/initcode.o > $U/initcode.asm：

这行命令使用 $(OBJDUMP) 工具生成汇编代码，并将其重定向到文件 $U/initcode.asm 中。
总的来说，这段代码用于将一个汇编文件 $U/initcode.S 编译成一个可执行文件 $U/initcode，并且生成了对应的汇编代码和二进制文件。
```

##三、添加build
修改如下
```Makefile
BUILD_DIR=$U/out

build:
ifeq ($(wildcard $(BUILD_DIR)),)
	mkdir -p $(BUILD_DIR)
	@echo "Created directory: $(BUILD_DIR)"
else
	rm -rf $(BUILD_DIR)/*
endif


$U/_sleep:$U/sleep.o $(ULIB)
	make build
	$(LD) $(LDFLAGS) -N -e main -Ttext 0 -o $U/_sleep $U/sleep.o $U/ulib.o $U/printf.o $U/usys.o -Map=$(BUILD_DIR)/sleep.map
	$(OBJDUMP) -S $U/_sleep > $(BUILD_DIR)/sleep.asm
```
在Makefile中加入build目标，检查是否存在BUILD_DIR=$U/out文件夹，存在则删除以前生成的out文件，若不存在则创建这个文件夹。此外，在make目标中加入map文件生成，并将汇编文件放到build目录中，方便查看。
