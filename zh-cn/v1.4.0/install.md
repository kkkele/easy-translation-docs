# 安装使用

> 因不同v1.3.0的底层架构发生变化，请开发者一定要统一插件和核心模块的版本，推荐使用v1.3.0以上版本

## 核心模块

> maven，推荐使用{{version}}以上版本

- spring-boot 3以下的版本

  ```xml
  <dependency>
      <groupId>io.github.kkkele</groupId>
      <artifactId>easy-translation-spring-boot-starter</artifactId>
      <version>{{version}}</version>
  </dependency>
  ```

- spring-boot3 以上的版本

  ```xml
  <dependency>
      <groupId>io.github.kkkele</groupId>
      <artifactId>easy-translation-spring-boot3-starter</artifactId>
      <version>{{version}}</version>
  </dependency>
  ```

- 非spring-boot环境

  需要手动去加载一些全局配置

  ```xml
  <dependency>
      <groupId>io.github.kkkele</groupId>
      <artifactId>easy-translation-core</artifactId>
      <version>{{version}}</version>
  </dependency>
  ```


> gradle，推荐使用{{version}}以上版本
>
> 

- spring-boot 3以下的版本

  ```gradle
  implementation group: 'io.github.kkkele', name: 'easy-translation-spring-boot-starter', version: '{{version}}'
  ```

- spring-boot3 以上的版本 

  ```gradle
  implementation group: 'io.github.kkkele', name: 'easy-translation-spring-boot3-starter', version: '{{version}}'
  ```

- 非spring-boot环境

  ```gradle
  implementation group: 'io.github.kkkele', name: 'easy-translation-core', version: '{{version}}'
  ```

  

## 插件

### 性能分析器

```xml
<dependency>
    <groupId>io.github.kkkele</groupId>
    <artifactId>easy-translation-perf-record</artifactId>
    <version>{{version}}</version>
</dependency>
```

```gradle
implementation group: 'io.github.kkkele', name: 'easy-translation-perf-record', version: '{{version}}'
```

### 翻译回调器

已在spring模块内置

```xml
<dependency>
    <groupId>io.github.kkkele</groupId>
    <artifactId>easy-translation-execute-callback</artifactId>
    <version>{{version}}</version>
</dependency>
```

```gradle
implementation group: 'io.github.kkkele', name: 'easy-translation-execute-callback', version: '{{version}}'
```

