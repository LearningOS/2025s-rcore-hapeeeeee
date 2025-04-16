# 1. 
- `ch2b_bad_address` 向0地址写入数据报错`PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.`
原因：地址 0x0 在系统中都是未映射的虚拟地址；向0x0写入数据被看作向非法地址写入数据
- `ch2b_bad_instructions`和`ch2b_bad_register`在用户态执行特权指令报错：`IllegalInstruction in application, kernel killed it.`
原因：用户程序尝试执行 `sret` 和 `csrr` 指令 —— 这是 `RISC-V`中的特权指令只能在 S-mode（Supervisor）下执行。CPU 检测到权限不够.触 发IllegalInstruction 异常。进入 trap_handler，被识别为 Trap::Exception(Exception::IllegalInstruction)
