# Ubuntu VS Code 桌面启动 Codex 完整修复指南

> 目标：解决 Ubuntu 中“终端执行 `code` 可以正常使用 Codex，但点击桌面 / Dock 图标后 Codex 异常”的问题，并整理为唯一、稳定的 VS Code 启动入口。
>
> 本文是在原《Ubuntu VS Code 桌面图标启动 Codex 修复》基础上的完整扩展版。

---

## 1. 已验证现象

在本机 ASUS / Ubuntu 22.04 环境中，曾实测：

| 启动方式 | 结果 |
|---|---|
| `code` | Codex 正常 |
| `/usr/bin/code` | Codex 正常 |
| `/usr/share/code/code` | 曾出现 Codex WebView 异常 |
| Ubuntu 原始桌面入口 | 曾出现 Codex 异常 |
| 代理环境 + `/usr/bin/code` | Codex 正常，OpenAI 登录正常 |
| 自定义 `code-proxy` + `/usr/bin/code` | 最终采用 |

需要注意：`/usr/share/code/code` 并非在所有机器上必然异常。另一台机器直接使用它也可以正常工作。因此这里记录的是**本机稳定 workaround / 启动路径**，不要把它理解成 VS Code 的通用缺陷。

正常且最终采用的路径：

```text
Ubuntu Dock / 应用菜单
        ↓
~/.local/share/applications/code.desktop
        ↓
~/.local/bin/code-proxy
        ↓
注入 HTTP/HTTPS/ALL_PROXY
加载 DEEPSEEK_API_KEY（如果存在）
        ↓
/usr/bin/code
        ↓
/usr/share/code/bin/code
        ↓
VS Code 官方 CLI Launcher
        ↓
/usr/share/code/code
        ↓
VS Code / Codex
```

---

## 2. 先确认 VS Code 实际安装情况

不要因为 GNOME 中出现两个 VS Code 图标就直接判断安装了两套 VS Code。

检查：

```bash
type -a code
which -a code

dpkg -l | grep -Ei '(^ii.*code|visual.*studio)' || true
snap list 2>/dev/null | grep -Ei 'code|visual' || true
flatpak list 2>/dev/null | grep -Ei 'code|visual' || true

ls -l /usr/bin/code /usr/local/bin/code /snap/bin/code ~/.local/bin/code 2>/dev/null
ls -ld /usr/share/code /opt/visual-studio-code /snap/code ~/.vscode 2>/dev/null
```

本机最终确认只有一套 DEB 版 VS Code：

```text
/usr/bin/code -> /usr/share/code/bin/code
/usr/share/code/
~/.vscode/
```

GNOME 中曾出现两个图标，是因为同时存在两个 Desktop Entry：

```text
/usr/share/applications/com.microsoft.VSCode.desktop
~/.local/share/applications/code.desktop
```

它们不是两套 VS Code。

---

## 3. 创建统一启动脚本 `code-proxy`

以下代理端口以本机的 `127.0.0.1:7890` 为例。

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/code-proxy <<'EOF'
#!/bin/sh

# ============================================================
# VS Code / OpenAI proxy
# ============================================================
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY="http://127.0.0.1:7890"
export ALL_PROXY="socks://127.0.0.1:7890"

export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
export all_proxy="$ALL_PROXY"

# ============================================================
# DeepSeek API Key
# 文件不存在时不会影响 GPT/OpenAI 模式
# ============================================================
[ -f "$HOME/.config/codex/deepseek.env" ] && \
    . "$HOME/.config/codex/deepseek.env"

# ============================================================
# IMPORTANT:
# 使用 VS Code 官方 CLI launcher。
# 不直接把桌面入口指向 /usr/share/code/code。
# ============================================================
exec /usr/bin/code "$@"
EOF

chmod +x ~/.local/bin/code-proxy
```

检查：

```bash
sed -n '1,120p' ~/.local/bin/code-proxy
```

---

## 4. 创建用户级 VS Code Desktop Entry

新版 VS Code 的系统 Desktop Entry 通常是：

```text
/usr/share/applications/com.microsoft.VSCode.desktop
```

复制一份到用户目录：

```bash
mkdir -p ~/.local/share/applications

cp /usr/share/applications/com.microsoft.VSCode.desktop \
   ~/.local/share/applications/code.desktop
```

修改所有 `Exec=`，统一走 `code-proxy`：

```bash
python3 - <<'PY'
from pathlib import Path

p = Path.home() / ".local/share/applications/code.desktop"

lines = []
for line in p.read_text().splitlines():
    if line.startswith("Exec="):
        if "--new-window" in line:
            line = f"Exec={Path.home()}/.local/bin/code-proxy --new-window %F"
        else:
            line = f"Exec={Path.home()}/.local/bin/code-proxy %F"
    lines.append(line)

p.write_text("\n".join(lines) + "\n")
PY
```

---

## 5. 修复 Dock 点击后“又生成第二个 VS Code 图标”

这是本次排查中很重要的一步。

新版官方 Desktop Entry 使用：

```ini
StartupWMClass=com.microsoft.VSCode
```

如果自定义 Desktop Entry 仍写成：

```ini
StartupWMClass=Code
```

GNOME 会把：

```text
收藏夹中的 launcher
```

和：

```text
实际运行的 VS Code 窗口
```

识别成两个应用，因此点击图标后 Dock 会再冒出一个 VS Code 图标。

修复：

```bash
sed -i \
's/^StartupWMClass=Code$/StartupWMClass=com.microsoft.VSCode/' \
~/.local/share/applications/code.desktop
```

为了防止文件中原本没有该字段，也可以检查：

```bash
grep '^StartupWMClass=' ~/.local/share/applications/code.desktop
```

正确结果：

```text
StartupWMClass=com.microsoft.VSCode
```

如果没有输出，手动在 `[Desktop Entry]` 区域加入：

```ini
StartupWMClass=com.microsoft.VSCode
```

---

## 6. 隐藏系统原始 Desktop Entry，避免出现两个 VS Code 应用入口

系统入口：

```text
/usr/share/applications/com.microsoft.VSCode.desktop
```

不要直接删除或修改它，因为 VS Code 升级后可能重新生成。

在用户目录创建同名覆盖文件：

```bash
cat > ~/.local/share/applications/com.microsoft.VSCode.desktop <<'EOF'
[Desktop Entry]
Type=Application
Name=Visual Studio Code
Hidden=true
EOF
```

这样系统入口被用户级配置隐藏，而我们自己的：

```text
~/.local/share/applications/code.desktop
```

继续作为唯一可见入口。

刷新：

```bash
update-desktop-database ~/.local/share/applications/ 2>/dev/null || true
```

---

## 7. 检查最终 Desktop Entry

```bash
grep -E '^(Name|Exec|Icon|StartupWMClass)=' \
    ~/.local/share/applications/code.desktop
```

核心内容应类似：

```text
Name=Visual Studio Code
Exec=/home/<用户名>/.local/bin/code-proxy %F
Icon=vscode
StartupWMClass=com.microsoft.VSCode

Name=New Empty Window
Exec=/home/<用户名>/.local/bin/code-proxy --new-window %F
Icon=vscode
```

同时确认系统入口已经被隐藏：

```bash
cat ~/.local/share/applications/com.microsoft.VSCode.desktop
```

应为：

```ini
[Desktop Entry]
Type=Application
Name=Visual Studio Code
Hidden=true
```

---

## 8. 完全重启 VS Code

```bash
pkill -f '/usr/share/code/code'
sleep 2

pgrep -af '/usr/share/code/code' || \
    echo "VS Code 已完全退出"
```

然后直接从 Ubuntu Dock / 应用菜单启动 Visual Studio Code。

不要再使用旧的系统收藏夹入口。如果 GNOME 仍缓存了旧图标：

1. 从收藏夹移除旧 VS Code。
2. 从“显示应用程序”找到当前唯一的 Visual Studio Code。
3. 重新添加到收藏夹。

---

## 9. 验证桌面启动的进程环境

保持通过桌面启动的 VS Code 正在运行：

```bash
PID=$(pgrep -o -f '/usr/share/code/code')

echo "PID=$PID"

sudo tr '\0' '\n' < /proc/$PID/environ | \
grep -E '^(HTTP_PROXY|HTTPS_PROXY|ALL_PROXY|DEEPSEEK_API_KEY)=' | \
sed 's/^DEEPSEEK_API_KEY=.*/DEEPSEEK_API_KEY=<LOADED>/'
```

DeepSeek Key 已配置时应看到：

```text
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
ALL_PROXY=socks://127.0.0.1:7890
DEEPSEEK_API_KEY=<LOADED>
```

这个检查命令不会打印真实 Key。

---

## 10. Codex GUI 异常时查看日志

如果 Codex Sidebar 打不开：

```bash
LOG=$(find ~/.config/Code/logs \
      -type f \
      -path '*/openai.chatgpt/Codex.log' \
      -printf '%T@ %p\n' 2>/dev/null | \
      sort -nr | head -1 | cut -d' ' -f2-)

echo "$LOG"
tail -120 "$LOG"
```

本机曾出现：

```text
Activating Codex extension
Spawning codex app-server
Initialize received id=1
...
CodexWebviewProvider:
Webview renderer did not become ready
reason=renderer_ready_timeout
```

这种情况下要区分：

```text
Codex app-server / CLI 后端
```

和：

```text
VS Code WebView renderer 前端
```

如果已经出现 `Initialize received`，说明不能简单判断为 DeepSeek API 或 Codex CLI 启动失败。

---

## 11. 最终启动架构

```text
                    Ubuntu GNOME
                         │
                         ▼
          Visual Studio Code 唯一图标
                         │
                         ▼
 ~/.local/share/applications/code.desktop
                         │
                         ▼
             ~/.local/bin/code-proxy
                         │
          ┌──────────────┴──────────────┐
          │                             │
          ▼                             ▼
 HTTP/HTTPS/ALL_PROXY          DEEPSEEK_API_KEY
                                      │
          └──────────────┬─────────────┘
                         ▼
                    /usr/bin/code
                         │
                         ▼
              /usr/share/code/bin/code
                         │
                         ▼
                    VS Code
                         │
                         ▼
                 Codex Extension
```

---

## 12. 建议备份

```bash
mkdir -p ~/vscode-codex-backup

cp -a ~/.local/bin/code-proxy \
      ~/vscode-codex-backup/

cp -a ~/.local/share/applications/code.desktop \
      ~/vscode-codex-backup/

cp -a ~/.local/share/applications/com.microsoft.VSCode.desktop \
      ~/vscode-codex-backup/
```

注意：

```text
~/.config/codex/deepseek.env
```

包含 API Key，不建议放入普通 Git 仓库或随意复制到共享备份中。

---

## 13. 最简恢复流程

重新安装 VS Code 后，如果需要恢复本方案，核心就是：

```text
1. 恢复 ~/.local/bin/code-proxy
2. 恢复 ~/.local/share/applications/code.desktop
3. 确认 StartupWMClass=com.microsoft.VSCode
4. 创建用户级 com.microsoft.VSCode.desktop，Hidden=true
5. update-desktop-database
6. 完全退出 VS Code
7. 从唯一的用户级图标重新启动
```

最终必须保持：

```text
桌面图标
    ↓
code.desktop
    ↓
code-proxy
    ↓
/usr/bin/code
    ↓
VS Code / Codex
```
