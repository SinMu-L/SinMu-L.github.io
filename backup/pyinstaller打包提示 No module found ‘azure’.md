![Image](https://github.com/user-attachments/assets/bc63f526-06d3-409c-95ae-952c5e69887a)

这个错误折磨了我很久，目前也没有解决。

AI给到的建一个是将 `main.spec` 文件中的`hiddenimports=[ 'azure.storage.blob', 'azure.identity'],` 改为 `hiddenimports=['azure', 'azure.storage', 'azure.storage.blob', 'azure.identity'],`
```
a = Analysis(
    ['sync2cloud.py'],
    pathex=[],
    binaries=[],
    datas=[],
    hiddenimports=['azure', 'azure.storage', 'azure.storage.blob', 'azure.identity'],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    noarchive=False,
    optimize=0,
)
```

## 我尝试过的方案

1. 【无效，同样的报错】使用 pyinstaller --onefile --hidden-import=azure --hidden-import=azure.storage --hidden-import=azure.storage.blob --hidden-import=azure.identity --hidden-import=azure.core sync2cloud.py 
2. 【无效，同样的报错】.spec文件中添加hiddenimports=['azure', 'azure.storage', 'azure.storage.blob', 'azure.identity', 'azure.core'],
3. 【无效，同样的报错】.spec文件中添加hiddenimports=[ 'azure-storage-blob', 'azure-identity'],


之前使用的是 gemini pro 2.5 preview，更换成claude 3.7 sonnet 效果斐然，下面记录下解决的步骤

## 解决步骤

我先是提供了这么一段提示词，作为system prompt
```
环境：windows

确认当前 Python 路径：D:\sinmu\code\python_code\chat_mcp_sync\venv\Scripts\python.exe

python -c "import sys; print(sys.executable)" 的执行结果是：D:\sinmu\code\python_code\chat_mcp_sync\venv\Scripts\python.exe

我确定我激活了虚拟环境

我现在准备用pyinstaller 打包 D:\sinmu\code\python_code\chat_mcp_sync\sync2cloud.py 文件。

总是未成功，你根据我的问题给出一些建议和你的思考
```

然后给出了一些建议，说是修改 spec 文件，提到了指定 azure 依赖的路径

据此我就检查了下使用的python编译器的路径，发现默认使用的是系统python编译器路径，没有使用虚拟环境中的路径

```
497 INFO: PyInstaller: 6.11.1, contrib hooks: 2024.11
498 INFO: Python: 3.12.1
527 INFO: Platform: Windows-10-10.0.19045-SP0
527 INFO: Python environment: C:\Users\xxx\AppData\Local\Programs\Python\Python312
使用的 site-packages 路径: C:\Users\xxx\AppData\Local\Programs\Python\Python312\Lib\site-packages
```

重新激活虚拟环境。并使用` python -c "import sys; print(sys.executable)"` 来确保 python编译器的路径
```shell
$ python -c "import sys; print(sys.executable)"
D:\sinmu\code\python_code\chat_mcp_sync\venv\Scripts\python.exe
```

然后重新执行`pyinstaller sync2cloud.spec` 就解决了 【No module found ‘azure’】问题


-----

spec文件放到最后
```
# -*- mode: python ; coding: utf-8 -*-
import site
import os
import sys


# 获取当前虚拟环境的 site-packages 目录
site_packages = next((p for p in sys.path if p.endswith('site-packages')), None)
print(f"使用的 site-packages 路径: {site_packages}")

a = Analysis(
    ['sync2cloud.py'],
    pathex=[],
    binaries=[],
    datas=[(os.path.join(site_packages, 'azure'), 'azure')],
    hiddenimports=['azure', 'azure.storage', 'azure.storage.blob', 'azure.identity', 'azure.core', 
                  'azure.storage.blob._shared', 'azure.storage.blob._generated',
                  'azure.identity._internal', 'msrest', 'msrestazure'],
    hookspath=[],
    hooksconfig={},
    runtime_hooks=[],
    excludes=[],
    noarchive=False,
    optimize=0,
)
pyz = PYZ(a.pure)

exe = EXE(
    pyz,
    a.scripts,
    a.binaries,       # 包含二进制文件
    a.zipfiles,       # 包含压缩文件
    a.datas,          # 包含数据文件
    [],
    name='sync2cloud',
    debug=False,
    bootloader_ignore_signals=False,
    strip=False,
    upx=True,
    console=True,
    disable_windowed_traceback=False,
    argv_emulation=False,
    target_arch=None,
    codesign_identity=None,
    entitlements_file=None,
)


```