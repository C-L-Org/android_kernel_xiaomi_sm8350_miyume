# 归档：SUSFS Inline 钩子实验（不可刷入）

本分支是 2026-11-13 为 FinaSU v1.6.1 的 `CONFIG_KSU_HOOK_SUSFS_INLINE` 后端做的 5.4 移植实验，**已实测卡开机，禁止上机**。

## 结论
- 构建完全通过：`CONFIG_KSU_HOOK_SUSFS_INLINE=y`、`CONFIG_KSU_HOOK_TRACEPOINT is not set`、`check_susfs_inline.sh` 通过、`undefined symbol` 为 0。
- 产物 run42（venus-test-42）刷入后**卡开机**。
- 共创者经验一致：**FinaSU 的 SUSFS Inline 后端在 5.4 会 panic（init exec 路径），5.4 必须用 Tracepoint 后端**。

## 本分支改了什么（12 处直调移植）
- `fs/exec.c`：重写 `do_execveat_common`，接 `ksu_handle_execveat` / `ksu_handle_post_execveat_sucompat` / `ksu_setup_preinit_file` / `ksu_preinit_handle_post_execveat`；加 `ksu54:` ratelimited 诊断日志
- `fs/open.c`：`ksu_handle_faccessat(struct filename **)`，仅命中 `/system/bin/su` 才 `getname()/putname()`
- `fs/stat.c`：`ksu_handle_stat` / `ksu_handle_vfs_fstat` 改 `struct filename **`，仅命中才取 name
- `fs/read_write.c`：`ksu_handle_sys_read(fd)` 新签名
- `arch/arm64/configs/venus_defconfig`：`CONFIG_KSU_HOOK_SUSFS_INLINE=y`（位于 `CONFIG_KSU_SUSFS=y` 之后）

## 若要继续实验（不建议）
1. 先用 `disable_preinit=true` 二分，判断是否 preinit 改动导致卡死。
2. CI 侧需按 kernel_ref 条件启用 Inline（`--enable KSU_HOOK_SUSFS_INLINE --disable KSU_HOOK_TRACEPOINT`），不要全局改，否则 main 构建会因缺钩子触发 `$(error)`。
3. 回滚：`kernel_ref=main`。

## 可用版本
- run43 / venus-test-43：v1.6.1 + Tracepoint，实测可开机（ksud 1.6.1，7,737,200 B）。
