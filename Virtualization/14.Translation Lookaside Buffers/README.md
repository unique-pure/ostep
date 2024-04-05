# Note/Translation
使用分页作为支持虚拟内存的核心机制可能会导致较高的性能开销。通过将地址空间分割成小的、固定大小的单元（即**页**），分页需要大量的映射信息。由于映射信息通常存储在物理内存中，因此分页逻辑上需要对程序生成的每个虚拟地址进行额外的内存查找。在每次取指令或显式加载或存储之前访问内存以获取地址转换信息的速度非常慢。

因此我们的问题关键是：我们如何才能加快地址转换，并从总体上避免分页似乎需要的额外内存引用？需要什么硬件支持？需要什么操作系统参与？

当我们想让事情变得更快时，操作系统通常需要一些帮助。帮助通常来自操作系统的老朋友：硬件。为了加速地址转换，我们将添加一个叫做<font color="red">快表（<i>translation-lookaside buffer, TLB</i>）</font>的硬件缓存。 TLB 是芯片内存管理单元 (MMU) 的一部分，只是常用的虚拟到物理地址转换的硬件缓存；因此，更好的名称是<font color="red">地址转换缓存（<i>address-translation cache</i>）</font>。在每次虚拟内存引用时，硬件首先检查 TLB 以查看其中是否保存了所需的转换；如果是这样，则（快速）执行转换，而无需查阅页表（其中包含所有转换）。由于其巨大的性能影响，TLB 真正意义上使虚拟内存成为可能。

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Address-Translation-With-TLB.png" alt="image-20240402101148522" style="zoom:67%;" />

## 1 TLB基本算法

下段代码粗略显示了硬件如何处理虚拟地址转换，假定有一个简单的**线性页表**（即页表是一个数组）和一个**硬件管理的 TLB**（即硬件处理页表访问的大部分责任）。

```c
1:  VPN = (VirtualAddress & VPN_MASK) >> SHIFT
2:  (Success, TlbEntry) = TLB_Lookup(VPN)
3:  if (Success == True) // TLB Hit
4:      if (CanAccess(TlbEntry.ProtectBits) == True)
5:          Offset = VirtualAddress & OFFSET_MASK
6:          PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
7:          Register = AccessMemory(PhysAddr)
8:      else
9:          RaiseException(PROTECTION_FAULT)
10: else // TLB Miss
11:     PTEAddr = PTBR + (VPN * sizeof(PTE))
12:     PTE = AccessMemory(PTEAddr)
13:     if (PTE.Valid == False)
14:         RaiseException(SEGMENTATION_FAULT)
15:     else if (CanAccess(PTE.ProtectBits) == False)
16:         RaiseException(PROTECTION_FAULT)
17:     else
18:         TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits)
19:         RetryInstruction()
```

硬件遵循的算法是这样工作的：首先，从虚拟地址中提取虚拟页码（VPN）（第一行），然后检查 TLB 是否持有该 VPN 的转换（第 2 行）。如果是，我们就有一个 TLB 命中，这意味着 TLB 保留了转换。成功！现在我们可以从相关的 TLB 条目中提取物理页码 (PFN)，将其与原始虚拟地址的偏移量连接起来，形成所需的物理地址 (PA)，然后访问内存（第 5-7 行），前提是保护检查没有失败（第 4 行）。

如果 CPU 在 TLB 中没有找到转换（TLB 未命中），我们还需要做一些工作。在此示例中，硬件会访问页表以查找转换（第 11-12 行），并假设进程生成的虚拟内存引用有效且可访问（第 13、15 行），然后用转换更新 TLB（第 18 行）。这一系列操作代价高昂，主要是因为访问页表需要额外的内存引用（第 12 行）。最后，一旦更新了 TLB，硬件就会重试指令；这一次，转换可以在 TLB 中找到，内存引用也会得到快速处理。

TLB 和所有缓存一样，其建立的前提是在一般情况下，转换都能在缓存中找到（即命中）。在这种情况下，由于 TLB 位于处理核心附近，而且设计得相当快，因此几乎不会增加开销。当出现未命中时，就会产生高昂的分页成本；必须访问页表才能找到转换，这就会产生额外的内存引用（或更多，如果页表更复杂）。如果这种情况经常发生，程序的运行速度可能会明显变慢；相对于大多数 CPU 指令而言，内存访问的成本相当高，而 TLB 未命中会导致更多的内存访问。因此，我们希望尽可能避免 TLB 未命中。

## 2 示例：访问数组

为了弄清楚 TLB 的操作，让我们研究一个简单的虚拟地址跟踪，看看 TLB 如何提高其性能。在此示例中，假设内存中有一个由 10 个 4 字节整数组成的数组，从虚拟地址 100 开始。进一步假设我们有一个小的 8 位虚拟地址空间，具有 16 字节大小的页面；因此，虚拟地址分为 4 位 VPN（有 16 个虚拟页面）和 4 位偏移量（每个页面有 16 个字节）。

下图显示了系统 16 个 16 字节页面上的布局。如图所示，数组的第一个条目（a[0]）开始于（VPN=06，偏移量=04）；在这一页上只能容纳三个 4 字节的整数。数组继续进入下一页（VPN=07），在这一页上找到接下来四个条目（a[3] ... a[6]）。最后，10 个条目数组的最后三个条目（a[7] ... a[9]）位于地址空间的下一页（VPN=08）。

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Accessing-An-Array-Example-1.png" alt="image-20240402103222135" style="zoom:67%;" />

现在让我们考虑一个访问每个数组元素的简单循环，在 C 中看起来像这样：

```c
int sum = 0;
for (i = 0; i < 10; i++) {
	sum += a[i];
}
```

为了简单起见，我们假设循环生成的唯一内存访问是数组（忽略变量 i 和 sum，以及指令本身）。当访问第一个数组元素 (a[0]) 时，CPU 将看到对虚拟地址 100 的加载。硬件从中提取 VPN (VPN=06)，并使用它来检查 TLB 的有效转换。假设这是程序第一次访问数组，结果将是 TLB 未命中。

下一次访问是 a[1]，这里有一个好消息：TLB 命中！因为数组的第二个元素紧挨着第一个元素，所以它位于同一个页面上；因为我们在访问数组的第一个元素时已经访问了这个页面，所以转换已经加载到TLB中。这就是我们成功的原因。访问 a[2] 会遇到类似的成功（另一个命中），因为它也与 a[0] 和 a[1] 位于同一页面上。

不幸的是，当程序访问a[3]时，我们遇到了另一个TLB未命中。然而，下一个条目 (a[4] ... a[6]) 将再次命中 TLB，因为它们都驻留在内存中的同一页上。

最后，访问 a[7] 会导致最后一次 TLB 未命中。硬件再次查询页表以找出该虚拟页在物理内存中的位置，并相应地更新TLB。最后两次访问（a[8] 和 a[9]）受益于该 TLB 更新；当硬件在 TLB 中查找转换时，会产生另外两次命中。

让我们总结一下在对数组进行十次访问期间 TLB 的活动：**miss**、hit、hit、**miss**、hit、hit、hit、**miss**、hit、hit。因此，我们的 TLB 命中率（即命中次数除以总访问次数）为 70%。虽然这不是太高（事实上，我们希望命中率接近 100%），但它是非零的，这可能会令人惊讶。尽管这是程序第一次访问数组，但 TLB 由于<font color="red">空间局部性</font>而提高了性能。数组的元素被紧密地打包到页面中（即，它们在空间上彼此靠近），因此只有第一次访问页面上的元素才会产生 TLB 未命中。

![image-20240402105218655](https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Spatial-Locality.png)

还要注意页面大小在本例中的作用。如果页面大小是原来的两倍（32 字节，而不是 16 字节），那么阵列访问的未命中次数就会更少。由于典型的页面大小更接近 4KB，这些类型的密集、基于数组的访问实现了出色的 TLB 性能，每页访问只出现一次未命中。

关于 TLB 性能的最后一点：如果程序在循环结束后不久再次访问数组，我们很可能会看到更好的结果，前提是我们有足够大的 TLB 来缓存所需的转换：hit、hit、hit、hit、hit、hit、hit、hit、hit、hit。在这种情况下，TLB 命中率会很高，这是因为<font color="red">时间局部性</font>，即在时间上快速重新引用内存项。与其他高速缓存一样，TLB 的成功依赖于空间和时间的局部性，而空间和时间的局部性是程序的属性。如果相关程序具有这种局部性（许多程序都具有这种局部性），那么 TLB 的命中率很可能会很高。

![image-20240402105157590](https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/temporal)

## 3 谁处理TLB未命中？

我们必须回答一个问题：谁来处理 TLB 未命中？有两个可能的答案：硬件或软件（操作系统）。在过去，硬件具有复杂的指令集（有时称为<font clor="red">CISC</font>，用于复杂指令集计算机），而构建硬件的人不太信任那些鬼鬼祟祟的操作系统人员。因此，硬件将完全处理 TLB 未命中。为此，硬件必须确切地知道页表在内存中的位置（**通过页表基址寄存器**），以及它们的确切格式；如果未命中，硬件将“遍历”页表，找到正确的页表条目并提取所需的转换，用转换更新 TLB，然后重试指令。具有**硬件管理 TLB** 的“旧”架构的一个示例是 Intel x86 架构，它使用固定的**多级页表**；当前页表由CR3寄存器指向。

更现代的架构（例如，MIPS R10k 或 Sun 的 SPARC v9，RISC 或精简指令集计算机）具有所谓的<font color="red">软件管理 TLB</font>。当 TLB 未命中时，硬件只会引发异常，该异常会暂停当前指令流，将特权级别提升到内核模式，并跳转到中断处理程序。您可能会猜到，此中断处理程序是操作系统中的代码，其编写的明确目的是处理 TLB 未命中。运行时，代码将在页表中查找转换，使用特殊的“特权”指令来更新TLB，然后从中断返回；此时，硬件会重试该指令（导致 TLB 命中）。

让我们讨论几个重要的细节。首先，从中断返回指令需要与我们之前在服务系统调用时看到的从中断返回指令略有不同。在后一种情况下，从中断返回应该在中断进入操作系统后的指令处恢复执行，就像从过程调用返回到紧随过程调用之后的指令一样。在前一种情况下，当从 TLB 未命中处理中断返回时，硬件必须在导致中断的指令处恢复执行；因此，此重试让指令再次运行，这一次导致 TLB 命中。因此，根据中断或异常的引发方式，硬件在中断进入操作系统时必须保存不同的 PC，以便在时间到来时正确恢复。

其次，当运行 TLB 未命中处理代码时，操作系统需要格外小心，不要导致发生无限的 TLB 未命中链。存在许多解决方案；例如，您可以将 TLB 未命中处理程序保留在物理内存中（它们**未映射**且不受地址转换影响），或者在 TLB 中保留一些条目以进行永久有效的转换，并将其中一些永久转换槽用于处理程序代码本身;这些有线转换总是命中 TLB。

软件管理方法的主要优点是灵活性：操作系统可以使用它想要实现页表的任何数据结构，而无需更改硬件。另一个优点是简单。如下面这段代码所示，硬件在未命中时不会做太多事情：只是引发异常，然后让操作系统 TLB 未命中处理程序完成其余的工作。

```c
1: VPN = (VirtualAddress & VPN_MASK) >> SHIFT
2: (Success, TlbEntry) = TLB_Lookup(VPN)
3: if (Success == True) // TLB Hit
4:     if (CanAccess(TlbEntry.ProtectBits) == True)
5:         Offset = VirtualAddress & OFFSET_MASK
6:         PhysAddr = (TlbEntry.PFN << SHIFT) | Offset
7:         Register = AccessMemory(PhysAddr)
8:     else
9:         RaiseException(PROTECTION_FAULT)
10: else // TLB Miss
11:     RaiseException(TLB_MISS)
```

## 4 TLB 内容：里面有什么？

让我们详细了解一下硬件 TLB 的内容。典型的 TLB 可能有 32、64 或 128 个条目，并且被称为全关联。基本上，这意味着任何给定的转换都可以在 TLB 的任何位置，硬件将并行搜索整个 TLB 以找到所需的转换。一个 TLB 条目可能是这样的：

![image-20240402111725643](https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Typical-TLB-Entry-Example.png)

请注意，VPN 和 PFN 都存在于每个条目中，因为转换可能最终位于这些位置中的任何一个（在硬件术语中，TLB 称为**全关联缓存**）。硬件并行搜索条目以查看是否存在匹配。

更有趣的是“其他位”。例如，TLB通常有一个**有效位**，它表示该条目是否具有有效的转换。同样常见的是**保护位**，它决定如何访问页面（如在页表中）。例如，代码页可能被标记为读取和执行，而堆页可能被标记为读取和写入。还可能有一些其他字段，包括地址空间标识符、脏位等。

## 5 TLB 问题：上下文切换

对于 TLB，在进程（以及地址空间）之间切换时会出现一些新问题。具体来说，TLB 包含仅对当前运行的进程有效的虚拟到物理的转换；这些转换对于其他进程没有意义。因此，当从一个进程切换到另一个进程时，硬件或操作系统（或两者）必须小心，以确保即将运行的进程不会意外使用某些先前运行的进程的转换。

为了更好地理解这种情况，让我们看一个例子。当一个进程 (P1) 运行时，它假设 TLB 可能正在缓存对其有效的转换，即来自 P1 的页表的转换。对于此示例，假设 P1 的第 10 个虚拟页映射到物理页 100。

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/TLB-Context-Switching-Example1.png" alt="image-20240402112819698" style="zoom:67%;" />

在此示例中，假设存在另一个进程 (P2)，并且操作系统很快可能决定执行上下文切换并运行它。这里假设 P2 的第 10 个虚拟页映射到物理页 170。如果两个进程的条目都在 TLB 中，则 TLB 的内容将为：

![image-20240402112944190](https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/TLB-Context-Switching-Example2.png)

在上面的 TLB 中，我们显然遇到了一个问题：VPN 10 转换为 PFN 100 (P1) 或 PFN 170 (P2)，但硬件无法区分哪个条目适用于哪个进程。因此，为了让TLB能够正确、高效地支持跨多进程的虚拟化，我们还需要做更多的工作。因此，关键在于：当在进程之间进行上下文切换时，最后一个进程的 TLB 中的转换对于即将运行的进程没有意义。为了解决这个问题，硬件或操作系统应该做什么？

对于这个问题有多种可能的解决方案。一种方法是简单地在上下文切换时**刷新** TLB，从而在运行下一个进程之前清空它。在基于软件的系统上，这可以通过显式（且特权）的硬件指令来完成；使用硬件管理的 TLB，当页表基址寄存器更改时可以执行刷新（请注意，操作系统无论如何都必须在上下文切换时更改 PTBR）。无论哪种情况，刷新操作都只是将所有有效位设置为 0，本质上是清除 TLB 的内容。

通过在每次上下文切换时刷新 TLB，我们现在有了一个可行的解决方案，因为进程永远不会意外地遇到 TLB 中的错误转换。然而，这是有代价的：每次进程运行时，它在接触其数据和代码页时都必须导致 TLB 未命中。如果操作系统频繁地在进程之间切换，这个成本可能会很高。

 为了减少这种开销，一些系统添加了硬件支持以实现跨上下文切换的 TLB 共享。特别地，一些硬件系统在TLB中提供<font color="red">地址空间标识符(ASID)</font>字段。您可以将 ASID 视为**进程标识符 (PID)**，但通常它的位数较少（例如，ASID 为 8 位，而 PID 为 32 位）。

如果我们采用上面的示例 TLB 并添加 ASID，很明显进程可以轻松共享 TLB：仅需要 ASID 字段来区分其他相同的转换。以下是添加了 ASID 字段的 TLB 的描述：

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Add-the-address-space--ASID-example.png" alt="image-20240402123816119" style="zoom:67%;" />

因此，利用地址空间标识符，TLB 可以同时保存来自不同进程的转换，而不会产生任何混淆。当然，硬件还需要知道当前正在运行哪个进程才能执行转换，因此操作系统必须在上下文切换时将某些特权寄存器设置为当前进程的 ASID。

您可能还想到了另一种情况，其中 TLB 的两个条目非常相似。在此示例中，两个不同进程的两个条目具有指向同一物理页面的两个不同 VPN：例如，当两个进程共享一个页面（例如代码页）时，可能会出现这种情况。

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Another-Case-ASID-example-1.png" alt="image-20240402124140798" style="zoom:67%;" />

在上面的示例中，进程 1 与进程 2 共享物理页 101； P1 将此页映射到其地址空间的第 10 页，而 P2 将此页映射到其地址空间的第 50 页。共享代码页（在二进制文件或共享库中）很有用，因为它减少了使用的物理页数量，从而减少了内存开销。

## 6 TLB 替换策略

与任何缓存一样，对于 TLB 来说也是如此，我们必须考虑的另一个问题是**缓存替换**。具体来说，当我们在 TLB 中安装新条目时，我们必须替换旧条目，因此问题的关键是：要替换哪一个？如何设计 TLB 替换策略？ 当我们添加新的 TLB 条目时，应该替换哪个 TLB 条目？当然，目标是最大限度地减少未命中率（或提高命中率），从而提高性能。

当我们解决将页面交换到磁盘的问题时，我们将详细研究这些策略；这里我们只重点介绍一些典型的政策。

* 一种常见的方法是逐出最近最少使用的条目或 **LRU** 条目。 LRU 尝试利用内存引用流中的局部性，假设最近未使用的条目可能是驱逐的良好候选者。
* 另一种典型的方法是使用随机策略，随机驱逐 TLB 映射。这种策略非常有用，因为它简单并且能够避免极端情况行为；例如，当程序在 TLB 大小为 n 的 n + 1 个页面上循环时，诸如 LRU 之类的“合理”策略就会表现得非常不合理；在这种情况下，LRU 会错过每次访问，而随机则做得更好。

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/TLB-Replacement-Policy-Example-1.png" alt="image-20240402130437307" style="zoom:67%;" />

## 7 实际系统里的TLB

最后，让我们简单看一下真正的TLB。此示例来自 MIPS R4000 ，这是一个使用软件管理的 TLB 的现代系统；如下图所示，可以看到稍微简化的 MIPS TLB 条目。

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/MIPS-R4000-Example-1.png" alt="image-20240402130721692" style="zoom:67%;" />

MIPS R4000 支持具有 4KB 页的 32 位地址空间。因此，我们期望在我们的经典虚拟地址中有 20 位 VPN 和 12 位偏移量。然而，正如您在 TLB 中看到的，VPN 只有 19 位；事实证明，用户地址仅来自一半的地址空间（其余部分为内核保留），因此只需要 19 位的 VPN。 VPN 最多可转换为 24 位物理页号 (PFN)，因此可以支持具有高达 64GB（物理）主内存（224 个 4KB 页）的系统。 

MIPS TLB 中还有一些其他有趣的位。我们看到一个全局位（G），它用于在进程之间全局共享的页面。因此，如果设置了全局位，则 ASID 将被忽略。我们还看到 8 位 ASID，操作系统可以使用它来区分地址空间（如上所述）。最后，我们看到 3 个 Coherence (C) 位，它们决定硬件如何缓存页面；当页面被写入时标记的脏位；一个有效位，告诉硬件条目中是否存在有效的转换。还有一个页面掩码字段（未显示），支持多种页面大小；稍后我们会看到为什么更大的页面可能有用。最后，还有部分未使用。

|       Flag       |                           Content                            |
| :--------------: | :----------------------------------------------------------: |
|    19-bit VPN    |                     剩余部分保留给内核。                     |
|    24-bit PFN    | 系统支持最多64GB的主存储（$2^{24}\times 4\text{KB pages}$页面）。 |
|  Global bit(G)   |             用于在进程间共享的页面（忽略ASID）。             |
|       ASID       |               操作系统用于区分地址空间的标识。               |
| Coherence bit(C) |                    确定页面如何被硬件缓存                    |
|   Dirty bit(D)   |                     标记页面是否已被写入                     |
|   Valid bit(V)   |               告诉硬件条目中是否存在有效的转换               |

MIPS TLB 通常有 32 或 64 个这样的条目，其中大部分由用户进程在运行时使用。然而，有一些是为操作系统保留的。操作系统可以设置一个有线寄存器来告诉硬件TLB要为操作系统保留多少个插槽；操作系统将这些保留的映射用于它想要在关键时刻访问的代码和数据，此时 TLB 未命中会出现问题（例如，在 TLB 未命中处理程序中）。

由于 MIPS TLB 是由软件管理的，因此需要有更新 TLB 的指令。 MIPS 提供了四种这样的指令：

* TLBP：它探测 TLB 以查看其中是否存在特定的转换； 
* TLBR：将 TLB 条目的内容读入寄存器；
* TLBWI：替换特定的 TLB 条目； 
* TLBWR：它替换随机 TLB 条目。操作系统使用这些指令来管理 TLB 的内容。

当然，至关重要的是这些指令具有特权；想象一下，如果用户进程可以修改 TLB 的内容，它会做什么（提示：几乎任何事情，包括接管机器、运行自己的恶意“操作系统”，甚至让 Sun 消失）。

## 8 总结

我们已经看到硬件如何帮助我们更快地进行地址转换。通过提供小型专用片上 TLB 作为地址转换缓存，大多数内存引用有望在无需访问主内存中的页表的情况下得到处理。因此，在常见情况下，程序的性能几乎就像内存根本没有被虚拟化一样，这对于操作系统来说是一项出色的成就，并且对于现代系统中分页的使用当然至关重要。

然而，TLB 并没有让每个现有的程序都变得美好。特别是，如果程序在短时间内访问的页数超过了TLB所能容纳的页数，则程序将产生大量的TLB未命中，从而运行速度会变得相当慢。我们将这种现象称为<font color="red">超出 TLB 覆盖范围</font>，对于某些程序来说这可能是一个很大的问题。一个解决方案是支持更大的页面大小；通过将关键数据结构映射到程序地址空间中由较大页面映射的区域，可以增加 TLB 的有效覆盖范围。对大页面的支持通常由诸如数据库管理系统（DBMS）之类的程序利用，这些程序具有某些既大又随机访问的数据结构。

另一个值得一提的 TLB 问题是：TLB 访问很容易成为 CPU 管道中的瓶颈，特别是对于所谓的**物理索引缓存**。对于这样的缓存，必须在访问缓存之前进行地址转换，这会大大减慢速度。由于这个潜在的问题，人们研究了各种巧妙的方法来使用虚拟地址访问缓存，从而避免缓存命中时昂贵的转换步骤。这种虚拟索引缓存解决了一些性能问题，但也给硬件设计带来了新问题。

* 巨型页面（2MB 页面）

	<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Mega-pages-Example.png" alt="image-20240402132747619" style="zoom:67%;" />

* 千兆页面（1GB 页面）

	<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Giga-Pages-Example.png" alt="image-20240402132821426" style="zoom:67%;" />

* 最先进的 TLB（AMD Zen4）

	<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/State-Of-The-Art-TLB.png" alt="image-20240402132921289" style="zoom:67%;" />

# Homework
在本作业中，您将测量访问 TLB 的大小和成本。这个想法基于 Saavedra-Barrera 的工作，他开发了一种简单但美观的方法来测量缓存层次结构的多个方面，所有这些都使用一个非常简单的用户级程序。

基本思想是访问大型数据结构（例如数组）中的一定数量的页面并对这些访问进行计时。例如，假设一台机器的 TLB 大小恰好为 4（这会非常小，但对于本讨论的目的很有用）。如果您编写的程序涉及 4 个或更少的页面，则每次访问都应该是 TLB 命中，因此相对较快。然而，一旦您在循环中重复触摸 5 个或更多页面，每次访问的成本都会突然跳升，达到 TLB 未命中的程度。

循环遍历数组一次的基本代码应如下所示：

```c
int jump = PAGESIZE / sizeof(int);
for (i = 0; i < NUMPAGES * jump; i += jump) {
	a[i] += 1;
}
```

在此循环中，数组 a 的每一页都会更新一个整数，最多可达 `NUMPAGES` 指定的页数。通过重复对这样的循环进行计时（例如，在围绕此循环的另一个循环中进行几亿次，或者无论需要多少个循环才能运行几秒钟），您可以计算每次访问所需的时间（平均）。通过查找随着 `NUMPAGES` 增加而增加的成本跳跃，您可以粗略地确定第一级 TLB 有多大，确定第二级 TLB 是否存在（如果存在，则有多大），并且总体上可以很好地了解TLB 命中和未命中如何影响性能。

下图显示了随着循环中访问的页面数量增加，每次访问的平均时间。

<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Discovering-TLB-Sizes-And-Miss-Costs.png" alt="image-20240402141327724" style="zoom:67%;" />

正如您在图中所看到的，当仅访问几个页面（8 个或更少）时，平均访问时间约为 5 纳秒。当访问 16 个或更多页面时，每次访问的时间会突然跃升至约 20 纳秒。成本的最终跳跃发生在大约 1024 页时，此时每次访问大约需要 70 纳秒。从这些数据中，我们可以得出结论，存在两级 TLB 层次结构；第一个非常小（可能包含 8 到 16 个条目）；第二个较大但速度较慢（可容纳大约 512 个条目）。第一级 TLB 中的命中和未命中之间的总体差异相当大，大约为十四倍。 TLB 性能很重要！

1. 在计时方面，您需要使用计时器（例如 gettimeofday()）。这样的计时器有多精确？一个操作需要多长时间才能精确计时？(这将有助于确定在一个循环中需要重复多少次页面访问才能成功计时）

	计时器（如 C/C++ 中的 gettimeofday()）的精度取决于底层硬件和操作系统。一般来说，这些定时器可以提供微秒（10^-6 秒）甚至纳秒（10^-9 秒）精度。不过，实际可达到的精度可能会有所不同。果要对非常短的操作进行计时，可能需要在循环中多次重复操作，以累积可测量的持续时间。通过测量多次重复所需的总时间，然后对结果进行平均或分析，可以减少定时器分辨率对最终计时精度的影响。所需的迭代次数取决于定时器的精度和所定时操作的预期持续时间。

2. 编写一个名为 tlb.c 的程序，可以粗略地测量访问每个页面的成本。程序的输入应该是：要触摸的页面数和尝试的次数。

	```c
	#include <stdio.h>
	#include <stdlib.h>
	#include <time.h>
	#include <unistd.h>
	
	#define handle_error(msg)  \
	  do {                     \
	    perror(msg);           \
	    exit(EXIT_FAILURE);    \
	  } while (0)
	
	int main(int argc, char *argv[]) {
	  // Check if the correct number of command-line arguments is provided
	  if (argc < 3) {
	    fprintf(stderr, "Usage: %s pages trials\n", argv[0]);
	    exit(EXIT_FAILURE);
	  }
	
	  // Retrieve the page size
	  long PAGESIZE = sysconf(_SC_PAGESIZE); // 4096
	  // Calculate the jump size
	  long jump = PAGESIZE / sizeof(int);    // 1024
	
	  // Parse the input arguments
	  int pages = atoi(argv[1]);
	  int trials = atoi(argv[2]);
	  // Check if the input values are valid
	  if (pages <= 0 || trials <= 0) {
	    fprintf(stderr, "Invalid input\n");
	    exit(EXIT_FAILURE);
	  }
	
	  // Allocate memory for the array
	  int *a = calloc(pages, PAGESIZE);
	  // Initialize the start time
	  struct timespec start, end;
	  // Get the start time
	  if (clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &start) == -1)
	    handle_error("clock_gettime");
	
	  // Perform the specified number of trials
	  for (int j = 0; j < trials; j++) {
	    // Access each page in the array
	    for (int i = 0; i < pages * jump; i += jump)
	      a[i] += 1;
	  }
	
	  // Get the end time
	  if (clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &end) == -1)
	    handle_error("clock_gettime");
	
	  // Calculate and print the average time per access (in nanoseconds)
	  printf("%f\n",
	         ((end.tv_sec - start.tv_sec) * 1E9 + end.tv_nsec - start.tv_nsec) /
	             (trials * pages));
	
	  // Free the allocated memory
	  free(a);
	  return 0;
	}
	```

3. 现在，用您最喜欢的脚本语言（csh、python 等）编写一个脚本来运行该程序，同时将访问的页面数量从 1 变化到几千，每次迭代可能增加两倍。在不同的机器上运行脚本并收集一些数据。需要多少次试验才能获得可靠的测量结果？

	100000 次试验。

4. 接下来，绘制结果图表，制作一张与上图类似的图表。使用像 `ploticus` 甚至 `zplot` 这样的好工具。可视化通常使数据更容易理解；你认为这是为什么？

	```shell
	make
	python plot.py 14 100000 --single_cpu
	```

	<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/TLB-Size-Measurement.png" alt="Darwin_100000" style="zoom:67%;" />

5. 需要注意的一件事是编译器优化。编译器会做各种聪明的事情，包括删除循环，这些循环会增加程序其他部分随后不会使用的值。如何确保编译器不会从 TLB 大小测量器中删除上面的主循环？

	使用gcc的优化选项 `gcc -O0` 来禁用优化。这是默认设置。

	<img src="https://raw.githubusercontent.com/unique-pure/NewPicGoLibrary/main/img/Darwin_100000_close_-O0.png" alt="Darwin_100000" style="zoom:67%;" />

6. 另一件需要注意的事情是，当今大多数系统都配备了多个 CPU，当然每个 CPU 都有自己的 TLB 层次结构。为了真正获得良好的测量结果，您必须仅在一个 CPU 上运行代码，而不是让调度程序将其从一个 CPU 弹跳到下一个 CPU。你怎么能这么做呢？ 

	在 Linux 上使用 `sched_setaffinity(2)` 、 `pthread_setaffinity_np(3)` 、 `taskset(1)` 或 `sudo systemd-run -p AllowedCPUs=0 ./tlb.out` 、 `cpuset_setaffinity(2)` 或 `cpuset(1)` 在 FreeBSD 上。或者使用 `hwloc-bind package:0.pu:0 -- ./tlb.out` 。

7. 可能出现的另一个问题与初始化有关。如果在访问上面的数组 `a` 之前没有对其进行初始化，则由于需求归零等初始访问成本，第一次访问它的成本将非常昂贵。这会影响您的代码及其时间吗？您可以采取什么措施来抵消这些潜在成本？

	使用 `calloc(3)` 初始化数组然后测量时间。


