# 翻译器回调执行

> 本页面展示的为SpringBoot环境下是如何使用的，在非SpringBoot请看其他文档

如果开发者希望在翻译器执行完后，调用一个回调方法，再次处理翻译结果或者是做其他诸如日志之类的动作。

你可以引入该插件。

## Maven

**【重要】**如果开发者使用了easy-translation-spring-boot-starter或者easy-translation-spring-boot3-starter，则无需引入，因为已经继承了相关依赖。

```xml
<dependency>
    <groupId>io.github.kkkele</groupId>
    <artifactId>easy-translation-execute-callback</artifactId>
    <version>{{version}}</version> <!--请自行替换为最新版本-->
</dependency>
```

## gradle

**【重要】**如果开发者使用了easy-translation-spring-boot-starter或者easy-translation-spring-boot3-starter，则无需引入，因为已经继承了相关依赖。

```gradle
implementation group: 'io.github.kkkele', name: 'easy-translation-execute-callback', version: ${last.version}
```

## TranslationCallBack

```java
public interface TranslationCallBack<T> {

    /**
     * 匹配翻译器名称，当匹配到时，为该翻译器执行增加回调
     *
     * @return 正则表达式
     */
    String match();

    /**
     * 翻译成功回调
     * @param result 翻译结果
     */
    void onSuccess(T result);
}

```

开发者需要实现`TranslationCallBack`接口，并交由Spring容器管理。

`TranslationCallBack`的`onSuccess`需要开发者实现一个回调方法，对翻译的结果进行处理。

而`TranslationCallBack`的`match`需要开发者提供一个正则表达式，匹配该正则表达式的翻译器将会在创建阶段添加该回调方法。

举个例子

```java
    @Component
    public  class PrintCallback implements TranslationCallBack<Pair<String,String>> {

        @Override
        public String match() {
            return "getPair";
        }

        @Override
        public void onSuccess(Pair<String, String> result) {
            System.out.println(result);
        }

    }
```

它将在getPair翻译器执行后，打印结果。
