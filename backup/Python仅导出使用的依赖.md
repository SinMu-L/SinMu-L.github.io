代码仓库：https://github.com/bndr/pipreqs

命令：
```shell
pip install pipreqs
```

如果不需要 Jupyter Notebook 的支持，可以安装 pipreqs，但不需要依赖任何依赖项。具体操作如下：
```shell
pip install --no-deps pipreqs
pip install yarg==0.1.9 docopt==0.6.2
```

使用
```
pipreqs /home/project/location
```

requirements.txt 的内容如下
```
wheel==0.23.0
Yarg==0.1.9
docopt==0.6.2
```