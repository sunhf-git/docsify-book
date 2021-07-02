# 前言
Apache Nifi是做数据集成常用框架,擅长在IOT领域进行数据采集，通常在生产环境中通过Nifi提供的API和自定义扩展实现自身的业务需求。由于时间关系和文章篇幅控制，一些网络上讲的比较清晰的文章这里会直接给出链接。不需要重复造轮子，nifi学习资料匮乏，通过写专栏的方式实现整理出一套健全的学习文档。

# Apache Nifi
[nifi工作原理及相关产品](https://cloud.tencent.com/developer/article/1596567)
[中文用户指南](https://nifichina.github.io/general/overview.html#nifi%E6%98%AF%E4%BB%80%E4%B9%88)
[官方文档](https://nifi.apache.org/docs.html)
[官方用户指南](https://nifi.apache.org/docs/nifi-docs/html/user-guide.html)
nifi是一个在各异构数据源中进行数据集成的一套框架，主要用于处理大数据的数据集成。以下是NIFI的设计原则，它主要通过各种处理器实现数据格式和字段的转换来实现数据集中。
[NIFI的关键特性](https://www.cnblogs.com/hdpdriver/p/10738855.html)
- 基于Web的用户界面
    - 设计，控制，反馈和监视之间的无缝体验
- 高度可配置
    - 容忍损失与保证交付
    - 低延迟与高吞吐量
    - 动态优先级
    - 可以在运行时修改流程
    - 背压
- 资料来源
    - 从头到尾跟踪数据流
- 专为扩展而设计
    - 构建自己的处理器等
    - 实现快速开发和有效测试
- 安全
    - SSL，SSH，HTTPS，加密内容等...
    - 多租户授权和内部授权/策略管理
    
    
# 快速开始
[nifi 简介](https://peterxugo.github.io/2017/03/30/nifi-%E7%AE%80%E4%BB%8B/)
[下载安装nifi](https://nifichina.github.io/general/GettingStarted.html#%E4%B8%8B%E8%BD%BD%E5%AE%89%E8%A3%85nifi)

# Nifi WebUI基本操作
[使用webui 实现文件同步](https://blog.csdn.net/dongdong9223/article/details/84940992)

# Nifi的架构设计
[nifi 架构设计](https://peterxugo.github.io/2017/06/06/nifi%E6%9E%B6%E6%9E%84/)

## 组与标签

## 用户访问权限与登录


# 开发自定义处理器
事实上，nifi提供的默认处理器在生产环境中能用上的并不多，因为各种业务需要处理器的使用仍然需要以自定义为主。自定义的处理器写完之后打包后的nar文件放入nifi服务端的extensions文件夹即可。
开发自定义处理器的原理其实还是通过spi技术将自定义的Processor实现类加载进去。具体实现过程建议以nifi自带的处理器源码作为参考。
[demo地址(默认版本较旧，将nifi版本升级到最新的就行了,这个项目格式规范一些)](https://github.com/aperepel/nifi-workshop)
[开发一个简单的json处理器](https://blog.csdn.net/mianshui1105/article/details/75313480)

# ExecuteScript使用
[官方文档翻译1](https://blog.csdn.net/weixin_39445556/article/details/84781864)
[官方文档翻译2](https://blog.csdn.net/weixin_39445556/article/details/84794735)
[官方文档翻译3](https://blog.csdn.net/weixin_39445556/article/details/84838528)

# 开发自定义报告任务

# 实现支持字段转换的处理器


# 使用Nifi提供的RestApi进行任务管理

# 高性能nifi
[使用nifi处理10亿/秒数据](https://cloud.tencent.com/developer/article/1618205)
[Nifi团队性能优化文章](https://blog.csdn.net/weibokong789/article/details/88554855?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522158780494219724835812110%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=158780494219724835812110&biz_id=0&utm_source=distribute.pc_search_result.none-task-blog-2~blog~first_rank_v2~rank_v25-13)
[nifi优化配置](https://blog.csdn.net/weixin_39445556/article/details/84339249)
[nifi性能测试](https://blog.csdn.net/qq_28326077/article/details/62232130?ops_request_misc=&request_id=&biz_id=102&utm_source=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-0)

# 实战例子
[从mysql binlog中读取数据到Hbase](https://blog.csdn.net/baixf/article/details/94622813)

