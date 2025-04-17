1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 `泫炫光`和`hqm_99` 就`rv64寄存器`、`ci失败`、`简答题的深入意义`等以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

无

2. 此外，我也参考了 `ChatGPT` ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

https://chatgpt.com/


3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。


# 1. 
- `ch2b_bad_address` 向0地址写入数据报错`PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.`
原因：地址 0x0 在系统中都是未映射的虚拟地址；向0x0写入数据被看作向非法地址写入数据
- `ch2b_bad_instructions`和`ch2b_bad_register`在用户态执行特权指令报错：`IllegalInstruction in application, kernel killed it.`
原因：用户程序尝试执行 `sret` 和 `csrr` 指令 —— 这是 `RISC-V`中的特权指令只能在 S-mode（Supervisor）下执行。CPU 检测到权限不够.触 发IllegalInstruction 异常。进入 trap_handler，被识别为 Trap::Exception(Exception::IllegalInstruction)

# 2.
## 2.1: 
- 刚进入`__restore`时，sp代表内核栈的栈顶，同时也是rust中cx: &mut TrapContext的地址。
- 第一种使用场景是，trap.S执行`call trap_handler`后，继续向后执行`__restore`
- 第二种使用场景是，`trap_handler`的函数地址被保存在`TaskContext`的ra参数中

## 2.2：
- 特殊处理了`sstatus`、`sepc`、`sscratch`三个寄存器
`sstatus`是特权状态控制，专用于 S 特权级（即 supervisor mode）。它包含了一些关键的控制位，特别在发生中断/异常以及执行 sret（从 S 特权级返回）时起作用。`sepc`记录了中断/异常发生时的下一条即将执行的指令的位置
`sscratch`临时寄存器，保存用户栈的栈顶地址

# 3
- `x4` = tp（thread pointer，线程指针）一般用作线程局部数据的基址，和多线程相关的上下文信息。因为还没支持多线程，所以没有用到
- `x2` = sp （stack pointer，栈指针）它指向当前栈的栈顶，他随着sp的变化，在最后csrrw指令被修改为用户栈的栈顶

# 4
- `sp`是用户栈的栈顶，`sscratch`保存着内核栈的栈顶

# 5
- 从`sret`之后进入用户态。`sret` 是 RISC-V 架构中在 **S-mode（Supervisor 特权级）** 使用的指令，全称是：
> **Supervisor Return**

它的作用是：  
✅ **从一次 trap（中断/异常）中恢复，并返回到先前的执行上下文中（通常是用户态）**。

---

## 🧩 详细解释：`sret` 的行为

`sret` 做了三件重要的事：

1. **恢复中断使能位**：
   - 它会把 `sstatus.SPIE` 的值恢复到 `sstatus.SIE`，也就是**恢复 trap 前是否允许中断的状态**。

2. **切换回原来的特权级（通常是 U-mode）**：
   - 它会根据 `sstatus.SPP` 的值决定恢复后的运行特权级：
     - `SPP = 0` → 返回到 **U-mode**
     - `SPP = 1` → 返回到 **S-mode**

3. **跳转回原始程序位置**：
   - 它会从 `sepc` 寄存器中读取返回地址，并跳转执行（即 trap 前正在运行的那条指令）。

---

## 📦 所以一条 `sret` 相当于做了下面这些伪操作：

```c
// 恢复中断状态
sstatus.SIE = sstatus.SPIE;

// 切换回 U-mode（如果 SPP == 0）
privilege_level = sstatus.SPP == 0 ? User : Supervisor;

// 跳转到原程序位置
pc = sepc;
```

---

## 🧪 举个小例子

你设了一个系统调用，在 trap handler 结束时这样写：

```assembly
ld t0, 32*8(sp)       # 恢复 sstatus
ld t1, 33*8(sp)       # 恢复 sepc
csrw sstatus, t0
csrw sepc, t1
sret
```

这就表示：

- 恢复陷入前的 CPU 状态；
- 跳回到用户程序继续执行；
- 按照原来的中断开启/关闭设置来继续跑。

---

## 🚫 注意

`sret` 只能在 **S-mode 下执行**。  
如果你在 U-mode 或 M-mode 下尝试执行，会触发非法指令异常。

---

## ✅ 总结一句话：

> `sret` 是 RISC-V 中 trap-handler 返回用户态的标准做法，**它恢复中断状态、特权级和程序计数器**，让 CPU 继续回到被中断的那个世界。

# 7
通过`ecall`从用户态进内核态, `sret`从内核态进用户态


