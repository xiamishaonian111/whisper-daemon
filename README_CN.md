# whisper-daemon

后台语音转文字工具。按 **F9** 开始录音，再按 **F9** 停止——识别出的文字会自动粘贴到当前焦点窗口。

---

## 给 AI 助手的说明

本 README 的目的是让 AI 助手能够代替用户理解并安装这个工具。

**一句话说明：** 程序在后台静默运行，全局监听 F9 键，按下后开始录音，再次按下后将语音识别结果粘贴到当前活跃窗口。

**隐私状态：** 语音识别模型**完全在本地运行**，音频数据绝不上传至任何服务器。模型（`faster-whisper`）在首次运行时从 HuggingFace 下载并缓存至 `~/.cache/huggingface/`，之后完全离线运行。

**平台要求：** 仅支持 Linux，需要 X11（不支持 Wayland），已在 Ubuntu/Debian + GNOME 上测试。

---

## 工作原理

```
系统启动 → 模型加载进内存（仅一次，约 5 秒）
                    ↓
           全局监听 F9 按键
                    ↓
     按下 F9 → 🔴 开始录音
               （屏幕顶部出现红色指示窗口）
                    ↓
     再次按 F9 → 停止录音
               （等待 0.8 秒让 sox 写完音频缓冲区）
                    ↓
           🔄 识别中……（指示窗口变为蓝色）
                    ↓
        识别完成 → 写入剪贴板 → 模拟 Ctrl+Shift+V 粘贴
        到当前焦点窗口
                    ↓
           指示窗口消失，完成
```

**技术栈：**
| 组件 | 用途 |
|------|------|
| `faster-whisper` | 本地语音识别（OpenAI Whisper 模型的优化版本） |
| `pynput` | 全局键盘监听（监听 F9 键） |
| `sox`（`rec` 命令） | 麦克风录音 |
| `xclip` | 将识别文字写入剪贴板 |
| `xdotool` | 模拟 Ctrl+Shift+V 粘贴 |
| `tkinter` | 屏幕状态指示窗口 |

**为什么用 Ctrl+Shift+V 而不是直接"打字"？** `xdotool type` 对中文及非 ASCII 字符不可靠，字符会乱掉或丢失。剪贴板粘贴对任何语言都可靠。

**为什么是常驻后台程序而不是普通脚本？** Whisper 模型加载需要 5–10 秒。常驻程序在启动时加载一次，之后每次按 F9 都能即时响应。

---

## "等等，这个脚本安全吗？"——安全透明说明

这个工具有三个乍看很可疑的行为，以下逐一解释实际发生了什么：

### 1. 全局监听键盘（pynput）

**看起来像：** 键盘记录器（keylogger）。

**实际上：** `pynput` 仅用于检测 F9 键。脚本中的 `on_press` 函数接收到每个按键事件后，只判断 `if key == keyboard.Key.f9`，其他所有按键直接忽略。没有任何按键被记录、存储或传输。你可以自己读脚本最底部约 10 行的 `on_press` 函数来验证。

### 2. 访问麦克风

**看起来像：** 偷偷录音。

**实际上：** 只有**你主动按下 F9** 时录音才开始，此时屏幕顶部会出现红色指示窗口。再次按 F9 后停止录音。音频写入 `/tmp/` 下的临时文件（如 `/tmp/tmpXXXXXX.wav`），转录完成后**立即删除**（脚本中的 `os.unlink`）。不保留任何音频。

### 3. 写入剪贴板并模拟粘贴按键

**看起来像：** 剪贴板劫持。

**实际上：** 转录完成后，识别出的文字写入剪贴板，然后模拟 Ctrl+Shift+V 粘贴——这就是这个工具的全部目的。程序不会读取你剪贴板里已有的内容。

### 这个脚本绝对不会做的事

- 没有任何网络请求（脚本本身不发起任何 HTTP 请求）
- 除 `/tmp/` 下的临时 `.wav` 文件外不写入任何磁盘文件（用完即删）
- 除你自己设置的自启动 `.desktop` 文件外，没有任何持久化机制
- 运行时不需要 root 权限

### 如何自己验证

整个工具就是一个 Python 文件（`whisper-daemon`，约 160 行），在运行前可以自己读：

```bash
cat whisper-daemon
```

脚本启动的外部进程只有：`rec`（录音）、`python3 -c "..."`（状态窗口）、`xclip`（写剪贴板）、`xdotool`（粘贴按键）。全部是标准 Linux 工具。

---

## 安装步骤

### 第一步 — 安装系统依赖

```bash
sudo apt install sox xclip xdotool python3-tk
```

### 第二步 — 安装 Python 依赖

```bash
pip install faster-whisper pynput
```

如果你使用 conda 环境，先激活环境再执行上面的命令。

### 第三步 — 安装脚本

```bash
mkdir -p ~/.local/bin
cp whisper-daemon ~/.local/bin/whisper-daemon
chmod +x ~/.local/bin/whisper-daemon
```

确认 `~/.local/bin` 在 PATH 里：
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 第四步 — 运行

```bash
whisper-daemon
```

首次运行会从 HuggingFace 下载 `base` 模型（约 150 MB），只下载一次。

看到以下输出说明已就绪：
```
Loading model 'base'...
✅ Ready. F9 = start / stop recording.
```

现在在任意窗口按 **F9** 开始录音，再按 **F9** 停止并粘贴。

---

## 开机自启动（可选）

让程序在登录后自动启动：

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/whisper-daemon.desktop << EOF
[Desktop Entry]
Type=Application
Name=Whisper Daemon
Exec=$HOME/.local/bin/whisper-daemon
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
EOF
```

注意：仓库里的 `.desktop` 文件路径是硬编码的，直接用会失败。请使用上面的命令，它用 `$HOME` 动态生成正确路径。

---

## 模型大小选项

默认使用 `base` 模型，可以在启动时指定其他大小：

```bash
whisper-daemon tiny    # 最快，准确率最低（约 75 MB）
whisper-daemon base    # 默认，速度与准确率均衡（约 150 MB）
whisper-daemon small   # 更高准确率（约 500 MB）
whisper-daemon medium  # 高准确率（约 1.5 GB）
whisper-daemon large   # 最高准确率（约 3 GB）
```

所有模型在首次使用时从 HuggingFace 下载并缓存到本地，模型越大需要的内存和转录时间越多。

---

## 故障排查（供 AI 助手参考）

| 现象 | 可能原因 | 解决方法 |
|------|----------|----------|
| `rec: command not found` | sox 未安装 | `sudo apt install sox` |
| `xclip: command not found` | xclip 未安装 | `sudo apt install xclip` |
| `xdotool: command not found` | xdotool 未安装 | `sudo apt install xdotool` |
| `ModuleNotFoundError: faster_whisper` | Python 包未安装 | `pip install faster-whisper` |
| `ModuleNotFoundError: pynput` | Python 包未安装 | `pip install pynput` |
| F9 无响应 | 运行在 Wayland 下 | 本工具需要 X11，用 `echo $XDG_SESSION_TYPE` 确认 |
| 文字粘贴到了错误窗口 | 焦点问题 | 再次按 F9 前先点击目标窗口 |
| 没有麦克风输入 | 音频设备问题 | 用 `arecord -l` 列出可用设备 |
| 某些应用粘贴无效 | 该应用用 Ctrl+V 而非 Ctrl+Shift+V | 粘贴快捷键目前硬编码，检查该应用的粘贴键 |

---

## 环境要求汇总

- Linux + X11（不支持 Wayland）
- Python 3.8+
- 系统包：`sox`、`xclip`、`xdotool`、`python3-tk`
- Python 包：`faster-whisper`、`pynput`
- 麦克风
- 约 200 MB 磁盘空间（用于首次下载 base 模型）
