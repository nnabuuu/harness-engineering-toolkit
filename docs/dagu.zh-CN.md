# 用 DAGU 监控进度

`/harness-create` 在生成 harness 的同时会生成 `dag.yaml`。如果本地安装了 [DAGU](https://dagu.readthedocs.io)，harness 会自动注册，你可以在 Web UI 中实时监控运行进度。

**你能看到什么：**
- 步骤依赖图（generator → evaluator → exit check）
- 每步的状态（运行中 / 成功 / 失败）、耗时、stdout 日志
- 跨迭代的运行历史
- 从 UI 手动重试和重新运行

**原理：** 创建 harness 时，会在 DAGU 的 DAGs 目录（`~/.dagu/dags/harness-{task-name}.yaml`）建一个指向 harness 中 `dag.yaml` 的符号链接（symlink）。DAGU 自动发现这个链接——不需要复制文件，也不需要改配置。Harness 留在 `.harness-workspace/` 里，DAGU 原地读取。

也可以手动管理注册：

```bash
bash harness.sh --register-dagu     # 重新创建 symlink
bash harness.sh --unregister-dagu   # 移除 symlink
```

**安装 DAGU（可选）：**

```bash
brew install dagu-org/brew/dagu   # macOS
dagu server                        # 启动 UI，默认 http://localhost:8080
```

不装 DAGU 也不影响使用，`bash harness.sh` 独立运行，功能完全相同。
