# 生成翻译器

> 本页面展示的为SpringBoot环境下是如何使用的，在非SpringBoot请看其他文档
>
> 如果您使用的是<font size=4>**spring-boot**</font>版本，那么，框架已经为您提供了一个组合注解`@Translator`，来省略掉使用注解时还要拼写name的麻烦。

## 核心注解

> 所有的注册成为翻译器的方法或者枚举都要使用`@Translation`**注解或标记了该注解的组合注解**
>
> 您也可以自定义其他的组合注解，只要您标记了`@Translation`即可。(下文只说`@Translation`，但请读者不要忽略其组合注解)

```java
/**
 * 用来标记成为翻译器的静态方法，动态方法，枚举类
 */
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Translation {

    /**
     * 翻译器名称
     */
    String name() default "";

    /**
     * 实现的bean名称
     */
    String invokeBeanName() default "";

    /**
     * invokeBeanName解析器
     */
    Class<? extends BeanNameResolver> beanNameResolver() default DefaultBeanNameResolver.class;

    /**
     * 原型Bean还是单例Bean
     */
    InvokeBeanScope scope() default InvokeBeanScope.SINGLETON;
}
```

### name

name属性用于标记一个翻译器名称，开发者可以在DefaultTranslatorContext中使用name获取对应的翻译器。

如果被标记的方法是动态方法的话，开发者可以使用到其他属性。

### invokeBeanName

invokeBeanName，用于指定激活翻译器的bean。

开发者可以搭配`beanNameResolver`来自定义实现对beanName的解析。

### beanNameResolver

用于解析`invokeBeanName`属性，框架提供的默认解析器并不做解析。

### scope

标记该Translator是单例还是原型的，如果指定scope为原型Bean，那么，每次调用该翻译器进行翻译的时候，都会重新使用getBean来获取bean对象，然后再生成该Translator对象进行翻译。

如果是单例Translator，即使调用该Translator的Bean的scope为`prototype`，也只会初始化一次，后复用同一个对象。

## 方法翻译器

### 对象动态翻译器

`EasyTranslation`会自动装载在可以扫描的包下的被`@Translation`标记的动态方法。

```java
//标记在接口的方法上也可以，如果bean有多个实现类请指定BeanName，否则会随机选取一个实现类当作翻译器的实现
public interface ShopService {

    boolean checkExist(Integer shopId);

    @Translator(value = "id_to_shop_name",invokeBeanName = "shopServiceImpl")
    String getShopName(Integer shopId);
}

//推荐直接标记在实现类上
@Service
@RequiredArgsConstructor
public class ShopServiceImpl implements ShopService {

    private final ShopMapper shopMapper;

    @Override
    public boolean checkExist(Integer shopId) {
        return shopMapper.selectById(shopId) != null;
    }

    @Override
    @Idempotent
    @Translator(value = "id_to_shop_name")
    public String getShopName(Integer shopId) {
        return shopMapper.selectById(shopId).getShopName();
    }
}

```

### 静态翻译器

`EasyTranslation`会自动装载在可以扫描的包下的被`@Translation`标记的静态方法，并注册到容器中。

```java
    @Translation(name = "getTime")
    public static Date getDate() {
        return DateUtil.date();
    }
```

这个类并不需要被Spring容器所管理。

## 多参翻译

您或许在上文注意到，在[静态翻译器](#静态翻译器)中引用的例子里，方法没有任何的参数。

这便是`EasyTranslation`的强大之处，它允许您将任意参数的方法转化为翻译器，并且提供及其灵活的判断条件。

首先记住两个概念，一个是mapper，一个other

### mappers

mapper意为映射，它会将您的mapper字符串转化成对象的对应属性。

```java
    private Integer typeId;

    @Mapping(translator = "getTypeById", mappers = @Mapper("typeId"), receive = "typeName")
    private String typeName;
```

例如，上述类的在翻译typeName字段时，将使用typeId属性，然后在处理后传递给翻译器。

### other

other为辅助判断条件，它将在翻译执行翻译时，尝试变成对应的参数类型并传递给翻译器执行。

### @TransMapper

标记了该注解的方法参数，将会成为**mapper**字段。

**需要注意**的是，为了简便开发，当您的方法参数没有添加任何`@TransMapper`的参数且没有任何添加了`@TransOther`的字段，且参数个数大于0时，框架会自动将**第一个参数**标记为**mapper**字段。

所以，之前的例子中，没有添加`@TransMapper`注解依然可以通过**mapper**进行一个翻译填充。

### @TransOther

标记了该注解的方法参数，将会成为**other**字段。

当方法参数上存在`@TransMapper`或者`@TransOther`时，所有不标记的参数也将成为other字段。

**需要注意**的是，为了简便开发，当您的方法参数没有添加任何`@TransMapper`的参数且没有任何添加了`@TransOther`的字段，且参数个数大于0时，框架会自动将**第一个参数**标记为**mapper**字段。**其余字段全为other字段。**

所以，之前的例子中，没有添加`@TransMapper`注解依然可以通过**mapper**进行一个翻译填充。

**【重要】**mapper对应的属性值将按顺序依次传入`@TransMapper`标记的参数，然后other也将按顺序依次填补剩下的方法参数。

来看几个例子

```java
	@Translator("demo_getName")
    public static String getName() {
        return "小明";
    }

    @Translator("demo_getName1")
    public static String getName1(String name) {
        return "超级" + name;
    }

    @Translator("demo_getNameByCondition2")
    public static String getNameByCondition2(@TransOther String name) {
        return "超级" + name;
    }

    @Translator("demo_getName2")
    public static String getName2(String name, Integer sort) {
        return "天下第" + sort + "! => " + name;
    }

    @Translator("demo_getName3")
    public static String getName3(String name, String professional, Integer sort) {
        return "天下第" + sort + professional + "! => " + name;
    }

    @Translator("demo_getName4")
    public static String getName4(String professional, @TransMapper String name, Integer sort) {
        return "天下第" + sort + professional + "! => " + name;
    }

    @Translator("demo_getName5")
    public static String getName5(String professional, @TransMapper String name, @TransMapper Integer sort) {
        return "天下第" + sort + professional + "! => " + name;
    }
```

对应的`@Mapping`写法分别为

```java
    @Mapping(translator = "demo_getName", sort = -1)
    private String var0; //String getName();

    @Mapping(translator = "demo_getName1", mappers = @Mapper("var0"))
    private String var1; //String getName1(String name);

    @Mapping(translator = "demo_getNameByCondition2", other = {"小红","小绿","小青"})
    private String var2; //String getNameByCondition2(@TransOther String name);

    @Mapping(translator = "demo_getName2", mappers = @Mapper("var0"), other = "1")
    private String var3; //String getName2(String name, Integer sort);

    @Mapping(translator = "demo_getName3", mappers = @Mapper("var0"), other = {"程序员", "1"})
    private String var4; //String getName3(String name, String professional, Integer sort);

    @Mapping(translator = "demo_getName4", mappers = @Mapper("var0"), other = {"程序员", "1"})
    private String var5; //String getName4(String professional, @TransMapper String name, Integer sort);

    private Integer sort = 1;

    @Mapping(translator = "demo_getName5", mappers = @Mapper({"var0", "sort"}), other = "程序员")
    private String var6; // String getName5(String professional, @TransMapper String name, @TransMapper Integer sort);
```

如果您的方法参数过多，导致了项目不能成功启动，首先要做一些额外的配置。

## 应用启动失败？请扩充翻译器类型

我们首先实现这样一个`interface`，您需要几个参数，方法就写几个参数，为了避免其他麻烦，请你覆盖Translator的doTranslate接口，并调用自身的方法。

且将方法的参数和返回都配置为Object类型。

```java
public interface ThreeParamTranslator extends Translator {

    Object translate(Object var0, Object var1, Object var2);

    @Override
    default Object doTranslate(Object... args) {
        return translate(args[0], args[1], args[2]);
    }
}

```

然后，我们需要实现一个`TranslationAutoConfigurationCustomizer` Bean，在该Bean中对config进行注册，将这个新的翻译器类型注册进去。

```java
@Bean
public TranslationAutoConfigurationCustomizer translationAutoConfigurationCustomizer() {
    return config -> {
        config.registerTranslatorClazz(ThreeParamTranslator.class);
    };
}
```

这样，我们的三参方法就会被处理成对应的三参翻译器。项目就可以顺利启动了。

框架默认提供了**0到5参的翻译器**以解决最普遍的需求。

## 枚举翻译器

枚举翻译器限定为一参的，使用方法如下

在枚举类上标记`@Translation`，然后使用`@TransMapper`标记mapper字段，使用`@TransValue`标记翻译字段，框架将自动为您生成一个 **一参翻译器**。

所以，枚举不支持多参，不支持other条件，复杂的枚举只会导致编程的困难，所以，作者认为只有这样可以满足大部分需求。

```java
@Translator("getStatus")
public enum Status {

    UNKNOWN(0, "未知"),

    NORMAL(1, "正常"),

    DELETED(2, "删除");

    @TransMapper
    private int code;

    @TransValue
    private String desc;

    Status(int code, String desc)
    {
        this.code = code;
        this.desc = desc;
    }

    public int getCode()
    {
        return code;
    }

    public String getDesc()
    {
        return desc;
   }
}
```

