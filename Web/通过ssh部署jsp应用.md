# 通过ssh部署jsp应用

标签（空格分隔）： Web

---
- ssh登录远程服务器
```
ssh userName@server
```
- 上传war包服务器的临时目录(比如/var/temp)
```
scp /path/filename username@servername:/var/temp
```
- 停掉服务
```
/etc/init.d/tomcat7 stop
```
- 删除原来app
```
rm -rf /var/tomcat/war/xxx.war 
```
- 清除缓存
```
rm -rf /var/tomcat/apache-tomcat-7.0.63/work/Catalina/* /*清除缓存*/
```
- 拷贝war包到tomcat的app目录下（默认是webapps）
```
cp /var/tmp/xxx.war /var/tomcat/war/xxx.war
```
- 重启服务
```
/etc/init.d/tomcat7 start
```
- 查看tomcat启动日志，检查启动是否有异常 
```
tail -100f /var/tomcat/apache-tomcat-7.0.63/logs/catalina.out
```
## 中文乱码
假设编码用utf-8:
1. tomcat日志乱码
	解决：在`catalina.sh`增加 `JAVA_OPTS="-Dfile.encoding=utf-8"`
	注：这个参数必须在jvm启动时加上，在程序中通过设置system property的方式是没有效果的，原因是jvm启动时读取file.encoding并cache，后续只使用启动时读取的编码。	
2. tomcat参数的乱码问题
	解决：在server.xml的connector中增加 URIEncoding="utf-8"




