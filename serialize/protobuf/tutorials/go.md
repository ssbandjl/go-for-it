# 教程

参考文档: https://developers.google.com/protocol-buffers/docs/gotutorial



## 编译protocal buffers

安装插件:

```sh
go get google.golang.org/protobuf/cmd/protoc-gen-go
```

在当前目录生成

```
protoc -I=. --go_out=. ./addressbook.proto
```



