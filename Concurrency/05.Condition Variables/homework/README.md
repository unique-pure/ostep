
# 大纲

本作业让你探索一些真实代码，这些代码使用锁和条件变量来实现本章讨论的各种形式的生产者/消费者队列。你将查看真实代码，在各种配置下运行它，并通过它来了解哪些可行，哪些不可行，以及其他错综复杂的问题。不同版本的代码对应着 "解决 "生产者/消费者问题的不同方法。大多数是错误的，只有一个是正确的。阅读本章，进一步了解什么是生产者/消费者问题，以及代码的一般作用。第一步是下载代码，然后键入 make 来构建所有变体。你会看到四个变体：
- `main-one-cv-while.c`: 用单一条件变量解决生产者/消费者问题。
- `main-two-cvs-if.c`: 相同，但使用了两个条件变量和一个 if 来检查是否睡眠。
- `main-two-cvs-while.c`: 相同，但使用了两个条件变量和 while 来检查是否睡眠。这是正确的版本。
- `main-two-cvs-while-extra-unlock.c`: 相同，但释放锁，然后在 fill 和 get 例程周围重新获取它。

查看“pc-header.h”也很有用，它包含所有这些不同主程序的通用代码以及 Makefile，以便正确构建代码。
每个程序都使用以下标志：
- `-l <number of items each producer produces>`
- `-m <size of the shared producer/consumer buffer>`
- `-p <number of producers>`
- `-c <number of consumers>`
- `-P <sleep string: how producer should sleep at various points>`
- `-C <sleep string: how consumer should sleep at various points>`
- `-v [verbose flag: trace what is happening and print it]`
- `-t [timing flag: time entire execution and print total time]`

前四个参数相对不言自明：`-l`指定每个生产者应循环多少次（因此每个生产者产生多少个数据项），`-m`控制共享缓冲区的大小（大于或等于1），`-p`和`-c`分别设置生产者和消费者的数量。

更有趣的是两个睡眠字符串，一个用于生产者，一个用于消费者。通过这些标志，你可以让每个线程在执行过程中的某些时刻休眠，从而切换到其他线程；这样做可以让你玩转每个解决方案，也许还能找出具体问题，或研究生产者/消费者问题的其他方面。字符串的说明如下。例如，如果有三个生产者，字符串应分别指定每个生产者的睡眠时间，并用冒号分隔。这三个生产者的睡眠时间字符串应该是这样的

```sh
  sleep_string_for_p0:sleep_string_for_p1:sleep_string_for_p2
```

每个睡眠字符串又是一个逗号分隔的列表，用于决定代码中每个睡眠点的睡眠时间。为了更好地理解每个生产者或消费者的睡眠字符串，我们来看看 `main-two-cvs-while.c` 中的代码，特别是生产者代码。在这段代码中，生产者线程循环一段时间，通过调用 `doo_fill()` 将元素放入共享缓冲区。在这个填充例程周围有一些等待和信号代码，以确保在生产者尝试填充时缓冲区没有满。
```c
void *producer(void *arg) {
    int id = (int) arg;
    int i;
    for (i = 0; i < loops; i++) {   p0;
        Mutex_lock(&m);             p1;
        while (num_full == max) {   p2;
            Cond_wait(&empty, &m);  p3;
        }
        do_fill(i);                 p4;
        Cond_signal(&fill);         p5;
        Mutex_unlock(&m);           p6;
    }
    return NULL;
}
```

从代码中可以看到，有许多标有 p0、p1 等的点。这些点是代码可以休眠的地方。消费者代码（此处未显示）也有类似的等待点（c0 等）。为生产者指定睡眠字符串非常简单：只需确定生产者在每个点 `p0、p1、...、p6` 的睡眠时间。例如，字符串 "1,0,0,0,0,0,0 "表示生产者应该在标记 p0 处（抓取锁之前）休眠 1 秒钟，然后在循环中每次都不休眠。现在让我们展示一下运行这些程序的输出结果。首先，假设用户只运行一个生产者和一个消费者。我们完全不添加任何睡眠（这是默认行为）。在本例中，缓冲区大小设置为 2 (`-m 2`)。首先，让我们构建代码：

```sh
prompt> make
gcc -o main-two-cvs-while main-two-cvs-while.c -Wall -pthread
gcc -o main-two-cvs-if main-two-cvs-if.c -Wall -pthread
gcc -o main-one-cv-while main-one-cv-while.c -Wall -pthread
gcc -o main-two-cvs-while-extra-unlock main-two-cvs-while-extra-unlock.c -Wall -pthread
```

现在我们运行代码：

```sh
prompt> ./main-two-cvs-while -l 3 -m 2 -p 1 -c 1 -v
```

在这种情况下，如果跟踪代码（使用 verbose 标志 -v），就会在屏幕上看到以下输出（或类似输出）：

```sh
 NF             P0 C0
  0 [*---  --- ] p0
  0 [*---  --- ]    c0
  0 [*---  --- ] p1
  1 [u  0 f--- ] p4
  1 [u  0 f--- ] p5
  1 [u  0 f--- ] p6
  1 [u  0 f--- ] p0
  1 [u  0 f--- ]    c1
  0 [ --- *--- ]    c4
  0 [ --- *--- ]    c5
  0 [ --- *--- ]    c6
  0 [ --- *--- ]    c0
  0 [ --- *--- ] p1
  1 [f--- u  1 ] p4
  1 [f--- u  1 ] p5
  1 [f--- u  1 ] p6
  1 [f--- u  1 ] p0
  1 [f--- u  1 ]    c1
  0 [*---  --- ]    c4
  0 [*---  --- ]    c5
  0 [*---  --- ]    c6
  0 [*---  --- ]    c0
  0 [*---  --- ] p1
  1 [u  2 f--- ] p4
  1 [u  2 f--- ] p5
  1 [u  2 f--- ] p6
  1 [u  2 f--- ]    c1
  0 [ --- *--- ]    c4
  0 [ --- *--- ]    c5
  0 [ --- *--- ]    c6
  1 [f--- uEOS ] [main: added end-of-stream marker]
  1 [f--- uEOS ]    c0
  1 [f--- uEOS ]    c1
  0 [*---  --- ]    c4
  0 [*---  --- ]    c5
  0 [*---  --- ]    c6

Consumer consumption:
  C0 -> 3
```

在描述这个简单示例中发生的事情之前，让我们先了解一下输出中对共享缓冲区的描述，如左图所示。起初它是空的（一个空槽用 `---` 表示，两个空槽用 `[*--- --- ]` 表示）；输出还显示了缓冲区中的条目数（`num_full`），起始值为 0。你可能还会注意到一些额外的标记：u 标记显示了 `use_ptr`所在的位置（这是下一个消耗值的消费者将从这里获得值的位置）；同样，f 标记显示了 `fill_ptr` 所在的位置（这是下一个生产者将生产值的位置）。当你看到 `*` 标记时，它只是表示 `use_ptr` 和 `fill_ptr` 指向了同一个槽。沿着右边，你可以看到每个生产者和消费者即将执行的步骤的跟踪。例如，生产者会抓取锁（`p1`），然后，由于缓冲区有一个空槽，生产者会在其中产生一个值（`p4`）。然后，生产者继续直到释放锁 (`p6`)，然后尝试重新获取锁。在本例中，消费者获取了锁，并消耗了值（`c1, c4`）。请进一步研究该跟踪，以了解生产者/消费者解决方案是如何在单个生产者和消费者之间运行的。现在，让我们添加一些暂停来改变跟踪的行为。在这种情况下，假设我们想让生产者休眠，以便消费者可以先运行。我们可以通过以下方法实现这一目的： 

```sh
prompt> ./main-two-cvs-while -l 1 -m 2 -p 1 -c 1 -P 1,0,0,0,0,0,0 -C 0 -v

The results:
 NF             P0 C0
  0 [*---  --- ] p0
  0 [*---  --- ]    c0
  0 [*---  --- ]    c1
  0 [*---  --- ]    c2
  0 [*---  --- ] p1
  1 [u  0 f--- ] p4
  1 [u  0 f--- ] p5
  1 [u  0 f--- ] p6
  1 [u  0 f--- ] p0
  1 [u  0 f--- ]    c3
  0 [ --- *--- ]    c4
  0 [ --- *--- ]    c5
  0 [ --- *--- ]    c6
  0 [ --- *--- ]    c0
  0 [ --- *--- ]    c1
  0 [ --- *--- ]    c2
 ...
```

正如你所看到的，生产者点击代码中的 `p0` 标记，然后从其睡眠规范中抓取第一个值（在本例中为 1），因此每个生产者在尝试抓取锁之前都会睡眠 1 秒钟。这样，消费者开始运行，抓取锁，但发现队列是空的，于是休眠（释放锁）。然后生产者运行（最终），一切如你所料。请注意：必须为每个生产者和消费者提供睡眠规范。因此，如果你创建了两个生产者和三个消费者（使用 `-p 2 -c 3`），就必须为每个生产者和消费者指定睡眠字符串（例如，`-P 0:1` 或 `-C 0,1,2:0:3,3,3,1,1,1`）。睡眠字符串的长度可以短于代码中睡眠点的数量；剩余的睡眠时间段将被初始化为 0。

# Homework
1. 第一个问题的重点是 `main-two-cvs-while.c`（工作解决方案）。首先，研究代码。你是否了解运行程序时应该发生什么？

	程序定义了两个条件变量 `empty` 和 `fill`，以及一个互斥量 `m`。它们分别用于控制生产者和消费者线程的同步和互斥。

	生产者线程通过不断地向缓冲区中填充数据，直到达到循环次数 `loops`。在填充数据之前，它会获取互斥锁 `m`，并检查缓冲区是否已满。如果缓冲区已满，则会等待条件变量 `empty`，直到有消费者消费了一些数据并通知。填充数据后，生产者发送信号通知消费者线程缓冲区已经填充，然后释放互斥锁`m`。

	而消费者线程不断地从缓冲区中取出数据，直到取出的数据为 `END_OF_STREAM`。在取出数据之前，它会获取互斥锁 `m`，并检查缓冲区是否为空。如果缓冲区为空，则会等待条件变量 `fill`，直到有生产者向缓冲区中填充了数据并通知。取出数据后，消费者发送信号通知生产者线程缓冲区已经有空间，然后释放互斥锁。

	对于`main-common.c`，首先就是解析命令行参数，然后根据参数创建多个消费者生产者线程，并进行模拟。

2. 使用一名生产者和一名消费者运行，并让生产者产生一些值。从缓冲区（大小 1）开始，然后增加它。代码的行为如何随着缓冲区的增大而变化？ （或者确实如此？）您预测 `num_full` 具有不同的缓冲区大小（例如， `-m 10` ）和不同数量的生产项目（例如， ` -l 100` ），当您将消费者睡眠字符串从默认（nosleep）更改为 `-C 0,0,0,0,0,0,1` 时？

	```c
	$ ./main-two-cvs-while -p 1 -c 1 -m 1 -v
	$ ./main-two-cvs-while -p 1 -c 1 -m 10 -v
	$ ./main-two-cvs-while -p 1 -c 1 -m 1 -l 100 -v
	$ ./main-two-cvs-while -p 1 -c 1 -m 1 -C 0,0,0,0,0,0,1 -v
	$ ./main-two-cvs-while -p 1 -c 1 -m 10 -l 10 -C 0,0,0,0,0,0,1 -v
	```

3. 如果可能，请在不同的系统（例如 Mac 和 Linux）上运行代码。您是否发现这些系统有不同的行为？

	```shell
	$ ./main-two-cvs-while -p 1 -c 3 -l 6 -v
	```

	

	在 Linux 5.14.4 上，大多数时候单个消费者会获取所有值，但在 macOS 11.6 上偶尔会发生这种情况。

4. 让我们看看一些时间安排。您认为以下执行（有一个生产者、三个消费者、一个单条目共享缓冲区，并且每个消费者在 `c3` 点暂停一秒钟）需要多长时间？ `./main-two-cvs-while -p 1 -c 3 -m 1 -C 0,0,0,1,0,0,0:0,0,0,1,0,0,0:0,0,0,1,0,0,0 -l 10 -v -t`

	11秒。它在线程重新获取锁后休眠。

5. 现在将共享缓冲区的大小更改为 3 ( `-m 3` )。这会对总时间产生影响吗？

	有11个c3，所以11秒

6. 现在将睡眠位置更改为 `c6` （这模拟了消费者从队列中取出某些内容，然后对其执行某些操作），再次使用单条目缓冲区。在这种情况下，您预计什么时间？ `./main-two-cvs-while -p 1 -c 3 -m 1 -C 0,0,0,0,0,0,1:0,0,0,0,0,0,1:0,0,0,0,0,0,1 -l 10 -v -t`

	它调用 `sleep` 12 次，但只用了 5 秒。这是因为它在释放锁后会休眠，以便其他线程可以继续前进。

7. 最后，再次将缓冲区大小更改为 3 ( `-m 3` )。你预测现在时间？

	有13个c6，但还有5秒

8. 现在让我们看看 `main-one-cv-while.c` 。假设有一个生产者、一个消费者和一个大小为 1 的缓冲区，您能否配置一个睡眠字符串，从而导致此代码出现问题？

	不能

9. 现在将消费者数量更改为两个。您能否为生产者和消费者构造睡眠字符串，从而导致代码出现问题？

	```c
	// all threads sleeping, like the situation in Figure 30.11
	$ ./main-one-cv-while -c 2 -v -P 0,0,0,0,0,0,1
	```

10. 现在检查 `main-two-cvs-if.c` 。你能让这段代码出现问题吗？再次考虑只有一个消费者的情况，然后考虑有多个消费者的情况。

	在c3处就绪，但运行时没有数据。始终使用 `while` 。

11. 最后，检查 `main-two-cvs-while-extra-unlock.c` 。如果在执行 put 或 get 操作之前释放锁，会出现什么问题？考虑到睡眠字符串，你能可靠地导致这样的问题发生吗？会发生什么坏事呢？

	```
	$ ./main-two-cvs-while-extra-unlock -p 1 -c 2 -m 10 -l 10 -v -C 0,0,0,0,1,0,0:0,0,0,0,0,0,0
	```

	第一个消费者只消费一个值。


