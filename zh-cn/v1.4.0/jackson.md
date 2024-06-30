# Jackson序列化时翻译

> 因为Jackson序列化时会对每个节点都进行处理，所以为了防止重复处理的情况，这里将不再能配合`@RefTranslation`注解

## 开启配置

application.yml中开启配置

```yml
easy-translation:
  json-serialize: true #默认为true
```

即可在进行Jackson序列化时自动执行翻译，不再需要`@TranslationExecute注解`

不过，需要注意的是，开发者需要指定`@Mapping`的`timing`字段为JSON_SERIALIZE

```java
@Data
public class JsonVo {

    private Integer id = 0;

    private Integer cardType;

    @Mapping(translator = "getCardTypeNames", strategy = MappingStrategy.BATCH,timing = TranslateTiming.JSON_SERIALIZE,mappers = @Mapper("cardType"))
    private String cardTypeName;

    @Translator("getCardTypeNames")
    public static List<String> getCardTypeNames(List<Integer> cardTypes) {
        return cardTypes.stream()
                .map(cardType -> {
                    switch (cardType) {
                        case 1:
                            return "A";
                        case 2:
                            return "B";
                        case 3:
                            return "C";
                        default:
                            return "D";
                    }
                })
                .collect(Collectors.toList());
    }
}

```

如上述代码所示

## @JsonMapping

因为框架支持组合注解，所以为开发者简单封装了一个注解，不再需要重复的写timing属性。

```java
@Inherited
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Mapping(timing = TranslateTiming.JSON_SERIALIZE)
public @interface JsonMapping {

    /**
     * 翻译器名称
     */
    String translator() default "";

    /**
     * 接收的属性内容
     */
    String receive() default "";

    /**
     * 辅助条件，将值修改成对应类型后直接传递给翻译器
     */
    String[] other() default {};

    /**
     * 映射策略，是单个处理还是批量处理
     */
    MappingStrategy strategy() default MappingStrategy.SINGLE;

    /**
     * 替代原先的mapper字段
     * 后续将逐步替换掉mapper字段的使用
     * @since 1.4.0
     * @return
     */
    Mapper[] mappers() default {};

    /**
     * <p>分组依据
     * <p>辅助结果处理器进行结果筛选
     * <p>具体用法，可以查考默认结果处理器
     * @see com.superkele.translation.core.mapping.ResultHandler
     * @see com.superkele.translation.core.mapping.support.DefaultResultHandler
     */
    String[] groupKey() default {};

    /**
     * 结果处理器
     * @see com.superkele.translation.core.mapping.ResultHandler
     */
    String resultHandler() default "";

    /**
     * 当不为null时，是否也映射
     */
    boolean notNullMapping() default false;

    /**
     * 控制同步任务的执行顺序
     */
    int sort() default 0;

    /**
     * 是否异步执行;
     * 开启后仍然遵循sort排序，需要等待sort低的批次同步翻译字段全部执行完毕才开始翻译
     */
    boolean async() default false;

    /**
     * 在该字段翻译执行后再开始翻译，主要用于精细化控制异步翻译时的执行顺序
     * 当该字段生效时，无视sort的执行顺序
     * 该字段会使得field在前置事件执行完然后回调进行翻译，所以即时您将async设为false,也存在不在主线程中运行的情况
     * 这主要取决于最后触发该事件的翻译字段在哪个线程中
     */
    String[] after() default {};


    /**
     * 当翻译时，属性为空导致了空指针异常的解决方案
     */
    Class<? extends NullPointerExceptionHandler> nullPointerHandler() default DefaultNullPointerExceptionHandler.class;
}

```

使用案例。

```java
@Data
public class Person {

    private Integer personId;

    private String personName;

    private Integer sex = new Random().nextInt(1) + 1;

    @JsonMapping(translator = "getSexCode", mappers = @Mapper("sex"))
    private String sexCode;

    private Integer deptId = new Random().nextInt(10);

    @JsonMapping(translator = "getDeptById", mappers = @Mapper("deptId"))
    private Dept dept;

    @Translator("getSexCode")
    public static String getSexCode(Integer sex) {
        return sex == 1 ? "男" : "女";
    }

    @Translator("getPersonById")
    public static Person getPersonById(Integer personId) {
        Person person = new Person();
        person.setPersonId(personId);
        person.setPersonName("personName" + personId);
        person.setDeptId(personId);
        person.setDept(Dept.getDeptById(personId));
        return person;
    }
}

```

