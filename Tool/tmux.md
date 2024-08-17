tmux

新建会话

```
tmux new -s <session-name>
```

接入已创建的会话

```
tmux a -t <session-name>
```


快捷键

```
ctrl+b d	分离当前会话
ctrl+d  	杀死当前会话

ctrl+b "	将当前窗口水平分隔
ctrl+b %	将当前窗口垂直分隔

ctrl+b p 	切换上一个使用的窗口
ctrl+b n	切换下一个使用的窗口
```

![tmux_pane_v](D:\_temp\网络图片\tmux_pane_v.png)

配置

```
ctrl+b :  				进入配置模式
set -g mode-mouse on	进入鼠标模式
```

