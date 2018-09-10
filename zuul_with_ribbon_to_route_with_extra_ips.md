# zuul with ribbon to route with extra ips

## firstly you need to set routes in zuul config

```yaml
zuul:
    host:
        connect-timeout-millis: 10000
        socket-timeout-millis: 60000
    routes:
      ms:
        path: /ms/**
        sensitiveHeader:
        serviceId: ms-service
        stripPrefix: false
        retryable: false
    retryable: false
```

## then you need to set custom service in root config and set ips with listOfServers

```yaml
ms-service:
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
    ConnectTimeout: 10000
    ReadTimeout: 60000
    MaxTotalHttpConnections: 1000
    MaxConnectionsPerHost: 1000
    listOfServers: 127.0.0.1:8000, 127.0.0.1:8100

```