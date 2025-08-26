## 背景
需要模拟一个网站上用户的操作，并且希望记录下来用户的点击步骤和html、css、js、api等


## 调研

playwright 存在 [codegen](https://playwright.dev/python/docs/codegen) 功能和 [trace viewer](https://playwright.dev/python/docs/trace-viewer-intro) 功能
> codegen功能：类似屏幕录制功能，会按照你在屏幕上的动作，自动转为可执行的python代码

> trace viewer功能：Playwright Trace Viewer 是一个 GUI 工具，可让您探索测试的记录的 Playwright 跟踪，这意味着您可以前后移动测试的每个操作，并直观地查看每个操作期间发生的情况。

> 这里是的介绍： https://playwright.dev/python/docs/trace-viewer#introduction 


## 思路

无论是  [codegen](https://playwright.dev/python/docs/codegen)  还是 [trace viewer](https://playwright.dev/python/docs/trace-viewer-intro) 他们都有两种调用方式
1. 命令行
2. 代码调用

这里主要介绍用代码的方式实现结合



首先官方没有将两个合并的步骤，但是 [codegen](https://playwright.dev/python/docs/codegen) 支持自定义功能。[异步代码](https://playwright.dev/python/docs/codegen#record-using-custom-setup)如下

```python
import asyncio
from playwright.async_api import async_playwright

async def main():
    async with async_playwright() as p:
        # Make sure to run headed.
        browser = await p.chromium.launch(headless=False)

        # Setup context however you like.
        context = await browser.new_context() # Pass any options
        await context.route('**/*', lambda route: route.continue_())

        # Pause the page, and start recording manually.
        page = await context.new_page()
        await page.pause()

asyncio.run(main())
```

和AI对话后得知可以同时进行，只是不能用命令行的形式操作。 整体代码如下

```python

from playwright.sync_api import sync_playwright

def run_codegen_with_tracing():
    """
    启动一个类似 Codegen 的会话，并同时记录 Trace。
    """
    with sync_playwright() as p:
        # 你可以指定浏览器，例如 p.firefox 或 p.webkit
        browser = p.chromium.launch(headless=False)
        context = browser.new_context()

        # --- 核心步骤 1: 在所有操作开始前，启动 Tracing ---
        print("🚀 Tracing 已启动... 将在会话结束后保存。")
        context.tracing.start(
            name="codegen-trace",  # Trace 文件名的前缀
            screenshots=True,     # 捕获截图
            snapshots=True,       # 捕获 DOM 快照 (非常重要!)
            sources=True          # 包含源代码
        )

        # 创建一个新页面作为起点
        page = context.new_page()

        try:
            # --- 核心步骤 2: 启动 Playwright Inspector (Codegen 的核心) ---
            print("▶️ 启动 Playwright Inspector (Codegen)...")
            print("👉 请在弹出的浏览器窗口中进行操作，Inspector 会为你生成代码。")
            print("🛑 操作完成后，请直接关闭【浏览器窗口】或【Inspector 窗口】来结束录制。")

            # page.pause() 会暂停脚本，并打开 Inspector 工具。
            # 这就是 codegen 的魔法所在！
            page.pause()
            
            # 当用户关闭浏览器或 Inspector 时，pause() 会结束，代码会继续向下执行。

        finally:
            # --- 核心步骤 3: 在会话结束后，停止并保存 Trace 文件 ---
            print("\n💾 会话结束，正在保存 Trace 文件...")
            trace_path = "my_codegen_session.zip"
            context.tracing.stop(path=trace_path)

            print(f"✅ Trace 文件已成功保存到: {trace_path}")
            print(f"👉 你可以使用命令 'playwright show-trace {trace_path}' 来查看它。")

            # 清理资源
            context.close()
            browser.close()


if __name__ == "__main__":
    run_codegen_with_tracing()

```



## 补充

playwright官网没有提到 trace 的代码功能。具体的代码参考：https://github.com/microsoft/playwright-python/blob/3cbe13e58a4a20b4b3aaa1afbdc69747a7c37933/playwright/sync_api/_generated.py#L12729



