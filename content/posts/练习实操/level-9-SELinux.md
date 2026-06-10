+++
title = 'Level 9：修复一个 SELinux 权限问题'
date = '2026-06-09T14:03:00+08:00'
draft = false
+++

# Level 9：修复一个 SELinux 权限问题

## 我想做

当一个服务因为 SELinux 权限被拒绝而启动失败时，能独立定位并修复。这是 Android 嵌入式开发中最常见的调试场景。

## 先回答三个问题

1. **SELinux 在 Android 上到底保护什么？**
   - 没有 SELinux：一个进程拿到 root 就可以做任何事（读所有文件、杀其他进程、控制硬件）
   - 有 SELinux：每个进程被限制在最小的权限域中（principle of least privilege）
   - 即使进程是 root 用户，SELinux 仍然会拒绝未经授权的操作
   - avc: denied = SELinux 告诉你"这个操作不在你的权限范围内"

2. **domain 和 type 是什么关系？**
   - domain 是进程的标签（你是什么）
   - type 是文件/资源的标签（这是什么）
   - `allow domain type:class { permission }` = "谁 对什么 做什么"

3. **audit2allow 的原理？**
   - 从 `dmesg` 或 `/proc/avc` 中读取 avc denied 日志
   - 自动解析出需要的 `allow` 语句
   - 但注意：`audit2allow` 只是工具，直接套用可能给过多权限，要人工审查

## 需要懂的知识

**avc denied 日志解读**

```
avc: denied { read } for pid=1234 comm="luojService"
  name="secret_file" dev="mmcblk0p5" ino=5678
  scontext=u:r:luojService:s0
  tcontext=u:object_r:secret_data:s0
  tclass=file permissive=0
```

逐字段：
- `denied { read }` —— 被拒绝的操作
- `pid=1234 comm="luojService"` —— 哪个进程
- `scontext=u:r:luojService:s0` —— 进程的 domain（luojService）
- `tcontext=u:object_r:secret_data:s0` —— 目标的类型（secret_data）
- `tclass=file` —— 目标类型分类（文件）
- `permissive=0` —— 是否真的拒绝了（1=只警告没拦住，0=真的拦了）

转化为 allow 规则：
```te
allow luojService secret_data:file { read };
```

**TE 文件的位置**

```bash
ls device/amlogic/ross/sepolicy/vendor/
```

## 动手方案

```bash
# 第 1 步：复现一个 SELinux 错误
# 用 Level 3 的 luojService，故意去掉它的 .te 文件
# 或者在 luojService 中访问一个它没权限的文件

# 第 2 步：抓取 avc 错误
adb shell dmesg | grep avc | grep luojService

# 第 3 步：现场生成允许规则
adb shell dmesg | grep avc | grep luojService | audit2allow

# 第 4 步：写入 .te 文件
echo "allow luojService some_type:file { read };" >> device/amlogic/ross/sepolicy/vendor/luojService.te

# 第 5 步：编译烧录（参考 level-2）

# 第 6 步：验证
adb shell dmesg | grep avc | grep luojService
```

## 验收清单

| 项目 | 验证方式 |
|------|----------|
| 能制造 SELinux 错误 | 故意去掉权限，`dmesg \| grep avc` 看到 denied |
| 能修复 | 补上 .te 后错误消失 |
| 能解读日志 | 你能说出一条 avc denied 日志中每个字段的含义 |
