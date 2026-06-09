```
from textual.app import App, ComposeResult
from textual.widgets import Header, Footer, Input, Static
from textual.containers import Container
from DrissionPage import ChromiumPage, ChromiumOptions
import re

# ================= 配置 =================
BASE_URL_PREFIX = "https://note.ms/yume"
START_NODE = "0"
# =======================================

class WebFetcher:
    """后端交互类"""
    def __init__(self):
        co = ChromiumOptions()
        # 如果需要调试，改为 False 可见窗口
        co.headless(True)
        # co.set_paths(browser_path='/usr/bin/chromium')
        co.set_argument('--no-sandbox')
        co.set_argument('--disable-gpu')
        co.set_argument('--blink-settings=imagesEnabled=false')

        try:
            self.page = ChromiumPage(co)
        except Exception as e:
            print(f"浏览器启动失败: {e}")
            raise e

    def fetch(self, page_id):
        url = f"{BASE_URL_PREFIX}{page_id}"
        try:
            self.page.get(url)

            textarea = self.page.ele('css:textarea.content', timeout=5)

            if not textarea:
                return None, []

            content = textarea.property('value')

            if content is None:
                content = "" # 防止 value 为空时报错

            raw_matches = re.findall(r'跳转([a-zA-Z0-9]+)', content)
            links = []
            for link in raw_matches:
                clean = link[4:] if link.startswith('yume') else link
                if clean:
                    links.append(clean)

            return content, list(set(links))
        except Exception as e:
            return f"Error: {e}", []

    def close(self):
        self.page.quit()

class GameApp(App):

    TITLE = "Yume Note Client"

    CSS = """
    Screen {
        layout: vertical;
    }
    #game_container {
        height: 1fr;
        padding: 1 2;
        background: $surface;
        /* 允许垂直滚动 */
        overflow-y: auto;
    }
    #status_bar {
        height: auto;
        background: $accent;
        color: $text;
        padding: 0 1;
        text-style: bold;
    }
    Input {
        dock: bottom;
        border: wide $accent;
    }
    """

    def __init__(self):
        super().__init__()
        self.fetcher = None
        self.history_stack = []
        self.current_id = START_NODE

    def compose(self) -> ComposeResult:
        yield Header(show_clock=True)

        with Container(id="game_container"):

            yield Static("正在初始化浏览器...", id="content_view", markup=False)

        yield Static("Initializing...", id="status_bar")

        yield Input(placeholder="输入跳转ID (或 back 回退, q 退出)", id="minibuffer")
        yield Footer()

    def on_mount(self) -> None:
        """初始化"""
        # 使用 call_later 可以让 UI 先画出来，再执行耗时的浏览器启动
        # 避免启动时白屏太久
        self.call_later(self.init_browser)

    def init_browser(self):
        """延迟启动浏览器"""
        self.fetcher = WebFetcher()
        self.load_page(self.current_id)

    def load_page(self, page_id, push_history=True):
        """加载页面"""

        self.title = f"Yume Note - {page_id}"

        # 更新状态栏
        self.query_one("#status_bar").update(f"正在加载 {page_id} ...")

        # 阻塞式抓取
        content, links = self.fetcher.fetch(page_id)

        if content is None:
            self.query_one("#content_view").update(f"404 Not Found\n无法访问页面 {page_id}")
            self.query_one("#status_bar").update("请求失败")
            return

        # 更新历史
        if push_history and self.current_id != page_id:
            self.history_stack.append(self.current_id)
        self.current_id = page_id

        # 渲染内容
        # 使用 Static 组件，直接 update 纯文本即可，换行符会被保留
        self.query_one("#content_view").update(content)

        # 更新链接提示
        if links:
            link_text = " | ".join(links)
            self.query_one("#status_bar").update(f"可用出口: {link_text}")
        else:
            self.query_one("#status_bar").update("没有检测到出口 (Dead End)")

        self.query_one(Input).focus()

    def on_input_submitted(self, event: Input.Submitted) -> None:
        cmd = event.value.strip()
        self.query_one(Input).value = ""

        if not cmd:
            return

        if cmd.lower() in ('q', 'quit', 'exit'):
            if self.fetcher:
                self.fetcher.close()
            self.exit()
            return

        if cmd.lower() == 'back':
            if self.history_stack:
                prev = self.history_stack.pop()
                self.load_page(prev, push_history=False)
            else:
                self.query_one("#status_bar").update("已经是起点了")
            return

        target_id = cmd
        if target_id.startswith("yume"):
            target_id = target_id[4:]

        self.load_page(target_id)

if __name__ == "__main__":
    app = GameApp()
    app.run()
```