环境：windows 10



问题：VSCode 连接远程服务器文件夹的时候提示下面的错误
> BadInstallScriptResult │ Error: BadInstallScriptResult (Got bad result from install script)       


吐槽：上周五还好好的，这周一就不行了，不知道啥原因

找到了这个解决方案：

原因： 注册表中 `HKEY_CURRENT_USER\Software\Microsoft\Command Processor\AutoRun` 的默认值是空白，而非是某个环境变量或者0

![Image](https://github.com/user-attachments/assets/6eb953c1-58c0-4b23-b5b5-519f333ba9df)


答案来源：https://github.com/microsoft/vscode-remote-release/issues/6527