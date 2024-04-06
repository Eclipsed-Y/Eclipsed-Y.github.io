---
title: "利用SpringCache实现Redis随机过期时间的解决方案"
date: 2024-04-05T11:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags: ["first"]
author: "Me"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Desc Text."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/<path_to_repo>/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

## 起因

为了防止缓存雪崩，缓存过期时间最好设置成随机过期。

### 原始方案

```java
public UserInfoVO getUserInfo(Long id) {
    // 判断redis中是否存在缓存
    String key = "userInfo:" + id;
    if (Boolean.TRUE.equals(redisTemplate.hasKey(key))){
        return (UserInfoVO) redisTemplate.opsForValue().get(key);
    }
    if (id == null){
        throw new RequestExcetption(MessageConstant.ILLEGAL_REQUEST);
    }
    UserInfoDO userInfoDO = userInfoService.getOne(new QueryWrapper<UserInfoDO>().eq("user_id", id));
    if (userInfoDO == null || userInfoDO.getDeleted() == 1){
        throw new AccountNotFoundException(MessageConstant.ACCOUNT_NOT_FOUND);
    }
    UserInfoVO userInfoVO = new UserInfoVO();
    BeanUtils.copyProperties(userInfoDO, userInfoVO);
    // 装入缓存，设置随机时间
    Random random = new Random();
    redisTemplate.opsForValue().set(key, userInfoVO, 10 + random.nextInt(11), TimeUnit.SECONDS);
    return userInfoVO;
}
```

比如这里我想把用户信息数据缓存到Redis中，需要利用到RedisTemplate，为每个不同id的数据做key的拼接，然后用Random设置随机过期时间（如上是10-20分钟）。

并且因为采用旁路缓存模式，在更新数据后，同样需要对不同id做key的拼接，然后删除Redis中对应的key。如果是分页查询，那可能还需要设计一个哈希表来存储对应类别中的所有已缓存key，甚是麻烦。

因此考虑优化。

### 替换方案

```java
@Cacheable(value = "userInfo", cacheManager = "redisCacheManager", key = "#id")
public UserInfoVO getUserInfo(Long id) {
    if (id == null){
        throw new RequestExcetption(MessageConstant.ILLEGAL_REQUEST);
    }
    UserInfoDO userInfoDO = userInfoService.getOne(new QueryWrapper<UserInfoDO>().eq("user_id", id));
    if (userInfoDO == null || userInfoDO.getDeleted() == 1){
        throw new AccountNotFoundException(MessageConstant.ACCOUNT_NOT_FOUND);
    }
    UserInfoVO userInfoVO = new UserInfoVO();
    BeanUtils.copyProperties(userInfoDO, userInfoVO);
    return userInfoVO;
}
```

直接采用SpringCache，通过注解的形式让框架自动完成缓存处理。

### 新的问题

```java
@Bean
public RedisCacheManager redisCacheManager(RedisConnectionFactory factory){
    return RedisCacheManager.builder(factory)
        .withCacheConfiguration("user", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(2)))
        .build();
}
```

SpringCache需要CacheManager对缓存做配置，如上所示创建一个RedisCacheManager的Bean对象，但问题在于配置中entryTtl只能传入Duration类设置为定值，无法随机化。

## 解决

### Duration

```java
public final class Duration implements TemporalAmount, Comparable<Duration>, Serializable {
    // 省略
}
```

首先没法继承Duration修改里面的方法，因为该类被final修饰。

### entryTtl方法

```java
public RedisCacheConfiguration entryTtl(Duration ttl) {
    Assert.notNull(ttl, "TTL duration must not be null!");
    return new RedisCacheConfiguration(ttl, this.cacheNullValues, this.usePrefix, this.keyPrefix, this.keySerializationPair, this.valueSerializationPair, this.conversionService);
}
```

那从entryTtl方法出发，发现其作用是对当前类RedisCacheConfiguration的ttl属性赋值，那我是不是可以直接继承RedisCacheConfiguration类，然后改写这个方法，每次为ttl赋值一个随机数呢？

```java
public class RedisCacheConfiguration {
    private final Duration ttl;
    private final boolean cacheNullValues;
    private final CacheKeyPrefix keyPrefix;
    private final boolean usePrefix;
    private final RedisSerializationContext.SerializationPair<String> keySerializationPair;
    private final RedisSerializationContext.SerializationPair<Object> valueSerializationPair;
    private final ConversionService conversionService;
    private RedisCacheConfiguration(Duration ttl, Boolean cacheNullValues, Boolean usePrefix, CacheKeyPrefix keyPrefix, RedisSerializationContext.SerializationPair<String> keySerializationPair, RedisSerializationContext.SerializationPair<?> valueSerializationPair, ConversionService conversionService) {
        this.ttl = ttl;
        this.cacheNullValues = cacheNullValues;
        this.usePrefix = usePrefix;
        this.keyPrefix = keyPrefix;
        this.keySerializationPair = keySerializationPair;
        this.valueSerializationPair = valueSerializationPair;
        this.conversionService = conversionService;
    }
    // 省略代码
}
```

结果是也不能继承RedisCacheConfiguration类l，因为构造方法为私有，无法继承。

那换个思路，我什么时候需要用到ttl？

```java
public class RedisCacheManager extends AbstractTransactionSupportingCacheManager {
    private final RedisCacheWriter cacheWriter;
    private final RedisCacheConfiguration defaultCacheConfig;
    private final Map<String, RedisCacheConfiguration> initialCacheConfiguration;
    private final boolean allowInFlightCacheCreation;
    // 省略...
    protected RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
        return new RedisCache(name, this.cacheWriter, cacheConfig != null ? cacheConfig : this.defaultCacheConfig);
    }
    // 省略...
}
```

RedisCacheManager类初始化需要传入RedisCacheConfiguration配置，而在其createRedisCache方法中用到了这个配置，然后创建了一个RedisCache类。

```java
public class RedisCache extends AbstractValueAdaptingCache {
    private static final byte[] BINARY_NULL_VALUE;
    private final String name;
    private final RedisCacheWriter cacheWriter;
    private final RedisCacheConfiguration cacheConfig;
    private final ConversionService conversionService;
    protected RedisCache(String name, RedisCacheWriter cacheWriter, RedisCacheConfiguration cacheConfig) {
        super(cacheConfig.getAllowCacheNullValues());
        Assert.notNull(name, "Name must not be null!");
        Assert.notNull(cacheWriter, "CacheWriter must not be null!");
        Assert.notNull(cacheConfig, "CacheConfig must not be null!");
        this.name = name;
        this.cacheWriter = cacheWriter;
        this.cacheConfig = cacheConfig;
        this.conversionService = cacheConfig.getConversionService();
    }
    // 省略...
    public void put(Object key, @Nullable Object value) {
        Object cacheValue = this.preProcessCacheValue(value);
        if (!this.isAllowNullValues() && cacheValue == null) {
            // 抛出异常省略...
        } else {
            this.cacheWriter.put(this.name, this.createAndConvertCacheKey(key), this.serializeCacheValue(cacheValue), this.cacheConfig.getTtl());
        }
    }
    public Cache.ValueWrapper putIfAbsent(Object key, @Nullable Object value) {
        Object cacheValue = this.preProcessCacheValue(value);
        if (!this.isAllowNullValues() && cacheValue == null) {
            return this.get(key);
        } else {
            byte[] result = this.cacheWriter.putIfAbsent(this.name, this.createAndConvertCacheKey(key), this.serializeCacheValue(cacheValue), this.cacheConfig.getTtl());
            return result == null ? null : new SimpleValueWrapper(this.fromStoreValue(this.deserializeCacheValue(result)));
        }
    }
    // 省略...
}
```

进入RedisCache，可以发现其put和putIfAbsent两个方法使用了配置的getTtl方法，用于写入缓存的过期时间。

并且RedisCache的构造方法并没有私有化，因此可以继承，然后重写put和putIfAbsent两个方法。

```java
public class RandomTtlRedisCache extends RedisCache {
    /**
     * 最小ttl，单位毫秒
     */
    private int minTtl;
    /**
     * 最大ttl，单位毫秒
     */
    private int maxTtl;
    private Random random = new Random();
    private final String name;
    private final RedisCacheWriter cacheWriter;
    protected RandomTtlRedisCache(String name, RedisCacheWriter cacheWriter, RedisCacheConfiguration cacheConfig, int minTtl, int maxTtl) {
        super(name, cacheWriter, cacheConfig);
        this.minTtl = minTtl;
        this.maxTtl = maxTtl;
        this.name = name;
        this.cacheWriter = cacheWriter;

    }

    @Override
    public void put(Object key, @Nullable Object value) {
        Object cacheValue = this.preProcessCacheValue(value);
        if (!this.isAllowNullValues() && cacheValue == null) {
            throw new IllegalArgumentException(String.format("Cache '%s' does not allow 'null' values. Avoid storing null via '@Cacheable(unless=\"#result == null\")' or configure RedisCache to allow 'null' via RedisCacheConfiguration.", this.name));
        } else {
            this.cacheWriter.put(this.name, this.createAndConvertCacheKey(key), this.serializeCacheValue(cacheValue), this.getRandomTtl());
        }
    }

    @Override
    public Cache.ValueWrapper putIfAbsent(Object key, @Nullable Object value) {
        Object cacheValue = this.preProcessCacheValue(value);
        if (!this.isAllowNullValues() && cacheValue == null) {
            return this.get(key);
        } else {
            byte[] result = this.cacheWriter.putIfAbsent(this.name, this.createAndConvertCacheKey(key), this.serializeCacheValue(cacheValue), this.getRandomTtl());
            return result == null ? null : new SimpleValueWrapper(this.fromStoreValue(this.deserializeCacheValue(result)));
        }
    }

    private Duration getRandomTtl() {
        int randomTtl = minTtl + random.nextInt(maxTtl - minTtl + 1);
        return Duration.ofMillis(randomTtl);

    }

    private byte[] createAndConvertCacheKey(Object key) {
        return this.serializeCacheKey(this.createCacheKey(key));
    }
}
```

如上创建了一个RandomTtlRedisCache类继承RedisCache。如何调用这个类呢，自然是回到RedisCacheManager类中的createRedisCache方法。

同理，可以创建一个RandomTtlRedisCacheManager类继承RedisCacheManager。

```java
public class RandomTtlRedisCacheManager extends RedisCacheManager{
    private final RedisCacheWriter cacheWriter;
    private final RedisCacheConfiguration defaultCacheConfig;
    private final int minTtl;
    private final int maxTtl;

    public RandomTtlRedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration, int minTtl, int maxTtl) {
        super(cacheWriter, defaultCacheConfiguration);
        this.cacheWriter = cacheWriter;
        this.defaultCacheConfig = defaultCacheConfiguration;
        this.minTtl = minTtl;
        this.maxTtl = maxTtl;
    }

    @Override
    protected RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
        return new RandomTtlRedisCache(name, this.cacheWriter, cacheConfig != null ? cacheConfig : this.defaultCacheConfig, minTtl, maxTtl);
    }
}
```

至此，基本完成修改，只需要创建一个RandomTtlRedisCacheManager的Bean对象即可。

```java
@Bean
public RandomTtlRedisCacheManager redisCacheManager(RedisConnectionFactory factory){
    RedisCacheWriter cacheWriter = RedisCacheWriter.lockingRedisCacheWriter(factory);
    RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();
    return new RandomTtlRedisCacheManager(cacheWriter, redisCacheConfiguration, 30000, 120000);
}
```

比如这里设置了一个随机过期时间为30-120s的缓存配置。

## 对比优化

```java
@Bean
public RedisCacheManager redisCacheManager(RedisConnectionFactory factory){
    return RedisCacheManager.builder(factory)
        .withCacheConfiguration("user", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(2)))
        .build();
}
```

这是之前的RedisCacheManager，可以发现是可以设置缓存名称的，对应@Cacheable中的value属性，也就是说不同的value可以设置不同的配置。

但是新创建的RandomTtlRedisCacheManager只能设置一个基础配置，如何解决？

继续看源码。

### RedisCacheManager

```java
public class RedisCacheManager extends AbstractTransactionSupportingCacheManager {
    private final RedisCacheWriter cacheWriter;
    private final RedisCacheConfiguration defaultCacheConfig;
    private final Map<String, RedisCacheConfiguration> initialCacheConfiguration;
    private final boolean allowInFlightCacheCreation;
    
    public RedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration, Map<String, RedisCacheConfiguration> initialCacheConfigurations, boolean allowInFlightCacheCreation) {
        this(cacheWriter, defaultCacheConfiguration, allowInFlightCacheCreation);
        Assert.notNull(initialCacheConfigurations, "InitialCacheConfigurations must not be null!");
        this.initialCacheConfiguration.putAll(initialCacheConfigurations);
    }
    
    protected Collection<RedisCache> loadCaches() {
        List<RedisCache> caches = new LinkedList();
        Iterator var2 = this.initialCacheConfiguration.entrySet().iterator();

        while(var2.hasNext()) {
            Map.Entry<String, RedisCacheConfiguration> entry = (Map.Entry)var2.next();
            caches.add(this.createRedisCache((String)entry.getKey(), (RedisCacheConfiguration)entry.getValue()));
        }

        return caches;
    }
    
    // builder
    public static class RedisCacheManagerBuilder {
        @Nullable
        private RedisCacheWriter cacheWriter;
        private RedisCacheConfiguration defaultCacheConfiguration;
        private final Map<String, RedisCacheConfiguration> initialCaches;
        private boolean enableTransactions;
        boolean allowInFlightCacheCreation;
        public RedisCacheManager build() {
            // Assert断言省略...
            RedisCacheManager cm = new RedisCacheManager(this.cacheWriter, this.defaultCacheConfiguration, this.initialCaches, this.allowInFlightCacheCreation);
            cm.setTransactionAware(this.enableTransactions);
            return cm;
        }
    }
}
```

回到RedisCacheManager，builder创建了一个叫initialCaches的Map，在build后执行RedisCacheManager的构造方法，将其值传入initialCacheConfiguration。

而initialCacheConfiguration在loadCaches方法中被使用，针对每个名称和配置，均创建一个独立的cache，然后返回一个cache集合。

### 最终方案

因此，可以在RandomTtlRedisCacheManager继续重写loadCaches方法，并做优化。

```java
public class RandomTtlRedisCacheManager extends RedisCacheManager{
    private final RedisCacheWriter cacheWriter;
    private final RedisCacheConfiguration defaultCacheConfig;
    private final int minTtl;
    private final int maxTtl;
    private final Map<String, int[]> ttlConfigs;

    public RandomTtlRedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration, int minTtl, int maxTtl, Map<String, int[]> ttlConfigs) {
        super(cacheWriter, defaultCacheConfiguration);
        this.cacheWriter = cacheWriter;
        this.defaultCacheConfig = defaultCacheConfiguration;
        this.minTtl = minTtl;
        this.maxTtl = maxTtl;
        this.ttlConfigs = ttlConfigs;
    }

    @Override
    protected RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
        return new RandomTtlRedisCache(name, this.cacheWriter, cacheConfig != null ? cacheConfig : this.defaultCacheConfig, minTtl, maxTtl);
    }

    @Override
    protected Collection<RedisCache> loadCaches() {
        List<RedisCache> caches = new LinkedList<>();
        ttlConfigs.forEach((name, ttlArr) -> caches.add(new RandomTtlRedisCache(name, cacheWriter, defaultCacheConfig, ttlArr[0], ttlArr[1])));
        return caches;
    }
}
```

通过构造方法传入ttlConfigs，key为cache的名称，value为大小为2的int数组，对应最小最大随机时间，loadCaches遍历map，返回cache集合。

```java
@Bean
public RandomTtlRedisCacheManager redisCacheManager(RedisConnectionFactory factory){
    RedisCacheWriter cacheWriter = RedisCacheWriter.lockingRedisCacheWriter(factory);
    RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();
    // userInfo 缓存设置15 - 20分钟
    // article 缓存设置30 - 40分钟
    HashMap<String, int[]> ttlConfigs = new HashMap<>();
    ttlConfigs.put("userInfo", new int[]{900000, 1200000});
    ttlConfigs.put("article", new int[]{1800000, 2400000});
    return new RandomTtlRedisCacheManager(cacheWriter, redisCacheConfiguration, 30000, 120000, ttlConfigs);
}
```

RandomTtlRedisCacheManager这个Bean对象中，创建缓存ttl映射map，调用RandomTtlRedisCacheManager构造方法。

上图例子：

任何采用了上述redisCacheManage作为CacheManager的缓存注解，

名为userInfo缓存设置15-20分钟随机过期时间，

名为article缓存设置30-40分钟过期随机，

而其他缓存则为30-120s的随机过期时间。



