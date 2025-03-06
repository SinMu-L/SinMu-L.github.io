## 背景

我希望一个脚本新增数据到数据表中，再启动另一个脚本更新数据表的字段，但是每次启动两个脚本太麻烦了，我就想能不能一个启动一个脚本同时执行两个任务。

通过AI的帮助，写了下面的启动和清理多线程的代码块。感觉后续还会用到，做下记录。


```python
import multiprocessing
import time
from loguru import logger
import signal

# 子进程任务
def function_a(stop_event):
    while not stop_event.is_set():
        logger.info("正在新增数据到数据库...")
        time.sleep(2)
    logger.info("function_a 已停止")

def function_b(stop_event):
    while not stop_event.is_set():
        logger.info("正在更新数据库字段...")
        time.sleep(3)
    logger.info("function_b 已停止")

def handle_exit(signum, frame):
    logger.info("\n收到中断信号，正在准备退出程序...")
    stop_event.set()

if __name__ == "__main__":
    # 创建停止信号
    stop_event = multiprocessing.Event()

    # 注册信号处理器
    signal.signal(signal.SIGINT, handle_exit)

    # 创建进程
    process_a = multiprocessing.Process(target=function_a, args=(stop_event,))
    process_b = multiprocessing.Process(target=function_b, args=(stop_event,))

    # 启动进程
    process_a.start()
    process_b.start()

    try:
        # 主进程保持运行
        process_a.join()
        process_b.join()
    except KeyboardInterrupt:
        logger.info("程序手动中断")
    finally:
        # 设置停止信号并等待子进程退出
        stop_event.set()
        process_a.join()
        process_b.join()
        logger.info("已清理所有子进程并退出程序。")

```