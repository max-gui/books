# zuul with ribbon to set your own serverlists

## you need CustomServerList which extends AbstractServerList<Server>

```java
    public class CustomServerList extends AbstractServerList<Server>
```

## you can read serverslist from application.yml

```yaml
my-service:
  ribbon:
    NIWSServerListClassName: CustomServerList
    ConnectTimeout: 10000
    ReadTimeout: 60000
    MaxTotalHttpConnections: 1000
    MaxConnectionsPerHost: 1000
    listOfServers: 127.0.0.1:8000, 127.0.0.1:8100

```

```java
    @Value("${awf-service.ribbon.listOfServers}")
    private String ListOfServers;
```

## you need to override three methods

```java
    @Override
    public List<Server> getInitialListOfServers() {
        return getUpdatedListOfServers();
    }

    @Override
    public List<Server> getUpdatedListOfServers() {
        // get servers from database by clientName
        String ListOfServerss = "127.0.0.1:8000, 127.0.0.1:8100";
        // make your own derive to determine which servers do you want
        return derive(this.ListOfServers);
    }

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {
        this.clientConfig = clientConfig;
    }
```

## add your method to determine which servers do you want

```java
    private List<Server> derive(String value) {

        final List<Server> list =
            Arrays.stream(value.split(","))
                //map string to Server
                .map(e -> new Server(e.trim()))
                //filter for unhealthy Server
                .filter(e -> {
                    Request request = new Request.Builder()
                        .url("http://" + e.getHostPort() + "/info")
                        .build();
                    OkHttpClient client = new OkHttpClient();

                    try {
                        String resp_str = client.newCall(request).execute().body().string();
                        logger.info("resp_str: " + resp_str);
                        final JsonElement jobj = new JsonParser().parse(resp_str);
                        final String info_str =
                            jobj.getAsJsonObject()
                                .getAsJsonArray("data").get(0)
                            .getAsJsonObject().get("info.control").getAsString();
                        return info_str.toLowerCase().equals("ok");
                    } catch (Exception e1) {
                        logger.info("refresh_error", e1);
                        e1.printStackTrace();
                        return false;
                    }
                })
                .collect(Collectors.toList());

        logger.info(String.valueOf(list));
        return list;
    }
}
```

## that's all

<details>
<summary>All yaml config</summary>

```yaml
spring:
  application:
    name: my-zuul


eureka:
  instance:
    prefer-ip-address: true

feign:
  httpclient:
    enabled: false
  okhttp:
    enabled: true


ribbon:
  httpclient:
    enabled: false
  okhttp:
    enabled: true

zuul:
    host:
        connect-timeout-millis: 10000
        socket-timeout-millis: 60000
    routes:
      ms:
        path: /ms/**
        sensitiveHeader:
        serviceId: my-service
        stripPrefix: false
        retryable: false
    retryable: false

hystrix:
    command:
        default:
            execution:
                isolation:
                    thread:
                        timeoutInMilliseconds: 60000

---
#org
spring:
  profiles: org

server:
  port: 8040

eureka:
  client:
    service-url:
      defaultZone: http://10.13.12.179:8761/eureka/

my-service:
  ribbon:
    NIWSServerListClassName: CustomServerList
    ConnectTimeout: 10000
    ReadTimeout: 60000
    MaxTotalHttpConnections: 1000
    MaxConnectionsPerHost: 1000
    listOfServers: 127.0.0.1:8000, 127.0.0.1:8100

---
```

</details>

<details>
<summary>All java code</summary>

```java
import com.google.gson.JsonElement;
import com.google.gson.JsonParser;
import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractServerList;
import com.netflix.loadbalancer.Server;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;

public class CustomServerList extends AbstractServerList<Server> {

//  @Autowired
//  private ServerListRepository repository;
private Logger logger = LoggerFactory.getLogger(this.getClass());
  private IClientConfig clientConfig;

  @Value("${awf-service.ribbon.listOfServers}")
  private String ListOfServers;

  @Override
  public List<Server> getInitialListOfServers() {
    return getUpdatedListOfServers();
  }

  @Override
  public List<Server> getUpdatedListOfServers() {
// get servers from database by clientName
// String ListOfServerss = "127.0.0.1:8000, 127.0.0.1:8100";
    return derive(this.ListOfServers);
  }

  @Override
  public void initWithNiwsConfig(IClientConfig clientConfig) {
    this.clientConfig = clientConfig;
  }

  private List<Server> derive(String value) {


    final List<Server> list =
        Arrays.stream(value.split(","))
            //map string to Server
            .map(e -> new Server(e.trim()))
            //filter for unhealthy Server
            .filter(e -> {
                Request request = new Request.Builder()
                    .url("http://" + e.getHostPort() + "/info")
                    .build();
                OkHttpClient client = new OkHttpClient();

                try {
                    String resp_str = client.newCall(request).execute().body().string();
                    logger.info("resp_str: " + resp_str);
                    final JsonElement jobj = new JsonParser().parse(resp_str);
                    final String info_str =
                        jobj.getAsJsonObject()
                            .getAsJsonArray("data").get(0)
                        .getAsJsonObject().get("info.control").getAsString();
                    return info_str.toLowerCase().equals("ok");
                } catch (Exception e1) {
                    logger.info("refresh_error", e1);
                    e1.printStackTrace();
                    return false;
                }
            })
            .collect(Collectors.toList());

    logger.info(String.valueOf(list));
    return list;

    logger.info(String.valueOf(list));
    return list;
  }
}
```

</details>
