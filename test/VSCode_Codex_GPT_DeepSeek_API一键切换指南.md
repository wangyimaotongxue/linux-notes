# VS Code Codex：OpenAI GPT / DeepSeek API 完整配置与一键切换指南

> 目标：在保留 OpenAI / GPT 配置的同时，为 VS Code Codex 接入 DeepSeek V4 Pro，并通过 `codex-switch` 在两套配置之间安全切换。
>
> 本文是在原《VS Code Codex 接入 DeepSeek API 配置指南》基础上的完整扩展版。最终方案不再依赖每次运行 DeepSeek setup 脚本切换模型，而是维护两份独立配置。

---

## 1. 当前架构

VS Code OpenAI ChatGPT/Codex 扩展已经自带 Codex CLI，因此无需为了本方案额外通过 npm、Snap 或 Homebrew 安装 Codex。

查找：

```bash
find ~/.vscode/extensions \
    -type f -name codex 2>/dev/null
```

本机曾验证的路径形式：

```text
~/.vscode/extensions/openai.chatgpt-<版本>-linux-x64/bin/linux-x86_64/codex
```

注意：扩展升级后目录中的版本号会变化，因此不要长期依赖写死的旧版本目录。

可以动态获取：

```bash
CODEX="$(find ~/.vscode/extensions \
    -type f \
    -path '*/openai.chatgpt-*/bin/linux-x86_64/codex' \
    2>/dev/null | sort -V | tail -1)"

echo "$CODEX"
"$CODEX" --version
```

---

## 2. Codex 主配置

Codex 配置目录：

```text
~/.codex/
```

当前活动配置：

```text
~/.codex/config.toml
```

我们的最终设计：

```text
~/.codex/
├── config.toml
├── models.json
└── profiles/
    ├── config.gpt.toml
    └── config.deepseek.toml

~/.config/codex/
└── deepseek.env

~/.local/bin/
└── codex-switch
```

含义：

```text
config.gpt.toml
    = OpenAI / GPT 配置模板

config.deepseek.toml
    = DeepSeek V4 Pro 配置模板

config.toml
    = 当前真正生效的配置

deepseek.env
    = DeepSeek API Key，独立保存

codex-switch
    = 切换 config.toml 的开关
```

---

## 3. 先备份原 OpenAI / GPT 配置

如果当前 `~/.codex/config.toml` 仍然是正常的 OpenAI 配置：

```bash
mkdir -p ~/.codex/profiles

cp -a ~/.codex/config.toml \
      ~/.codex/profiles/config.gpt.toml

chmod 600 ~/.codex/profiles/config.gpt.toml
```

本机 GPT 配置核心形式：

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "high"

[projects."/home/uu/dm_joint_control"]
trust_level = "trusted"
```

项目路径按自己的实际环境保留。

---

## 4. DeepSeek 官方 setup 脚本：只作为初始化参考

原方案使用 DeepSeek setup 脚本：

```bash
curl -fsSL \
  https://cdn.deepseek.com/api-docs/codex-deepseek-setup-en.sh \
  -o /tmp/codex-deepseek-setup.sh
```

并执行：

```bash
bash /tmp/codex-deepseek-setup.sh
```

它可以帮助生成 DeepSeek 相关配置和 `models.json`。

但是最终双模型方案中：

> **不建议每次切模型都重新运行官方 setup 脚本。**

原因是它会直接修改：

```text
~/.codex/config.toml
```

这不利于稳定维护 OpenAI / DeepSeek 两套配置。

最终采用：

```text
官方脚本/已有结果
        ↓
取得 DeepSeek 所需 models.json / provider 配置
        ↓
保存成 config.deepseek.toml
        ↓
以后只用 codex-switch 切换
```

---

## 5. 安全保存 DeepSeek API Key

不要把 API Key 明文写进：

```text
config.toml
Markdown
Git
项目源码
截图
聊天记录
```

创建独立环境文件：

```bash
mkdir -p ~/.config/codex

read -s -p "请输入新的 DeepSeek API Key: " DEEPSEEK_KEY
echo

printf 'export DEEPSEEK_API_KEY=%q\n' "$DEEPSEEK_KEY" \
    > ~/.config/codex/deepseek.env

unset DEEPSEEK_KEY

chmod 600 ~/.config/codex/deepseek.env
```

不要用：

```bash
cat ~/.config/codex/deepseek.env
```

来检查，因为这会把 Key 显示在终端。

安全检查变量名：

```bash
grep -o '^export [A-Z0-9_]*' \
    ~/.config/codex/deepseek.env
```

应看到：

```text
export DEEPSEEK_API_KEY
```

加载：

```bash
source ~/.config/codex/deepseek.env
```

只检查是否存在和长度：

```bash
if [ -n "${DEEPSEEK_API_KEY:-}" ]; then
    echo "DEEPSEEK_API_KEY loaded: YES (${#DEEPSEEK_API_KEY} chars)"
else
    echo "DEEPSEEK_API_KEY loaded: NO"
fi
```

---

## 6. 创建 DeepSeek V4 Pro 配置

创建：

```bash
mkdir -p ~/.codex/profiles

cat > ~/.codex/profiles/config.deepseek.toml <<'EOF'
model = "deepseek-v4-pro"
model_provider = "deepseek"
model_reasoning_effort = "high"
web_search = "disabled"
show_raw_agent_reasoning = true
model_catalog_json = "/home/uu/.codex/models.json"

[model_providers.deepseek]
name = "DeepSeek"
base_url = "https://api.deepseek.com/"
wire_api = "responses"
env_key = "DEEPSEEK_API_KEY"

[projects."/home/uu/dm_joint_control"]
trust_level = "trusted"
EOF

chmod 600 ~/.codex/profiles/config.deepseek.toml
```

注意：

```text
/home/uu/.codex/models.json
/home/uu/dm_joint_control
```

是本机路径。换机器时按实际用户名和项目目录修改。

关键点是：

```toml
env_key = "DEEPSEEK_API_KEY"
```

因此真实 API Key 不需要出现在 TOML 配置中。

---

## 7. 检查 `models.json`

如果 DeepSeek 初始化过程已经生成：

```text
~/.codex/models.json
```

验证 JSON：

```bash
python3 -m json.tool ~/.codex/models.json >/dev/null \
    && echo "models.json OK"
```

应输出：

```text
models.json OK
```

---

## 8. 创建 `codex-switch` 一键切换工具

创建：

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/codex-switch <<'EOF'
#!/bin/bash
set -e

CODEX_HOME="$HOME/.codex"
PROFILE_DIR="$CODEX_HOME/profiles"
ACTIVE="$CODEX_HOME/config.toml"

GPT="$PROFILE_DIR/config.gpt.toml"
DEEPSEEK="$PROFILE_DIR/config.deepseek.toml"

die()
{
    echo "ERROR: $*" >&2
    exit 1
}

install_profile()
{
    local src="$1"
    local name="$2"

    [ -f "$src" ] || die "配置不存在: $src"

    mkdir -p "$CODEX_HOME"

    # 保留切换前的活动配置，便于误操作恢复
    if [ -f "$ACTIVE" ]; then
        cp -a "$ACTIVE" "$CODEX_HOME/config.toml.last"
        chmod 600 "$CODEX_HOME/config.toml.last"
    fi

    cp -a "$src" "$ACTIVE"
    chmod 600 "$ACTIVE"

    echo "Codex profile switched to: $name"
    echo
    echo "Active config:"
    grep -E '^(model|model_provider|model_reasoning_effort)[[:space:]]*=' \
        "$ACTIVE" || true

    echo
    echo "请完全退出并重新启动 VS Code，然后新建 Codex 会话。"
}

status()
{
    echo "========== Codex active config =========="

    if [ ! -f "$ACTIVE" ]; then
        echo "config.toml 不存在"
        exit 1
    fi

    grep -E '^(model|model_provider|model_reasoning_effort|model_catalog_json)[[:space:]]*=' \
        "$ACTIVE" || true

    echo
    if cmp -s "$ACTIVE" "$GPT"; then
        echo "Profile: GPT / OpenAI"
    elif cmp -s "$ACTIVE" "$DEEPSEEK"; then
        echo "Profile: DeepSeek V4 Pro"
    else
        echo "Profile: CUSTOM / UNKNOWN"
    fi

    if grep -q '^model_provider[[:space:]]*=[[:space:]]*"deepseek"' "$ACTIVE"; then
        if [ -f "$HOME/.config/codex/deepseek.env" ]; then
            echo "DeepSeek env file: PRESENT"
        else
            echo "DeepSeek env file: MISSING"
        fi
    fi
}

case "${1:-}" in
    gpt|openai)
        install_profile "$GPT" "GPT / OpenAI"
        ;;

    deepseek|ds)
        install_profile "$DEEPSEEK" "DeepSeek V4 Pro"
        ;;

    status)
        status
        ;;

    *)
        cat <<USAGE
Usage:
  codex-switch gpt
  codex-switch deepseek
  codex-switch status

Aliases:
  codex-switch openai
  codex-switch ds
USAGE
        exit 1
        ;;
esac
EOF

chmod +x ~/.local/bin/codex-switch
```

---

## 9. 确保 `~/.local/bin` 在 PATH

检查：

```bash
echo "$PATH" | tr ':' '\n' | grep -Fx "$HOME/.local/bin"
```

如果没有输出，在 `~/.bashrc` 加入：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

然后：

```bash
source ~/.bashrc
```

检查：

```bash
which codex-switch
```

应类似：

```text
/home/uu/.local/bin/codex-switch
```

---

## 10. GPT / DeepSeek 实际切换命令

### 切换到 OpenAI / GPT

```bash
codex-switch gpt
```

等价别名：

```bash
codex-switch openai
```

然后：

```text
完全退出 VS Code
→ 重新从桌面图标启动
→ 新建 Codex 会话
```

---

### 切换到 DeepSeek V4 Pro

```bash
codex-switch deepseek
```

也可以：

```bash
codex-switch ds
```

然后：

```text
完全退出 VS Code
→ 重新从桌面图标启动
→ 新建 Codex 会话
```

---

### 查看当前配置

```bash
codex-switch status
```

GPT 模式应看到类似：

```text
model = "gpt-5.6-sol"
model_reasoning_effort = "high"

Profile: GPT / OpenAI
```

DeepSeek 模式应看到类似：

```text
model = "deepseek-v4-pro"
model_provider = "deepseek"
model_reasoning_effort = "high"

Profile: DeepSeek V4 Pro
DeepSeek env file: PRESENT
```

---

## 11. 为什么切换后需要重启 VS Code

实际配置关系：

```text
codex-switch gpt
        │
        ▼
config.gpt.toml
        │
        └──── copy ────> ~/.codex/config.toml
                              │
                              ▼
                         VS Code Codex

codex-switch deepseek
        │
        ▼
config.deepseek.toml
        │
        └──── copy ────> ~/.codex/config.toml
                              │
                              ▼
                         VS Code Codex
```

Codex app-server 启动时会读取配置。

因此最稳妥的切换方式是：

```text
执行 codex-switch
        ↓
完全退出 VS Code
        ↓
重新启动 VS Code
        ↓
新建 Codex 会话
```

不要仅依赖已经打开的旧 Codex 会话刷新模型。

---

## 12. 让桌面启动的 VS Code 获得 DeepSeek Key

桌面环境通常不会自动继承终端中执行的：

```bash
source ~/.config/codex/deepseek.env
```

因此在：

```text
~/.local/bin/code-proxy
```

中加入：

```sh
[ -f "$HOME/.config/codex/deepseek.env" ] && \
    . "$HOME/.config/codex/deepseek.env"
```

完整启动关系：

```text
Ubuntu Desktop
      ↓
code.desktop
      ↓
code-proxy
      ├── HTTP_PROXY
      ├── HTTPS_PROXY
      ├── ALL_PROXY
      └── source deepseek.env
                 ↓
          DEEPSEEK_API_KEY
                 ↓
             /usr/bin/code
                 ↓
             VS Code
                 ↓
          Codex app-server
                 ↓
        DeepSeek Responses API
```

即使当前使用 GPT 模式，环境中存在 `DEEPSEEK_API_KEY` 也不会自动把 provider 改成 DeepSeek；真正决定模型/provider 的仍然是活动 `~/.codex/config.toml`。

---

## 13. 验证 DeepSeek CLI 路由

先加载 Key：

```bash
source ~/.config/codex/deepseek.env
```

动态找到 VS Code 内置 Codex：

```bash
CODEX="$(find ~/.vscode/extensions \
    -type f \
    -path '*/openai.chatgpt-*/bin/linux-x86_64/codex' \
    2>/dev/null | sort -V | tail -1)"

"$CODEX" --version
```

切换：

```bash
codex-switch deepseek
```

真实请求测试：

```bash
"$CODEX" exec \
    --strict-config \
    --skip-git-repo-check \
    "只回答一句话：DeepSeek V4 Pro through Codex is working."
```

成功时启动信息应明确显示：

```text
model: deepseek-v4-pro
provider: deepseek
```

并得到模型回复。

相比询问模型：

```text
“你是什么模型？”
```

Codex CLI 启动信息中的：

```text
model:
provider:
```

更适合作为实际路由判断依据。模型在回答正文中自报身份并不可靠。

---

## 14. 验证 GUI 使用 DeepSeek

```bash
codex-switch deepseek
```

完全退出：

```bash
pkill -f '/usr/share/code/code'
sleep 2
```

然后从已经修复好的 Ubuntu VS Code 图标重新启动。

进入 Codex 后：

1. 新建会话。
2. 检查模型选择器是否显示 DeepSeek V4 Pro。
3. 发一个很小的测试请求。
4. 如需严格确认，结合 DeepSeek API 后台用量记录或 Codex 日志判断。

不要仅通过模型正文中的：

```text
“我是 GPT-5”
```

之类自我描述判断真实 provider。

---

## 15. 验证 GPT 恢复

```bash
codex-switch gpt
```

检查：

```bash
codex-switch status
```

然后：

```bash
pkill -f '/usr/share/code/code'
sleep 2
```

重新从桌面启动 VS Code。

活动配置应该恢复：

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "high"
```

并且不再包含：

```toml
model_provider = "deepseek"
```

---

## 16. 配置备份

建议备份：

```bash
mkdir -p ~/vscode-codex-backup

cp -a ~/.codex/profiles \
      ~/vscode-codex-backup/

cp -a ~/.codex/models.json \
      ~/vscode-codex-backup/ \
      2>/dev/null || true

cp -a ~/.local/bin/codex-switch \
      ~/vscode-codex-backup/
```

如果还要备份桌面启动修复：

```bash
cp -a ~/.local/bin/code-proxy \
      ~/vscode-codex-backup/

cp -a ~/.local/share/applications/code.desktop \
      ~/vscode-codex-backup/

cp -a ~/.local/share/applications/com.microsoft.VSCode.desktop \
      ~/vscode-codex-backup/
```

### API Key 单独处理

```text
~/.config/codex/deepseek.env
```

属于敏感文件。

不要：

```text
上传 GitHub
发送到聊天
放进公开 Markdown
随普通配置压缩包公开共享
```

如果确实要离线备份，应使用自己的安全密钥管理或加密备份方式。

---

## 17. 最终日常使用

### 使用 GPT

```bash
codex-switch gpt
```

然后重启 VS Code。

### 使用 DeepSeek V4 Pro

```bash
codex-switch deepseek
```

然后重启 VS Code。

### 查看当前状态

```bash
codex-switch status
```

日常不需要：

```text
重新下载 setup 脚本
重新输入 API Key
重新安装 Codex
重新修改 VS Code Desktop Entry
```

---

## 18. 最终完整架构

```text
                         codex-switch
                              │
              ┌───────────────┴────────────────┐
              │                                │
              ▼                                ▼
     config.gpt.toml                 config.deepseek.toml
              │                                │
              └──────────────┬─────────────────┘
                             │
                             ▼
                    ~/.codex/config.toml
                             │
                             │
Ubuntu Dock                  │
    │                        │
    ▼                        │
code.desktop                 │
    │                        │
    ▼                        │
code-proxy                   │
    ├── Proxy                │
    ├── DEEPSEEK_API_KEY     │
    │                        │
    ▼                        ▼
 /usr/bin/code ──────────> VS Code
                             │
                             ▼
                       Codex Extension
                             │
                  ┌──────────┴──────────┐
                  │                     │
            GPT / OpenAI        DeepSeek V4 Pro
```

---

## 19. 故障恢复

### 切错配置

`codex-switch` 每次切换前会额外保存：

```text
~/.codex/config.toml.last
```

需要手动恢复时：

```bash
cp -a ~/.codex/config.toml.last \
      ~/.codex/config.toml
```

### DeepSeek Key 不存在

```bash
test -f ~/.config/codex/deepseek.env \
    && echo "DeepSeek env OK" \
    || echo "DeepSeek env MISSING"
```

### 当前到底是什么配置

```bash
codex-switch status
```

### Codex CLI 在哪里

```bash
find ~/.vscode/extensions \
    -type f \
    -path '*/openai.chatgpt-*/bin/linux-x86_64/codex' \
    2>/dev/null
```

---

## 20. 安全注意事项

如果 API Key 曾经完整出现在：

```text
聊天记录
终端截图
公开日志
Git commit
公开文档
```

应将该 Key 视为已经暴露：

```text
撤销旧 Key
    ↓
创建新 Key
    ↓
重新写入 ~/.config/codex/deepseek.env
    ↓
chmod 600
```

不要把真实 Key 写入本文档。
