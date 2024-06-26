---
title: Tacos lab2 directions
date: 2024-04-09 10:12:16
tags: [Operating System]
categories: [Operating System]
---

# 前置知识

在 `Tacos` 中关于 `lab2` 的文档实在是少得可怜，对于实现细节以及需要的背景知识的介绍都不甚清楚，再加上运行过程中各种各样奇奇怪怪的报错，使得这个 `lab` 要比 `lab1` 难上很多。因此，在这篇文章内我想先总结一下可能需要的背景知识以及在实现过程中我遇到的报错。

<!-- more -->

这个 `lab` 中我们需要与虚拟页表和文件系统进行交互，这里先介绍一下页表。页表以页为最小单位（`4kb`）维护了虚拟地址向物理地址的映射，在操作系统中，运行的每个程序都不是直接与物理内存进行交互，每个程序都有着自己的一张页表与独立的虚拟内存。如果我们想要访问某个数据，应该使用它的虚拟地址而非物理地址。访问未映射的虚拟地址或者访问到需要更高权限的地址会导致 `PageFault`。当线程调度系统切换当前执行线程时，待执行线程对应的页表会被**激活**，任意时刻只有一份页表处于激活状态。

根据 `pintos` ，我们为每个线程维护的虚拟地址总空间为 `4G`。注意这里并不是说每个页表都占据 `4G`，而是说整个可用的虚拟地址空间大小为 `4G`，在页表内只维护某些地址的映射关系。每张页表内，高于 `3G` 的部分是内核地址，低于 `3G` 的部分是用户地址。内核地址用于存放内核数据（比如内核的代码段 `.text` 之类的）。由于对于每个用户线程，内核部分的代码和数据都是完全相同的（你当然不希望不同的线程调用相同的系统函数干的事情不一样），因此尽管每个用户线程内都给内核地址划分出了 `1G` 的空间，这部分空间指向的物理地址是完全相同的。并且，事实上页表内并不需要特别地存放这部分地址的映射关系，因为它们采用直接映射的方式：地址为 $x$ 的地方一定对应物理地址 $x$ 减去 `3G`。因此在磁盘内只需要存放一份内核数据，就可以让所有线程都能使用内核代码。

用户数据被存放在用户地址空间内，这包括从目标文件内读取出的代码段 `.text`，全局变量 `.bss, .data` 等等。用户地址的映射关系是各线程独立的：对于不同的线程，相同的用户虚拟地址对应着不同的物理地址。

我们不妨来考虑一下当某个用户线程调用 `execute(file_name, argv)` 时会发生什么事情。

- 当前线程根据 `file_name` 定位到对应的目标文件，并为新线程初始化页表，此时新页表的用户地址没有任何映射。
- 读取目标文件内记录的初始栈指针 $p$，找到空闲的物理页，在新页表中建立以 $p$ 为最高地址的虚拟页向这个物理页的映射。此时如果这个新页表被激活，那么访问 $p$​ 相当于访问这个新申请的物理页。
- 把从目标文件读取到的 `.text`，`.bss`，`.data` 等等数据放进去。
- 初始化从 $p$ 开始执行的线程，将这个线程加入 `ready` 列表，等待被调度系统 `schedule`。

但是操作系统怎么知道哪个物理页是空闲的呢？怎么把这些数据给放到新线程的虚拟地址对应的物理地址呢？这里 $p$ 指的是新线程的新页表中的虚拟地址，这个页表还没有被激活，而当前线程的页表内位于 $p$ 这个虚拟地址的地方可能记录的是其它的东西。也就是说如果直接访问 $p$​，我们并不能像想象中那样访问到新申请的那一片物理页。

`Tacos` 中解决这个问题的方法是：将所有的用户数据在内核中也放置一份。当用户想要申请一片新的物理页时，`Tacos` 会先检查在内核页表中有哪些页还没有使用过，拿出其中的某一页 $v_p$。$v_p$ 对应到的物理地址是确定的，也就是说我们通过内核页表定位到了一片没有使用过的物理页 $h_p$。接下来，在新页表中将用户地址 $p$ 与物理地址 $h_p$ 关联起来，这样即使新页表是没有激活的，由于内核页表对于所有线程都完全一样，我们在当前线程中也可以通过 $v_p$ 访问到它指向的物理地址。

另一个需要了解的背景知识是 `inode` 和 `vnode`。`inode` 用于记录文件除了名字和实际内容以外的所有信息：文件在磁盘上的位置、文件大小、是否可写等等。`inode` 本身位于磁盘上：磁盘需要专门划分一段区域存储所有的 `inode`，它与文件本身是一一对应的。`vnode` 更像是 `inode` 的一种抽象，是内核需要维护的一种数据结构：它提供了对当前活跃的文件的访问接口，记录了当前有多少线程正在写这个文件等等，是文件访问的直接媒介。多个 `File` 指向的可能是同一个 `vnode`，这也就是说就算有很多很多线程同时访问某个文件，在内核中我们仅会打开一次这个文件。在 `linux` 系统中，对于网络文件当然是没有 `inode` 的，此时 `vnode` 的存在就可以使得用户无需区分这个文件到底在本地还是在网络，都可以通过统一的接口实现对文件的访问。

在文件系统中，文件的最小储存单位是扇区：每份文件都会与扇区对齐，定位了文件开头所在的扇区也就定位了文件。内核本身维护了从文件名到对应 `inode` 的映射关系（定位文件靠的是 `inode` 而非文件名）：给定一个文件名，内核会从 `inode_table` 中定位对应的 `inode`，从而获取到文件的所有信息。

# Argument Passing

在完成这个 task 之前需要先看懂 `kernel` 是怎么在 `execute` 中加载目标文件并让它跑起来的。

当内核已经加载完这个目标文件，准备创建新线程并开始执行的时候，就是我们向程序传入参数的时机。假定新线程栈指针为 `init_sp`，它在新的页表中已经建立了到物理地址 $h_p$ 的映射，我们需要找到内核页表中指向 $h_p$ 的虚拟地址 $v_p$。需要注意的一点是 `init_sp` 一定是某一页的开头，我们初始化的是以 `init_sp` 为最高地址的某一页。

拿到 $v_p$ 后，我们就可以直接用类似 `*(v_p as *mut usize)` 的方式访问里面的内容了。在 `pintos` 文档中介绍了传入参数的格式：先从后往前依次将 `argv` 的每个字符串压入栈中（需要注意需要加一个字符 `\0`），接着需要向栈中压入 `argc + 1` 个指针：最后一个指针为 $0$，前面的指针依次指向这些字符串的开头地址。`argc` 以及第一个指针所在的虚拟地址应当分别记录在寄存器 `x[10]` 和 `x[11]` 里面（即 `main` 函数的两个参数）

在这个 `task` 中，比较常见的报错是 `kernel page fault`，或者是在一些看起来毫不相关的文件内的报错。虽然这可能表明你的程序访问到了当前活跃页表内没有建立映射的虚拟地址，但并不一定是你的问题：这个 `lab` 的栈空间非常非常地小，只有 `4KB`，而 `kprintln` 对栈空间的开销很大，所有如果你输出了很多东西，或者在 `schedule` 里面输出了，很有可能会导致 `kernel page fault` 或者一些奇奇怪怪的问题。

另一件需要注意的事情是，没有访问过的区域的默认值并不是 $0$，所以我们得手动给字符串的最后一个位置填 $0$。

```rust
unsafe fn fill<T>(pointer: usize, value: T) {
    *(pointer as *mut T) = value
}

for argv in argv.iter().rev() {
    fill(current_p - 1, 0u8);
    current_p -= argv.len() + 1;
    copy_nonoverlapping(argv.as_ptr(), current_p as *mut u8, argv.len());
    argv_p.push(current_p);
}
```

# Process Control Syscalls

当处理完上一个 task 之后，想必你已经非常熟悉虚拟内存这一套东西了，在了解完关于 `inode` 和 `vnode` 的背景知识后，这个部分就纯粹是体力活。

这里记录一下有点困难，或者可能会出问题的地方：

- `execute` 中我们需要从指针里面获取文件名，如果你的做法是将 `char` 一个一个地 `push` 到一个 `vec` 里面，然后再调用 `vec.iter().collect()` 之类的，那么你不应该把 `\0` 也 `push` 到里面去。

- 为了实现 `wait`，我们需要调整 `thread` 的结构：维护它目前还没有被回收的所有子线程以及每个子线程的运行状态（正在运行还是已经退出）。当某个线程退出的时候，更新它的父线程的对应状态。在 `wait` 内部，如果要等待的线程还在运行，就一直 `schedule`。 

- 如何优雅地检查指针，优雅地 `return -1`？我的实现方式是搞了一个宏 

  ``` rust
  macro_rules! unwrap {
      ($x: expr) => {
          match $x {
              Some(inner) => inner,
              None => return -1,
          }
      };
  }
  ```

  这样就可以很方便地把一个 `Option` 给拆开，再结合对裸指针的包装类

  ```rust
  #[derive(Clone, Copy)]
  pub struct Pointer<T>(*mut T);
  
  impl<T> From<usize> for Pointer<T> {
      fn from(value: usize) -> Self {
          Pointer(value as *mut T)
      }
  }
  
  impl<T> Pointer<T> {
      pub fn check(&self) -> Option<*mut T> {
          let current = current();
          let pt = current.pagetable.as_ref().unwrap().lock();
          let entry = pt.get_pte(self.0 as usize);
  
          entry.and_then(|e| match e.is_user() && e.is_valid() {
              true => Some(self.0),
              false => None,
          })
      }
  }
  
  impl<T: Clone> Pointer<T> {
      pub fn take(&self) -> Option<T> {
          self.check().and_then(|ptr| unsafe { Some((*ptr).clone()) })
      }
  }
  ```

  实现起来就非常舒服。

- 通过给定文件名获取或者操作对应的文件可以访问 `DISKFS.get()` 下的成员函数， `File` 这个结构体中也已经实现了 `seek`，`stream_position`，`read`，`write` 等等一系列非常有用的函数。

- 打开文件时所有可能的标识符在 `user/lib/fcntl.h` 中有定义

  ```c user/lib/fcntl.h
  #define O_RDONLY 0x000
  #define O_WRONLY 0x001
  #define O_RDWR 0x002
  #define O_CREATE 0x200
  #define O_TRUNC 0x400
  ```

- 建议关闭 `lab1` 中处理线程优先级的那一部分，它们会让本来就不太够用的栈空间雪上加霜。