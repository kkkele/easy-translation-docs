# 安装使用

> 因不同v1.3.0的底层架构发生变化，请开发者一定要同一插件和核心模块的版本，推荐使用v1.3.0以上版本

## 核心模块

> maven，推荐使用1.3.1以上版本

- spring-boot 3以下的版本

  ```xml
  <dependency>
      <groupId>io.github.kkkele</groupId>
      <artifactId>easy-translation-spring-boot-starter</artifactId>
      <version>1.3.1</version>
  </dependency>
  ```

- spring-boot3 以上的版本

  ```xml
  <dependency>
      <groupId>io.github.kkkele</groupId>
      <artifactId>easy-translation-spring-boot3-starter</artifactId>
      <version>1.3.1</version>
  </dependency>
  ```

- 非spring-boot环境

  需要手动去加载一些全局配置

  ```xml
  <dependency>
      <groupId>io.github.kkkele</groupId>
      <artifactId>easy-translation-core</artifactId>
      <version>1.3.1</version>
  </dependency>
  ```


> gradle，推荐使用1.3.1以上版本
>
> 

- spring-boot 3以下的版本

  ```gradle
  implementation group: 'io.github.kkkele', name: 'easy-translation-spring-boot-starter', version: '1.3.0'
  ```

- spring-boot3 以上的版本 

  ```gradle
  implementation group: 'io.github.kkkele', name: 'easy-translation-spring-boot3-starter', version: '1.3.0'
  ```

- 非spring-boot环境

  ```gradle
  implementation group: 'io.github.kkkele', name: 'easy-translation-core', version: '1.3.0'
  ```

  

## 插件

### 性能分析器

```xml
<dependency>
    <groupId>io.github.kkkele</groupId>
    <artifactId>easy-translation-perf-record</artifactId>
    <version>1.3.0</version>
</dependency>
```

```gradle
implementation group: 'io.github.kkkele', name: 'easy-translation-perf-record', version: '1.3.0'
```

### 翻译回调器

已在spring模块内置

```xml
<dependency>
    <groupId>io.github.kkkele</groupId>
    <artifactId>easy-translation-execute-callback</artifactId>
    <version>1.3.0</version>
</dependency>
```

```gradle
implementation group: 'io.github.kkkele', name: 'easy-translation-execute-callback', version: '1.3.0'
```

