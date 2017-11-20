---
layout: post
title: Spring Boot 多Redis源配置简洁方案
date: 2017-11-19 21:09
tags: [Redis]
---

使用spring boot开发时，由于某些业务的复杂些，需同时与多个redis进行交互，在此之前需要初始化多个redis配置，初始化的过程大同小异，基本上都是相同的代码，初始化完成后，直接使用redis实例或者将redis实例注入到spring bean容器中，供需要的服务注入使用。

为了让开发更加便捷（Don’t Repeat Yourself），我的想法是在spring容器初始化过程中，将多个redis初始化并注入到容器里。

具体思路是通过spring提供的两个接口[```EnvironmentAware```](https://docs.spring.io/spring-boot/docs/1.5.8.RELEASE/reference/htmlsingle/#cloud-deployment-cloud-foundry-services)、
```BeanDefinitionRegistryPostProcessor```，在容器实例化bean的过程中，读取项目中所有redis
的配置，并将其实例注入到容器里。

为了方便的读取所有redis配置，redis的配置设计如下（properties配置格式也可以）：
    {% highlight yml %}
    redis:
      config-set:
        redis1: 127.0.0.1:6379
        redis2: 127.0.0.2:6379
    {% endhighlight %}

其中redis、config-set两个节点是固定的，通过这两个固定的节点解释到所有的redis配置，
"redis1"、"redis2"是两个redis实例在spring容器中的名字即Bean Name，在服务中配合@Qualifier注入所需的redis实例。

解析redis配置的代码片段如下：
    {% highlight java %}
    @Override
    public void setEnvironment(Environment environment) {
        RelaxedPropertyResolver propertyResolver = new RelaxedPropertyResolver(environment, "redis.");
        redisConfigMap = propertyResolver.getSubProperties("config-set.");
    }
    {% endhighlight %}


构建redis实例并将其注入到spring容器中的代码片段如下：
    {% highlight java %}
    private JedisPoolConfig jedisPoolConfig() {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMaxTotal(1000);
        jedisPoolConfig.setMaxIdle(500);
        jedisPoolConfig.setTestOnBorrow(false);
        jedisPoolConfig.setTestOnReturn(false);
        jedisPoolConfig.setTestWhileIdle(true);
        return jedisPoolConfig;
    }
    
     @Override
        public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
            redisConfigMap.forEach((beanName, host) -> {
                if (beanName == null || beanName.isEmpty()) {
                    throw new NullPointerException("initialize redis configuration failed because " +
                            "redis name can not be empty.");
                }
                if (host == null) {
                    throw new NullPointerException("initialize redis configuration failed because " +
                            "host can not be empty.");
                }
                logger.info("initialize redis({}) config ...", beanName);
    
                String ip = ((String) host).split(":")[0];
                int port = Integer.parseInt(((String) host).split(":")[1]);
                GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
                beanDefinition.setBeanClass(JedisPool.class);
                beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, this.jedisPoolConfig());
                beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(1, ip);
                beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(2, port);
                beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(3, 5000); //timeout为5秒
                beanDefinitionRegistry.registerBeanDefinition(beanName, beanDefinition);
            });
        }
    {% endhighlight %}

Spring Boot 多Redis源配置 [Demo](http://)
