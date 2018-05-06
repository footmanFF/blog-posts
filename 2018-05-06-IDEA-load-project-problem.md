去打开 IDEA 的 DEBUG 日志，日志地址在 help - Show log in finder 里，日志配置文件在 /Applications/IntelliJ IDEA.app/Contents/bin/log.xml。启动时的日志有一段明显很值得怀疑：

```
2018-05-06 15:34:45,576 [  93768]   INFO - ij.components.ComponentManager - com.seventh7.mybatis.ref.CmProject initialized in 75086 ms
2018-05-06 15:34:45,588 [  93780]   INFO - ellij.project.impl.ProjectImpl - 152 project components initialized in 75849 ms
```

com.seventh7.mybatis.ref.CmProject 初始化好了 75 秒。去把 mybatis 插件禁用以后重新测试：

```
2018-05-06 15:40:13,639 [ 138286]   INFO - ellij.project.impl.ProjectImpl - 150 project components initialized in 999 ms
```

禁用以后明显解决了，这个插件记得是之前安装的破解的 mybatis 插件。重新去 idea 插件仓库里搜，现在又了一个 Free MyBatis Plugin：

![2018-05-06 at 3.58 PM](/var/folders/fd/ptrbg3sx0cv0k988y2qdqxnm0000gn/T/se.razola.Glui2/C1B001A7-8D9A-4196-B6FF-15F425AC6472-481-0000BC842DC1C5CA/2018-05-06 at 3.58 PM.png) 



