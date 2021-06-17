# Eureka配置

**Eureka Server 1**

```yml
spring:
  application:
    name: Server-1
server:
  port: 30000
eureka:
  instance:
    prefer-ip-address: true
    status-page-url-path: /actuator/info
    health-check-url-path: /actuator/health
    hostname: localhost
  client:
    register-with-eureka: true
    fetch-registry: true
    prefer-same-zone-eureka: true
    #地区
    region: beijing
    availability-zones:
      beijing: zone-1,zone-2
    service-url:
      zone-1: http://localhost:30000/eureka/
      zone-2: http://localhost:30001/eureka/
```

**Eureka Server 2**

```yml
spring:
  application:
    name: Server-2
server:
  port: 30001
eureka:
  instance:
    prefer-ip-address: true
    status-page-url-path: /actuator/info
    health-check-url-path: /actuator/health
    hostname: localhost
  client:
    register-with-eureka: true
    fetch-registry: true
    prefer-same-zone-eureka: true
    #地区
    region: beijing
    availability-zones:
      beijing: zone-2,zone-1
    service-url:
      zone-1: http://localhost:30000/eureka/
      zone-2: http://localhost:30001/eureka/
```


**Eureka Client 1**

```yml
spring:
  application:
    name: Client-1
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    prefer-same-zone-eureka: true
    #地区
    region: beijing
    availability-zones:
      beijing: zone-2,zone-1 #zone-2为当前Zone
    service-url:
      zone-1: http://localhost:30000/eureka/,http://localhost:30001/eureka/
      zone-2: http://localhost:20000/eureka/,http://localhost:20001/eureka/
```

> **注意⚠️**：Region后面的Zone列表配置需要注意Zone的顺序，第一个Zone是当前Client所在的Zone！
