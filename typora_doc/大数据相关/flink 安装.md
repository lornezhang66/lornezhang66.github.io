# flink 安装


## 下载

>登录[Flink](http://apach.flink.org)官方网站，找到需要安装的Flink版本，1.4X

```sh
	wget http://mirror.bit.edu.cn/apache/flink/flink-1.4.0/flink-1.4.0-bin-scala_2.11.tgz
	tar -xvf flink-*.tgz
```

## 配置

> Flink web 默认启动的端口是8081 。
> 我的服务器8081已经被别的应用占了，启动成功后检查日志也没有出现错误。8081端口也没有办法访问。
> 最后查了一下。修改了一下*conf/flink-conf.yaml*
>
> >web.port: 8081 --> web.port: 8781

## 启动

```linux
	cd flink-* 
	bin/start-local.sh
```
## 验证
>查看日志
```sh
	tail -f log/flink-root-jobmanager-0-Z2zebjvditkx9xkjsrpv1Z.log
```