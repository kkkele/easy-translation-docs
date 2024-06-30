# 快速启动

> 如果您想找到示例中对应的代码，可以前往[Gitee](https://gitee.com/cai-zhiyuDaKeLe/easy-translation) | [Github](https://github.com/kkkele/easy-translation) 的demo模块查看
>
> 本页面展示的为SpringBoot环境下是如何使用的，在非SpringBoot请看其他文档

## 启用Easy-Translation

> 于v1.4.0后已经启用注解启动的方式，之前采用这种方式是希望扫描包的时候更加简单
>
> 1.4.0后作者决定改为扫描springboot启动类下的包名和properties中开发者指定的包名。

~~启动类或任意可以成为**Bean**的类上（例如：`@Component`）增加注解`@EnableTranslation`~~

```java
@SpringBootApplication
@EnableTranslation
public class EasyTranslationApplication {
    public static void main(String[] args) {
        SpringApplication.run(EasyTranslationApplication.class, args);
    }
}

```

~~**重要**的一点是，`@EnableTranslation`可以传递包路径~~

```java
@EnableTranslation({"com.superkele","io.github"})
```

~~如上图代码所示，`Easy-Translation`将会自动扫描“com.superkele"，"io.github"包及**添加了该注解的类的子包**。~~

~~扫描包是为了将路径下当作翻译器的**方法**和**枚举**获取，从而将他们注册转化成为翻译器（框架中，翻译器是一个具体的类）。~~

~~java中，调用一个动态方法是一个对象的动作。所以，当将一个动态方法转化为翻译器的时候，需要额外提供激活翻译器的对象。这个对象的提供将交由InvokeBeanFacotry去实现。~~

~~在SpringBoot环境下，框架已自动帮开发者实现了基于SpringApplicationContext的InvokeBeanFactory。~~

~~所以，**开发者可以无需关心如何注册激活对象。**~~

~~开发者可以手动指定激活翻译器的Bean的beanName，当开发者不指定时，框架将自动寻找符合该类型的Bean激活，所以，大部分场景下，开发者无需做其他复杂的配置。~~

~~需要注意的是，**静态方法和枚举并不需要对象来调用**，所以无需指定Bean，且**他们一定是单例的翻译器。**~~

------

application.yml中增加如下配置

```yml
easy-translation:
  enable: true #默认为true
  base-package:
    domain:
      - com.superkele #提前装载要翻译的类信息，即使不指定也可以正常翻译
    translator:
      - com.superkele.demo #扫描translator所在的包，如果有在非启动类子目录的下的翻译器，需要配置此项
```

当 easy-translation.enable = true时，框架将开始工作

## 注册翻译器

> 【**重要】**详细用法请看核心功能，这里只展示部分功能，帮助您快速上手

这里给出完整代码

```java
@Data
@Accessors(chain = true)
public class User {
    private Integer id;
    private String name;
    private Integer age;
}

@Service
@Slf4j
public class AppUserService implements UserService {

    /**
	* 动态翻译器，使用对象激活，所以如果使用了增强（代理）的对象，方法将得到增强
	*/
    @Translation(name = "produceRandomUser")
    public static User produceRandomUser(Integer id) {
        return new User()
                .setId(id)
                .setName(UUID.randomUUID().toString().substring(1, 6))
                .setAge(new Random().nextInt(100));
    }

    /**
	* 静态翻译器，由类调用，不需要对象激活
	*/
    @Translation(name = "getUserByIdSingleton")
    public User getUserByIdSingleton(Integer id) {
        return new User()
                .setId(id)
                .setName(UUID.randomUUID().toString().substring(1, 6))
                .setAge(new Random().nextInt(100));
    }
    
}

/**
* 枚举翻译器
*/
@Translation(name = "getStatus")
@Getter
public enum Status {
    NORMAL(0, "正常"),

    DISABLED(1, "禁用");


    @TransMapper
    private final Integer code;

    @TransValue
    private final String desc;

    Status(Integer code, String desc) {
        this.code = code;
        this.desc = desc;
    }
}

```

## 设置需要翻译的字段属性

> 【**重要】**详细用法请看核心功能，这里只展示部分功能，帮助您快速上手

```java
@Data
public class ProductVo {

    private Integer productId;

    private String productName;

    private Integer typeId;

    @Mapping(translator = "getTypeById", mappers = @Mapper("typeId"), receive = "typeName")
    private String typeName;

}


@Service
public class TypeService {

    @Translator(value = "getTypeById")
    public Type getTypeById(Integer id) {
        switch (id) {
            case 1:
                return new Type(1, "体育器械");
            case 2:
                return new Type(2, "食品");
            case 3:
                return new Type(3, "书籍");
            default:
                return null;
        }
    }

}

```

我们只需要在需要翻译然后自动填充的字段上使用`@Mapping`注解指定翻译器，即可使用将该类变为可被翻译的类。

**mappers = @Mapper("typeId")** 标明，该类翻译时将以对象的**typeId**属性为参数，然后经过`@Mapper`中指定的`paramHandler`处理后，传递给翻译器。

```java
/**
 * 本注解 @Mapper 将以value作为属性名，逐一获取对象的属性
 * 然后使用paramHandler包装成参数，传递给translator，然后执行翻译
 */
@Inherited
@Target(ElementType.TYPE_PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Mapper {

    String[] value() default "";

    String paramHandler() default "";
}
```

需要注意的是，`@Mapping`的`mappers`参数和`@Mapper`的`value`参数都为数组

为的是开发者可以方便的指定，某一类mapper参数都用同一个paramHandler

```java
    @Mapping(translator = "getDictValues",
            mappers = {
                    @Mapper({"dictType", "dictCode"})
            },
            receive = "dictValue",
            strategy = MappingStrategy.BATCH)
    private String dictValue;
```

## 在需要翻译的方法上添加注解

> 【**重要】**详细用法请看核心功能，这里只展示部分功能，帮助您快速上手

在需要填充的方法上添加注解`@TranslationExecute`，即可开启翻译功能

```java
    @Override
    @TranslationExecute
    public ProductVoV2 getDetailByIdV2(Integer id) {
        Product byId = getById(id);
        ProductVoV2 productVO = new ProductVoV2();
        productVO.setProduct(byId);
        return productVO;
    }
```

值得一提的是，您可以将该注解用在所有返回不为void的方法上，只要您**指定了需要被翻译的属性**即可，例如，可以在controller中处理。

```java
    @GetMapping("/{id}")
    @TranslationExecute(field = "data")
    public R<ProductVo> getDetailById(@PathVariable Integer id) {
        return R.ok(productService.getDetailById(id));
    }

@Data
public class R<T> {

    int code;

    T data;

    public R(int code, T data) {
        this.code = code;
        this.data = data;
    }

    public static <T> R<T> ok(T data) {
        return new R<>(200, data);
    }

    public static <T> R<T> error(T data) {
        return new R<>(500, data);
    }

    public static <T> R<T> error() {
        return new R<>(500, null);
    }

    public static <T> R<T> ok() {
        return new R<>(200, null);
    }
}

```

对于`List`，`array`，`map`等，`EasyTranslation`会对其进行拆包，然后执行翻译，例如

```java
    @GetMapping("/list")
    @TranslationExecute(field = "data")
    public R<List<ProductVo>> getList() {
        return R.ok(mappingToList(ListUtil.of(1, 2, 3, 4, 4)));
    }

    @GetMapping("/array")
    @TranslationExecute(field = "data")
    public R<ProductVo[]> getArray() {
        return R.ok(mappingToArray(1, 2, 3, 4, 5));
    }

    @GetMapping("/map")
    @TranslationExecute(field = "data")
    public R<Map<Integer, ProductVo>> getMap() {
        return R.ok(mappingToMap(1, 2, 3, 4, 5));
    }
```

如果您希望在 树结构，链表节点等嵌套结构，List<List<Object>> 或是类似二维数组等复杂结构上进行逐个拆包翻译，你可以自行实现

`UnpackingHandler`接口，然后在`@TranslationExecute`上指定unpackingHandler属性即可。

```java
    @GetMapping("/array")
    @TranslationExecute(field = "data",unpackingHandler = DefaultUnpackingHandler.class)
    public R<ProductVo[]> getArray() {
        return R.ok(mappingToArray(1, 2, 3, 4, 5));
    }

```

## 启动项目

至此，您可以启动您的SpringBoot应用，检验插件是否正常运行。

其他更多更复杂更强大的用法和配置，您可以在核心功能中进行一个翻阅查看。

如果有任何问题，欢迎联系作者或在[gitee](https://gitee.com/cai-zhiyuDaKeLe/easy-translation)，[github](https://github.com/kkkele/easy-translation)上提出您宝贵的issue。

![mmqrcode1715662117475](assets/mmqrcode1715662117475.png)