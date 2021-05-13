# Java 工程经验

## Debug

1. 通过 `java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 -jar xxx.jar` 命令启动 jar 包可以开启默认的远程调试端口。
