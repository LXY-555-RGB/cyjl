# RAG 成语接龙 + 本地 Qwen AI 助手

### 一.功能亮点

纯本地 RAG 检索：AI 从本地成语文本库匹配，不联网、不瞎编
随机接龙：每次从符合条件的成语中随机抽取，不再固定答案
严格规则校验：校验成语合法性、接龙首尾字规则
内置 AI 对话：可单独打开窗口与本地 Qwen 大模型自由聊天
GUI 图形界面：Tkinter 可视化，操作简单、体验流畅
游戏状态提示：实时显示当前接龙、需接字、胜负状态

<img src="运行结果\结果.jpg">

### 二.代码呈现

```python
import os
import random
import tkinter as tk
from tkinter import ttk, scrolledtext, messagebox
import requests
import threading  # 用于后台自动接龙

# ===================== 项目配置 =====================
FILE_PATH = r"F:\tutorial\damoxing\cydq\成语大全.txt"

# Ollama 模型配置
OLLAMA_API = "http://localhost:11434/api/generate"
MODEL_NAME = "qwen2.5:0.5b"
# ====================================================


# ===================== RAG 成语检索模块 =====================
def load_idioms():
    """加载本地成语库（RAG知识库）"""
    if not os.path.exists(FILE_PATH):
        messagebox.showerror("错误", f"未找到成语文件：\n{FILE_PATH}")
        return []
    with open(FILE_PATH, 'r', encoding='utf-8') as f:
        idioms = [line.strip() for line in f if line.strip()]
    return idioms


def find_next_idiom(last_char, idioms):
    """RAG检索：找出所有以 last_char 开头的成语，随机返回一个"""
    candidates = [idiom for idiom in idioms if idiom.startswith(last_char)]
    if not candidates:
        return None
    return random.choice(candidates)


# ===================== AI 对话模块 =====================
def chat_ollama(prompt):
    """调用本地 qwen2.5 模型"""
    payload = {
        "model": MODEL_NAME,
        "prompt": prompt,
        "stream": False
    }
    try:
        resp = requests.post(OLLAMA_API, json=payload, timeout=60)
        resp.raise_for_status()
        return resp.json().get("response", "AI 暂无回复")
    except Exception as e:
        return f"AI调用失败：请检查Ollama是否运行\n错误信息：{str(e)}"


# ===================== GUI 界面 =====================
class IdiomRAGApp:
    def __init__(self, root):
        self.root = root
        self.root.title("成语接龙 + Qwen2.5AI助手")
        self.root.geometry("700x650")
        self.root.resizable(False, False)

        # 加载成语库
        self.idiom_list = load_idioms()
        self.game_over = False
        self.last_ai_idiom = None  # 记录AI上一轮的成语
        self.ai_first = False      # AI是否先手
        self.ai_auto_playing = False  # AI自玩状态

        # ========== 界面样式 ==========
        style = ttk.Style()
        style.configure(".", font=("微软雅黑", 11))

        # 标题
        ttk.Label(root, text="RAG 成语接龙游戏", font=("微软雅黑", 16, "bold")).pack(pady=10)

        # 先手选择区域
        first_frame = ttk.Frame(root)
        first_frame.pack(pady=5)
        self.first_var = tk.StringVar(value="user")
        ttk.Radiobutton(first_frame, text="用户先出", variable=self.first_var, value="user").grid(row=0, column=0, padx=10)
        ttk.Radiobutton(first_frame, text="AI先出", variable=self.first_var, value="ai").grid(row=0, column=1, padx=10)

        # AI自玩区域
        auto_frame = ttk.Frame(root)
        auto_frame.pack(pady=3)
        self.auto_btn = ttk.Button(auto_frame, text="AI自己玩", command=self.toggle_ai_auto_play)
        self.auto_btn.grid(row=0, column=0, padx=5)
        ttk.Label(auto_frame, text="（自动接龙演示）").grid(row=0, column=1)

        # 状态栏
        self.status_label = ttk.Label(root, text=f"已加载成语：{len(self.idiom_list)} 个", foreground="green")
        self.status_label.pack()

        # 历史记录框
        ttk.Label(root, text="接龙历史").pack()
        self.history = scrolledtext.ScrolledText(root, width=80, height=12, state=tk.DISABLED)
        self.history.pack(pady=5)

        # 输入区域
        input_frame = ttk.Frame(root)
        input_frame.pack(pady=8)
        ttk.Label(input_frame, text="你的成语：").grid(row=0, column=0)
        self.entry = ttk.Entry(input_frame, width=20, font=("微软雅黑", 12))
        self.entry.grid(row=0, column=1, padx=5)
        self.entry.focus()

        # 按钮
        btn_frame = ttk.Frame(root)
        btn_frame.pack(pady=5)
        ttk.Button(btn_frame, text="开始接龙", command=self.play).grid(row=0, column=0, padx=5)
        ttk.Button(btn_frame, text="重新开始", command=self.reset).grid(row=0, column=1, padx=5)
        ttk.Button(btn_frame, text="AI对话", command=self.open_ai_window).grid(row=0, column=2, padx=5)

        # 回车绑定
        self.entry.bind("<Return>", lambda e: self.play())

    def add_log(self, text):
        """添加日志到历史框"""
        self.history.config(state=tk.NORMAL)
        self.history.insert(tk.END, text + "\n")
        self.history.see(tk.END)
        self.history.config(state=tk.DISABLED)

    def play(self):
        """接龙主逻辑"""
        if self.game_over:
            messagebox.showinfo("提示", "游戏已结束，请重新开始！")
            return

        # 第一次游戏：判断谁先手
        if self.last_ai_idiom is None and not self.ai_first:
            self.ai_first = (self.first_var.get() == "ai")
            if self.ai_first:
                # AI先手：随机出一个成语
                ai_idiom = random.choice(self.idiom_list)
                self.last_ai_idiom = ai_idiom
                self.add_log(f"AI先手：{ai_idiom}")
                self.status_label.config(text=f"AI：{ai_idiom}（请接：{ai_idiom[-1]}）", foreground="green")
                return

        user_idiom = self.entry.get().strip()
        self.entry.delete(0, tk.END)

        if not user_idiom:
            messagebox.showwarning("提示", "请输入成语！")
            return

        # 1. 检查是否在库中
        if user_idiom not in self.idiom_list:
            self.add_log(f"【{user_idiom}】不在成语库中 → 游戏结束")
            self.status_label.config(text="游戏结束", foreground="red")
            self.game_over = True
            return

        # 2. 规则检查：必须接AI上一个成语的尾字
        if self.last_ai_idiom is not None:
            required_start = self.last_ai_idiom[-1]
            if not user_idiom.startswith(required_start):
                self.add_log(f"错误！必须以【{required_start}】开头 → 游戏结束")
                self.status_label.config(text="游戏结束", foreground="red")
                self.game_over = True
                return

        # 3. AI 从成语库随机检索下一个
        user_last_char = user_idiom[-1]
        ai_idiom = find_next_idiom(user_last_char, self.idiom_list)

        if not ai_idiom:
            self.add_log(f"{user_idiom} → AI找不到成语，你赢了！")
            self.status_label.config(text="你赢了！", foreground="blue")
            self.game_over = True
            return

        # 4. 接龙成功
        self.last_ai_idiom = ai_idiom
        self.add_log(f"{user_idiom} → {ai_idiom}")
        self.status_label.config(text=f"AI：{ai_idiom}（请接：{ai_idiom[-1]}）", foreground="green")

    def reset(self):
        """重置游戏"""
        self.game_over = False
        self.last_ai_idiom = None
        self.ai_first = False
        self.ai_auto_playing = False
        self.entry.delete(0, tk.END)
        self.auto_btn.config(text="AI自己玩")
        self.status_label.config(text=f"已加载成语：{len(self.idiom_list)} 个", foreground="green")
        self.add_log("==================== 新游戏 ====================")

    # ===================== AI 自玩核心 =====================
    def toggle_ai_auto_play(self):
        """切换AI自动接龙模式"""
        if self.ai_auto_playing:
            self.ai_auto_playing = False
            self.auto_btn.config(text="AI自己玩")
            self.status_label.config(text="AI自玩已暂停", foreground="orange")
            return

        # 开始自玩
        self.reset()
        self.ai_auto_playing = True
        self.auto_btn.config(text="停止自玩")
        self.status_label.config(text="AI开始自己接龙...", foreground="blue")
        # 开线程避免界面卡死
        threading.Thread(target=self.ai_play_itself, daemon=True).start()

    def ai_play_itself(self):
        import time
        # 第一个成语
        current = random.choice(self.idiom_list)
        self.add_log(f"🤖 AI开局：{current}")

        while self.ai_auto_playing and not self.game_over:
            time.sleep(1.2)

            last_char = current[-1]
            next_idiom = find_next_idiom(last_char, self.idiom_list)

            if not next_idiom:
                self.add_log(f"AI接不下去了，自玩结束")
                self.status_label.config(text="AI自玩结束", foreground="red")
                self.ai_auto_playing = False
                self.auto_btn.config(text="AI自己玩")
                return

            self.add_log(f"{current} → {next_idiom}")
            current = next_idiom

        self.status_label.config(text="AI自玩已停止", foreground="gray")

    # ======================================================

    def open_ai_window(self):
        """打开AI对话窗口"""
        ai_win = tk.Toplevel(self.root)
        ai_win.title("Qwen2.5 本地AI助手")
        ai_win.geometry("600x500")

        # 对话记录
        chat_log = scrolledtext.ScrolledText(ai_win, width=70, height=18)
        chat_log.pack(pady=10)
        chat_log.config(state=tk.DISABLED)

        # 初始化欢迎语
        chat_log.config(state=tk.NORMAL)
        chat_log.insert(tk.END, "AI：你好！我是本地运行的 qwen2.5:0.5b 大模型\n")
        chat_log.config(state=tk.DISABLED)

        # 输入框
        input_frame = ttk.Frame(ai_win)
        input_frame.pack(pady=5)
        ai_entry = ttk.Entry(input_frame, width=40, font=("微软雅黑", 11))
        ai_entry.grid(row=0, column=0, padx=5)
        ai_entry.focus()

        def send_chat():
            prompt = ai_entry.get().strip()
            if not prompt:
                return
            ai_entry.delete(0, tk.END)

            # 显示用户消息
            chat_log.config(state=tk.NORMAL)
            chat_log.insert(tk.END, f"\n 你：{prompt}")
            chat_log.config(state=tk.DISABLED)

            # AI回复
            reply = chat_ollama(prompt)
            chat_log.config(state=tk.NORMAL)
            chat_log.insert(tk.END, f"\n AI：{reply}\n")
            chat_log.see(tk.END)
            chat_log.config(state=tk.DISABLED)

        ttk.Button(input_frame, text="发送", command=send_chat).grid(row=0, column=1)
        ai_entry.bind("<Return>", lambda e: send_chat())


# ===================== 运行 =====================
if __name__ == "__main__":
    root = tk.Tk()
    app = IdiomRAGApp(root)
    root.mainloop()
```

1.准备 成语大全.txt，一行一个成语
2.在代码中修改成语文件路径 FILE_PATH
3.启动 Ollama 服务
4.运行程序

```bash
python main.py
```

#### 游戏规则

玩家输入成语，必须在本地库中存在
必须以上一轮 AI 成语的最后一个字开头
AI 检索所有符合条件成语并随机回复
无匹配成语时，AI 失败，玩家获胜