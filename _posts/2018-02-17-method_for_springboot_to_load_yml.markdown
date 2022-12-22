---
layout: post
title: 两种SpringBoot加载YML配置文件的方法
date: 2018-02-17 19:53:02
categories: 原创
---

SpringBoot默认支持properties和YAML两种格式的配置文件。前者格式简单，但是只支持键值对。如果需要表达列表，最好使用YAML格式。SpringBoot支持自动加载约定名称的配置文件，例如`application.yml`。如果是自定义名称的配置文件，就要另找方法了。可惜的是，不像前者有`@PropertySource`这样方便的加载方式，后者的加载必须借助编码逻辑来实现。

# YamlPropertiesFactoryBean

SpringBoot文档中给出了`YamlPropertiesFactoryBean`和`YamlMapFactoryBean`两个类实现YAML配置文件的加载。前者将文件加载为Properties，后者为Map。

在网上搜到了这样一个方法。

```Java
package com.example.demo.config;

import org.springframework.beans.factory.config.YamlPropertiesFactoryBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
import org.springframework.core.io.ClassPathResource;

@Configuration
public class BootstrapConfig {

    @Bean
    public static PropertySourcesPlaceholderConfigurer properties() {
        PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer = new PropertySourcesPlaceholderConfigurer();
        YamlPropertiesFactoryBean yaml = new YamlPropertiesFactoryBean();
        yaml.setResources(new ClassPathResource("config/appprops.yml"));
        propertySourcesPlaceholderConfigurer.setProperties(yaml.getObject());
        return propertySourcesPlaceholderConfigurer;
    }
}

```

# YamlPropertySourceLoader

文档中还提到了`YamlPropertySourceLoader`，但是没有给出实现的代码。今天看着API摸索了一下，找到了方案。

```Java
package com.example.demo.config;

import org.springframework.boot.env.YamlPropertySourceLoader;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;
import org.springframework.core.env.MutablePropertySources;
import org.springframework.core.io.ClassPathResource;

import java.io.IOException;

@Configuration
public class BootstrapConfig {

    @Bean
    public PropertySourcesPlaceholderConfigurer properties2() {
        PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
        YamlPropertySourceLoader loader = new YamlPropertySourceLoader();
        MutablePropertySources sources = new MutablePropertySources();
        try {
            sources.addLast(loader.load("db", new ClassPathResource("config/appprops.yml"), null));   
        } catch (IOException e) {
            e.printStackTrace();
        }
        configurer.setPropertySources(sources);
        return configurer;
    }
}
```

# 简单总结

前者不抛异常，看着更舒服；而后者可以支持同时加载多个YAML文件。

