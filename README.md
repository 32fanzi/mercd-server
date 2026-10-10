# mercd-server

CN3.5 本地复原服务端。提供 Go 服务、SQLite 持久化、协议实现、gomobile 入口与本地客户端准备脚本。

## 快速启动

需要 Go 1.25 或更新版本，具体要求以 `server/go.mod` 为准。Go 模块位于 server，模块路径为 `github.com/Dnchua/mercd-server/server`，与既有 origin 对应。根 Makefile 转发开发命令。

```sh
make env       # 首次创建本地 .env，不覆盖已有文件
make deps      # 首次下载和校验依赖
make check
make build
make run       # 加载仓库根 .env，前台运行，Ctrl-C 停止
```

已准备 `.env` 的本地副本，默认监听回环地址，存档写入 state/cn35，请求记录默认关闭。其他机器克隆后先执行 make env。原游戏 APK、Unity 素材和故事内容须自行准备，服务端构建通过不代表完整游戏或客户端渲染已验收。

## 环境配置

[.env.example](.env.example) 可提交，`.env`、其他本地环境文件、玩家库和构建输出由 .gitignore 排除。加载优先级为显式命令行参数、系统环境变量、环境文件、程序默认值；支持 KEY=VALUE、export 前缀与成对引号，不执行 shell、不展开变量。

make run 和 server/run.sh 总是从仓库根运行，因此模板相对路径按仓库根解析。直接运行二进制时相对路径按当前目录解析；可用 -env-file 或 MERCD_ENV_FILE 选择其他文件，未指定时依次探测 mercd35.env、.env。显式路径不存在会报错。

```sh
server/run.sh --no-build -- -addr 127.0.0.1:18099 -server-host 127.0.0.1:18099
```

LAN 使用时同时调整 MERCD_ADDR 和 MERCD_SERVER_HOST；server-host 必须是客户端可达的具体 IP:端口。自己的网络地址不写进公开模板。

## 文档与数据

- [完整技术文档索引](docs/README.md)：页面全集、协议、数值、事务、公会与资源接入。
- [运行与自备材料](docs/运行与材料.md)。
- [数据来源](docs/数据来源.md)及[运行依赖](docs/01-数据来源与运行依赖.md)。
- [自备输入目录](inputs/README.md)。

shared/cards 是当前运行依赖，不能仅因离线工具也读取就删除。纪念版补缺数据与 CN3.5 配置包含混合来源，来源声明保留；不提供原反编译源码、原始玩家数据或签名／SDK 秘密。

## 开发与 Git 约定

提交源码、文档、go.mod、go.sum 和公开配置模板。不要提交 .env、客户端解码树、资源正文、state、WAL、备份或日志。代码格式使用 gofmt；Makefile 使用 tab，脚本使用 LF。历史 SQL 按原字节保存，.gitattributes 禁止换行转换，不能修改或删除既有迁移。

make check 执行格式、vet 和测试；make race 执行竞态检查；make cross 构建 Windows amd64、Linux amd64/arm64。真实故事与 bundle 缺失时相关测试明确 skip；API 文档可用 make api 生成。既有库升级前执行一致性备份，恢复配对二进制与新的 state 目录。

MIT LICENSE 用于有权许可的代码和文档；第三方数据来源与权利说明单独保留，见[数据来源](docs/数据来源.md)。
