[TOC]

# Consul

## -dev

### $ consul agent -dev

使用dev模式启动consul agent

### $ consul members

查看现存node

> 另一种方式 curl localhost:8500/v1/catalog/nodes
>
> 使用http请求也可以获取members

### $ dig

对consul的dns服务器发送请求 

```shell
dig @127.0.0.1 -p 8600 Judiths-MBP.node.consul
```

### $ consul leave

shut down gracefully



### $ dig @127.0.0.1 -p 8600 {service_name}.service.consul

对特定的服务进行dig

> curl http://localhost:8500/v1/catalog/service/{service_name}



### sidecar_service



