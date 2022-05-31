# 前言

最近在SpringCloudGateway网关加上动态路由和限流的功能，使用RequestRateLimiter配置令牌桶时，启动应用报了异常，异常信息为Unable to find GatewayFilterFactory with name RequestRateLimiter

![](http://img-blog.csdnimg.cn/79b0e73803b84580879be26346a03865.jpeg)

原因是：引入

```groovy
org.springframework.boot:spring-boot-starter-data-redis-reactive
```

 依赖时，排除了lettuce-core依赖（当时一个同事使用Lettuce操作Redis遇到问题，排除这个依赖使用Jedis了），导致生成RequestRateLimiterGatewayFilterFactory实例的时候生成不了，但是项目中又配置了限流，导致获取限流的Factory获取不到。

删除掉排除的lettuce-core依赖即可恢复正常。

# 定位过程

看到一个异常，一般先根据异常堆栈找到报错的代码，然后再报错代码附近打上断点进行Debug，根据报错信息往上一层层推，有一部分可能源码调用链嵌套很多，很难找出报错的代码，另一种方法就是根据报错的Message直接搜项目代码（包括Jar包里面，搜索不能完全用报错信息搜，拷贝部分报错信息，因为有些是变量拼接的，一部分一部分试）。

在这个问题里面，我是直接搜索异常信息，搜索 Unable to find

![](https://img-blog.csdnimg.cn/5c3f3bd22c4242968c7c62f5bf3d6934.jpeg)

发现是在 RouteDefinitionRouteLocator类的loadGatewayFilters方法中，直接打上断点去调试

![](https://img-blog.csdnimg.cn/75af590370cd42a08463927f1c9c7623.jpeg)

fatory是从当前类的gatewayFilterFactories中去取的，然后看下这个属性是如何初始化的

![](https://img-blog.csdnimg.cn/4756e78fcd3b47de81ab1f2ae4b9910f.jpeg)

这个属性也是从其他类传进来的，只是将传进来的GatewayFilterFactory的List转成Map，在往上看

![](https://img-blog.csdnimg.cn/4ed46f67d0c44055ae500cbb472ea2ab.jpeg)

是在GatewayAutoConfiguration配置类里面进行实例化RouteDefinitionRouteLocator类的，所以gatewayFilters是每个GatewayFilterFactory通过配置导入到Spring容器的，我们直接看限流使用到的RequestRateLimiterGatewayFilterFactory是在哪配置的，搜索这个类使用情况，发现是在GatewayAutoConfiguration类中配置的，配置代码如下

![](https://img-blog.csdnimg.cn/b3bb9c7f45704f3194b4e4556ff1f454.jpeg)

可以看到这里注入和很多（截图只截了三个）GatewayFilterFactory，RequestRateLimiterGatewayFilterFactory这个Bean的实例化是依赖RateLimiter的，因为限流默认是使用Redis的，直接看RedisRateLimiter这个类的配置，搜索这个类的使用情况，是在GatewayRedisAutoConfiguration类中配置的

![](https://img-blog.csdnimg.cn/434e599b4fe64ef7a5deee2da60b421d.jpeg)

RedisRateLimiter这个Bean又依赖ReactiveStringRedisTemplate，ReactiveStringRedisTemplate这个类是在RedisReactiveAutoConfiguration类中配置的

![](https://img-blog.csdnimg.cn/c32d6ae8d7ea4e8da07081ee574f7cb0.jpeg)

ReactiveStringRedisTemplate又依赖于ReactiveRedisConnectionFactory，这个Redis连接工厂唯一的实现类是LettuceConnectionFactory，LettuceConnectionFactory是基于Lettuce的，所以这个bean是不会实例化的，层层依赖导致RequestRateLimiterGatewayFilterFactory创建不了实例。



# 总结

遇到问题及时解决，不应该在项目中留下隐患，可能会给其他人员带来很大的技术债。

解决问题时一步步调试，多DEBUG，DEBUG过程中会对变量、方法调用过程有更清晰的认识，比如刚开始抛异常时，new RequestRateLimiterGatewayFilterFactory这句代码是不会跑的，即使打了断点，但是new其他的GatewayFilterFactory我也打了断点试了，却是会跑的，就把方向放在了RequestRateLimiterGatewayFilterFactory这个类的实例化上，再一步步往下调，所以我截图上代码基本都是有断点的。
