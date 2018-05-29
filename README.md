![build](https://api.travis-ci.org/wanghongfei/gogate.svg?branch=master)

# GoGate

Go语言实现的Spring Cloud网关，目标是性能，即使用更少的资源达到更高的QPS。

GoGate使用FastHttp库收发HTTP请求。



目前已经实现的功能有:

- 基于Eureka的服务发现、注册
- 请求路由、路由配置热更新
- 负载均衡
- 灰度发布(基于Eureka meta信息里的version字段分配流量)
- 微服务粒度的QPS控制
- 微服务粒度的流量统计(暂时实现为记录日志到/tmp目录下)

初步测试了一下性能，结论如下：

相同的硬件环境、Zuul充分预热且关闭Hystrix的前提下，Go版的网关QPS为Zuul的2.3倍，同时内存占用仅为Zuul的十分之一(600M vs 50M)。而且Go基本上第一波请求就能达到最大QPS, zuul要预热几次才会稳定。更详细的测试近期更新。



## 流程

![arc](http://ovbyjzegm.bkt.clouddn.com/gogate-arc.jpg)

服务路由: 根据URL匹配后端微服务

流量控制: 令牌桶算法控制qps

URL重写: 调整向后端服务发请求的URL

转发请求: 负载均衡、按比例分配流量



gogate没有提供默认的Post Filter，可根据需要自己实现相应函数。



## 使用

可以编译`main.go`直接生成可执行文件，也可以当一个库来使用，见`examples/usage.go`





## 路由配置

规则:

- 当`id`不为空时，会使用eureka的注册信息查询此服务的地址
- 当`host`不为空时, 会优先使用此字段指定的服务地址, 多个地址用逗号分隔
- 当请求路径匹配多个`prefix`时，配置文件中`prefix`最长的获胜

当路由配置文件发生变动时，访问

```
GET /_mgr/reload
```

即可应用新配置。

```yaml
services:
  user-service:
    # eureka中的服务名
    id: user-service
    # 以/user开头的请求, 会被转发到user-service服务中
    prefix: /user
    # 转发时是否去掉请求前缀, 即/user
    strip-prefix: true
    # 灰度配置
    canary:
      -
        # 对应eurekai注册信息中元数据(metadata map)中key=version的值
        meta: "1.0"
        # 流量比重
        weight: 3
      -
        meta: "2.0"
        weight: 4
      -
      	# 对应没有metadata的服务
        meta: ""
        weight: 1

  trends-service:
    id: trends-service
    # 请求路径当匹配多个prefix时, 长的获胜
    prefix: /trends
    strip-prefix: false
    # 设置qps限制, 每秒最多请求数
    qps: 1

  order-service:
    id: order-service
    prefix: /order
    strip-prefix: false

  img-service:
    # 如果有host, 则不查注册中心直接使用此地址, 多个地址逗号分隔
    host: localhost:4444,localhost:5555
    prefix: /img
    strip-prefix: false

# 上面都没有匹配到时
  common-service:
    id: common-service
    prefix: /
    strip-prefix: false
```



## Eureka配置

`eureka.json`文件



## gogate配置

`gogate.json`文件:

```json
{
    "appName": "gogate", // 向eureka注册时使用的服务名
    "host": "127.0.0.1",
    "port": 8080,
    "maxConnection": 1000, // gogate最大可接受的连接数
    "timeout": 3000, // gogate调用后端服务超时时间, 毫秒
    "version": "1.0",

    "eurekaConfig": "eureka.json", // eureka配置文件名
    "routeConfig": "route.yml", // 路由配置文件名

    "recordTraffic": true, // 是否开启流量记录功能
    "trafiicDir": "/tmp" // 流量日志文件写入目录, 当recordTraffic = true时有效
}
```





## 使用方法

可以在转发请求之前和之后添加自定义Filter来添加自定义逻辑。

见`examples/usage.go`