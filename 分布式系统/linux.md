环境变量相关

```
echo $PATH // 打印环境变量路径，在该路径下的脚本能够在任何目录下使用
// 配置环境变量
sudo vim /etc/profile.d/my_env.sh     ## 该目录下的脚本文件会在系统启动时执行，也可手动 source /etc/profile 更新
```

my_env.sh一般配置

```
export JAVA_HOME=your jdk dir    # 键入你的jdk环境目录
export PATH=$PATH:$JAVA_HOME/bin	# 追加环境变量
```

启动历史服务器

```
mapred --daemon start historyserver
```

mapreduce作业

```
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-3.1.3.jar wordcount /input /output
```

