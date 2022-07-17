# 按配置加载Bean
```java
@ConditionalOnProperty(
    prefix = "spring.mail",
    name = {"host"}
)
```

配置了spring.mail.host的才加载当前配置



