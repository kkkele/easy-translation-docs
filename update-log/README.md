<style>
    li{
 		 font-size: 18px; /* 设置默认的字体大小 */
	}
</style>
# 更新日志

## v1.4.0@2024-07-01

- pref：拆分了大量的类，将其功能细化成各种组件，增强扩展性。
- pref：增加包扫描的流程，提高项目刚启动时的响应速度。
- pref：默认注册0参到5参的翻译器。
- feat：将`@Mapping`的`mapper`参数改名字为`mappers`，且类型从string数组改为`@Mapper`数组。【**重要**】
- feat：`@Mapper`由`value`来决定映射对象的属性，使用`paramHandler`来改变该`mapper`对象，由此作为参数变换类型，批量处理的基础。【**重要**】
- feat：`@Mapping`增加`strategy`属性，增加批量翻译的能力。【**重要**】
- feat：`@Mapping`增加`resultHandler`属性，增加处理翻译结果的能力，由此作为分发批量翻译时分配翻译结果的基础。【**重要**】
- feat：`@Mapping`增加`groupKey`属性，作为辅助参与结果映射。
- feat：增加`jackson`序列化时翻译的功能。【**重要**】
- refactor：重构Config类写法。
- refactor：优化spring模块写法，使其更加优雅。
- refactor：对大部分包名，类名做了破坏性更新。【**重要**】
- test：demo模块增加大量测试用例，优化测试流程。
- bugfix：修复单例桶不生效的问题。

---

## v1.3.1@2024-06-11

- bugfix：修复`@RefTranslation`对于集合类型翻译失效的问题。

---

## v1.3.0@2024-06-10

- feat：增加对PrototypeScopeTranslator（即原型Bean生产Transaltor）的支持。**【重要】**
- refactor：为适配原型Bean生产Translator，将原先Context需要手动注册每个对象的操作（注册完即生成Translator对象，是单例的），交由InvokeBeanFactory，**方便对接各种框架。**框架提供了默认实现类，也可以参考spring-boot-starter下的实现，搭配core模块对接其他框架。【重要】
- feat：扩展`@Translator`参数。现在，开发者可以在接口中直接声明一个翻译器，并由指定beanName的Bean来实现，和决定这是个单例还是原型Translator。**【重要】**
- feat：增加`@RefTranslation注解`来实现关联翻译功能，现在，您可以在模型类中直接做一个嵌套操作（用法类似于`@TranslationExecute`），具体请看文档。**【重要】**
- refactor：更改底层结构，强化各个类的职责。
- bugfix：修复当翻译对象为空的时候，导致的空指针异常。
- pref：优化处理模型类时空指针异常的报错提示。

---

## v1.2.2@2024-05-19

- feat：增加插件性能分析器。
- feat：增加插件翻译回调注册器。
- refacotr：提前实例化translator，考虑后续增加原型bean调用动态方法。
- fix：插入TranslatorPostProcessor导致的空指针异常。
- fix：修复slf4j引起的日志依赖冲突，框架去除slf4j相关依赖，交由开发者自行实现。
- fix：增加对应的报错提示。

---

## v1.1.1@2024-05-18

- feat：增加other字段自动转为对应方法参数类型。
- refactor：重构TranslatorDefinitionReader写法，将其功能拆分为两个对象去完成。
- bugfix：重写Reflections框架，为其增加扫描组合注解的能力。
- bugfix：修复代理类的Bean翻译器无法找到的问题。
- refactor：小重构DefaultTransExecutorContext类。

---

## v1.1.0@2024-05-17

- refactor：更改easy-tranlsation-spring-boot-start命名为easy-tranlsation-spring-boot-start。
- feat：easy-translation-spring-boot3-starter正式发布。
- feat：增加空指针异常处理器，使得开发者可以灵活的处理映射过程中空指针的情况。**【重要】**
- refactor：贯彻组合优于继承的思想，将获取属性值的过程交给对象来做，使得开发者可以自由决定功能是如何实现的。**【重要】**
- refactor：默认采用MethodHandle来获取属性，而非反射，使得获取属性的性能有5倍左右的提升。
- refacotr：更改超时时间的单位为毫秒。

---

## v1.0.0 @2024-05-15

- 第一个版本正式发布！！！！

