# Storm 1.1.0 中文文档

![](docs/img/logo.png)

Apache Storm 是一个免费的，开源的，分布式的实时计算系统.

> #### NOTE（注意）
> 
> 在最新版本中, class packages 已经从 "backtype.storm" 改变成 "org.apache.storm" 了, 所以使用旧版本编译的 topology 代码不会像在 Storm 1.0.0 上那样运行了. 通过以下配置提供向后的兼容性
> 
> `client.jartransformer.class: "org.apache.storm.hack.StormShadeTransformer"`
> 
> 如果要运行使用较旧版本 Storm 编译的代码, 则需要在 Storm 安装中添加上述配置. 该配置应该添加到您用于提交 topologies（拓扑）的机器中.
> 
> 更多细节, 请参阅 [https://issues.apache.org/jira/browse/STORM-1202](https://issues.apache.org/jira/browse/STORM-1202).

## 维护地址

+   [在线阅读](http://storm.apachecn.org)
+   [在线阅读（Gitee）](https://apachecn.gitee.io/storm-doc-zh/)
+   [EPUB 格式](https://github.com/apachecn/storm-doc-zh/raw/dl/Storm%201.1.0%20%E4%B8%AD%E6%96%87%E6%96%87%E6%A1%A3.epub)
+   [Github](https://github.com/apachecn/storm-doc-zh/)

## 历史版本

+   [Storm 1.1.0 官方文档中文版](./)，整体翻译进度为 96%，更多细节请看[时光轴](docs/77.md)  
+   [Storm 1.0.1 官方文档中文版](http://cwiki.apachecn.org/pages/viewpage.action?pageId=2884006)（抱歉, 只翻译了一点点, 后面翻译其它的文档去了，现在已经迭代到 Storm 1.1.0 的中文文档中）

## 贡献指南

[请见这里](CONTRIBUTING.md)

## 负责人

* [@wangyangting](https://github.com/wangyangting)（那伊抹微笑）

## 贡献者

贡献者可自行编辑如下内容.

### 1.1.0

* [@wangyangting](https://github.com/wangyangting)（那伊抹微笑）
* [@jiangzhonglian](https://github.com/jiangzhonglian)（片刻）
* [@chenyyx](https://github.com/chenyyx)（Joy yx）
* [@XiaoLiz](https://github.com/XiaoLiz)（VoLi）
* [@huangtianan](https://github.com/huangtianan)（huangtianan）
* [@kris37](https://github.com/kris37)（kris37）
* [@stealthsMrs](https://github.com/stealthsMrs)（stealthsMrs）
* [@youyj521](https://github.com/youyj521)（冷夜雨）
* [@mikechengwei](https://github.com/mikechengwei)（mike）
* [@needav](https://github.com/needav)（needav）
* [@foxliuxiang](https://github.com/foxliuxiang)（foxliuxiang）
* [@cybj123](https://github.com/cybj123)（cybj123）
* [@ssandl](https://github.com/ssandl)（ssandl）

### 1.0.1

请参阅: [http://cwiki.apachecn.org/pages/viewpage.action?pageId=11534485](http://cwiki.apachecn.org/pages/viewpage.action?pageId=11534485)

## 联系方式

有任何建议反馈, 或想参与文档翻译, 麻烦方式如下:

* 企鹅: 1042658081
* 企鹅群: 214293307


## 下载

### Docker

```
docker pull apachecn0/storm-doc-zh
docker run -tid -p <port>:80 apachecn0/storm-doc-zh
# 访问 http://localhost:{port} 查看文档
```

### PYPI

```
pip install storm-doc-zh
storm-doc-zh <port>
# 访问 http://localhost:{port} 查看文档
```

### NPM

```
npm install -g storm-doc-zh
storm-doc-zh <port>
# 访问 http://localhost:{port} 查看文档
```
