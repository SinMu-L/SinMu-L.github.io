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

重试后看看效果