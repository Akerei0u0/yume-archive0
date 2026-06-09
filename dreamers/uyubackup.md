```
from DrissionPage import ChromiumPage, ChromiumOptions
import sqlite3
import re
import time

# ================= 配置区域 =================
BASE_URL_PREFIX = "https://note.ms/yume"
START_ID = "1"
DB_NAME = "yumenote_backup.db"
DELAY = 1.5
# ===========================================

def init_db():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS pages
                 (id TEXT PRIMARY KEY, content TEXT, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
    c.execute('''CREATE TABLE IF NOT EXISTS links
                 (source_id TEXT, target_id TEXT,
                  PRIMARY KEY (source_id, target_id))''')
    conn.commit()
    return conn

def is_scraped(conn, page_id):
    c = conn.cursor()
    c.execute("SELECT 1 FROM pages WHERE id = ?", (page_id,))
    return c.fetchone() is not None

def save_page(conn, page_id, content, links):
    c = conn.cursor()
    try:
        c.execute("INSERT OR IGNORE INTO pages (id, content) VALUES (?, ?)", (page_id, content))
        for link in links:
            c.execute("INSERT OR IGNORE INTO links (source_id, target_id) VALUES (?, ?)", (page_id, link))
        conn.commit()
    except Exception as e:
        print(f"Error saving {page_id}: {e}")

def main():
    conn = init_db()

    print("正在初始化本地浏览器...")
    co = ChromiumOptions()
    co.set_paths(browser_path='/usr/bin/chromium')
    co.headless(True)  # 开启无头模式，不显示界面
    co.set_argument('--no-sandbox')
    co.set_argument('--disable-gpu')

    # 启动浏览器
    page = ChromiumPage(co)

    queue = [START_ID]
    print(f"浏览器启动成功！开始抓取...")

    try:
        while queue:
            current_id = queue.pop(0)

            # 1. 检查是否已经抓取过
            if is_scraped(conn, current_id):
                continue

            url = f"{BASE_URL_PREFIX}{current_id}"
            print(f"正在访问: {url}")

            try:
                # 2. 访问页面
                page.get(url)

                # 等待元素
                textarea = page.ele('css:textarea.content', timeout=15)

                if not textarea:
                    print("  -> [超时/失败] 未找到内容区域。跳过此页。")
                    continue

                content = textarea.text

                # ================= 核心修复区域 =================
                # 3.1 正则提取原始链接
                raw_matches = re.findall(r'跳转([a-zA-Z0-9]+)', content)

                # 3.2 数据清洗 (把 yume21 变成 21)
                final_clean_list = []
                for link in raw_matches:
                    clean_link = link
                    # 如果是以 yume 开头，切掉前4位
                    if clean_link.startswith('yume'):
                        clean_link = clean_link[4:]

                    # 确保不为空
                    if clean_link:
                        final_clean_list.append(clean_link)

                # 3.3 最终去重 (一定要用清洗后的 list 做去重)
                unique_links = list(set(final_clean_list))
                # ===============================================

                # 4. 保存 (传入 unique_links)
                save_page(conn, current_id, content, unique_links)

                # 打印日志验证
                print(f"  -> [成功] 原始: {raw_matches} -> 最终路径: {unique_links}")

                # 5. 加入队列 (传入 unique_links)
                for nid in unique_links:
                    # 防止入队空值
                    if nid and not is_scraped(conn, nid):
                        queue.append(nid)

                time.sleep(DELAY)

            except Exception as e:
                print(f"  -> 发生错误: {e}")

    except KeyboardInterrupt:
        print("\n用户中断。")
    finally:
        conn.close()

if __name__ == "__main__":
    main()
```