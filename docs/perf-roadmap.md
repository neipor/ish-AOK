# iSH-AOK 全项目模拟性能优化路线图（面向 amd64 宿主）

> 目标：在不改变访客可见语义的前提下，系统性降低解释器/JIT/内存子系统/内核路径开销。

## 0. 先做测量，再做优化（必须）

### 0.1 基准分层
- **微基准**：指令级（算术、分支、访存、字符串指令、syscall 密集）。
- **系统基准**：`make -j`、`git status`、`python startup`、`apt update`、`go test` 等真实负载。
- **回归基准**：固定镜像 + 固定 workload + 固定 CPU 频率策略，输出 p50/p95。

### 0.2 指标
- 每条访客指令成本（host cycles / guest insn）。
- TLB 命中率、跨页访存比例、`tlb_handle_miss` 占比。
- JIT 命中率（`jit_lookup` 命中/编译比）、块失效率（invalidate 频率）。
- 中断/信号路径占比、锁竞争时间（读写锁等待）。

---

## 1. 解释器（`emu/interp.c` + `emu/decode.h`）

### 1.1 解码热路径
现状：解释器每步都走 `cpu_step32`，`modrm_compute`/`modrm_decode32` 是高频入口。  
建议：
1. **把 opcode 热点做频率驱动重排**：把最常见 opcode（mov/add/cmp/jcc/test）放在 `switch` 高局部区域，减少 I-cache 抖动。
2. **多模板解码拆分**：对 `OP_SIZE=16/32` 分离编译单元，避免模板宏膨胀导致指令缓存污染。
3. **访存与标志位惰性化**：继续扩大 flags lazy-eval 覆盖面，减少每条指令写 flags 的实际次数。

关联代码：`cpu_run`、`modrm_compute`、解码宏入口。 (`emu/interp.c`)

### 1.2 指令实现层
1. **热点指令专门化 fast path**（寄存器-寄存器、无段覆盖、无异常）。
2. **减少重复寄存器映射**：`regptr_from_reg` 在循环内多次调用时可缓存（块内/指令内）。
3. **跨页访问单独慢路径**：确保常见同页访问不进入通用分支。

关联代码：`REGISTER`/`get_modrm_val`/`set_modrm_val` 宏链路。 (`emu/interp.c`)

---

## 2. TLB/MMU/内存子系统（`emu/tlb.c`、`emu/memory.c`）

### 2.1 TLB 结构优化
现状：`tlb_flush` 全表清空；miss 时 `mmu_translate` + 再判断 changes。  
建议：
1. **代际版本号（generation）+ 懒失效**：避免每次变化都全量清表。
2. **扩大/分层 TLB**：小 L0（direct-mapped）+ 大 L1（set-associative）。
3. **read/write entry 分离**：减少写权限判断分支。

关联代码：`tlb_refresh`、`tlb_flush`、`tlb_handle_miss`。 (`emu/tlb.c`)

### 2.2 内存页表路径
现状：`mem_pt` 中有重复判空与较多分支；页遍历中函数调用层次较深。  
建议：
1. **`mem_pt` 分支收敛**：单出口 + 明确无锁读策略（RCU 或读锁约束）。
2. **批量 map/unmap**：在 `pt_map/pt_unmap_always` 增加批处理，减少重复 invalidate 与 `mem_changed` 频率。
3. **COW 快路径**：常见 fork 后只读页做批标记，延迟细粒度处理。

关联代码：`mem_pt`、`pt_map`、`pt_unmap_always`、`pt_copy_on_write`。 (`emu/memory.c`)

---

## 3. JIT 子系统（`jit/jit.c`、`jit/gen.c`、`jit/gadgets-*`）

### 3.1 JIT 命中与编译开销
现状：`jit_lookup` 哈希链查找，编译路径可能频繁触发。  
建议：
1. **增加 tiny direct-mapped block cache（按 ip）**：命中直接跳过 hash 链。
2. **热块阈值编译**：冷代码继续解释执行，避免一次性编译污染。
3. **后台/延迟编译**：主线程先解释，后台线程编译并原子切换。

关联代码：`jit_lookup`、`jit_block_compile`、`jit_insert`。 (`jit/jit.c`)

### 3.2 失效策略
1. **按页脏位批失效**：减少 `jit_invalidate_range` 的链表扫描成本。
2. **跳转链接恢复优化**：`jit_block_disconnect` 里的链表遍历可分层索引。
3. **jetsam 回收策略自适应**：按内存压力和执行热度回收，而不是单一阈值。

关联代码：`jit_invalidate_range`、`jit_block_disconnect`。 (`jit/jit.c`)

### 3.3 汇编 gadget 层
1. **减少保存/恢复寄存器集**（按调用约定最小化）。
2. **return chaining 命中统计**，优化 `jit_ret_chain` 失败路径。
3. **x86_64 与 aarch64 分别做 PGO/LTO**，不要共用保守编译参数。

关联代码：`jit/gadgets-x86_64/*.S`。

---

## 4. 锁与并发（`util/sync.h`、`emu/memory.c`、`jit/jit.c`）

1. **锁分层**：把全局/大粒度锁拆成 `mmu map lock`、`jit metadata lock`、`jetsam lock`。
2. **读多写少结构改 RCU/epoch**：TLB 元数据和 page table 索引可考虑无锁读。
3. **降低 rwlock 抖动**：对高频短临界区改为 per-cpu cache + 合并写回。

---

## 5. 内核/系统调用路径（`kernel/*`）

1. **syscall fast path 白名单**：`getpid/gettid/clock_gettime` 等做极简路径。
2. **减少 `copy_to_user/copy_from_user` 次数**：批量拷贝，避免小块高频。
3. **文件系统热点缓存**：路径解析、inode 元数据、fd 查找做小缓存。

---

## 6. 编译与工具链优化

1. **LTO + PGO**：解释器/JIT 代码收益非常大。
2. **分目标编译参数**：
   - 解释器：`-O3 -fno-semantic-interposition -fno-plt`
   - JIT 生成器：倾向 `-O2`（控制编译时间）
3. **热点函数 `hot` / `flatten` / `always_inline` 审慎使用**（以 perf 结果为准）。

---

## 7. 建议的落地节奏（90 天）

### Phase A（第 1-2 周）
- 建 perf 基线、接入 benchmark CI、输出 Top10 热点函数。

### Phase B（第 3-6 周）
- TLB 代际失效 + JIT tiny cache + syscall fast path。

### Phase C（第 7-10 周）
- JIT 热块阈值编译 + invalidate 优化 + 锁拆分。

### Phase D（第 11-13 周）
- PGO/LTO 收敛、回归稳定性、跨平台参数微调。

---

## 8. 风险控制

1. 每项优化配 **功能回归 + 性能回归** 双门禁。
2. 所有“激进优化”都保留 kill-switch（编译开关或运行时开关）。
3. 对 JIT/内存一致性改动优先加断言与统计计数器，再逐步放开。

---

## 9. 预期收益（经验值，需实测验证）

- 解释器路径：5%~20%。
- TLB/MMU 路径：10%~30%（内存密集型更高）。
- JIT 命中和失效策略：15%~40%（真实业务差异较大）。
- 系统调用与锁优化：5%~15%。

> 最终收益通常不可线性叠加，但在真实混合负载中拿到 25%~60% 总体提升是有机会的（以基线测量为准）。
