# 翻译字段自动填充

## 核心注解 

`@Mapping`

> v1.3.0后增加关联翻译功能，它使用`@RefTranslation`来实现相关功能，会对类中的某个属性进行整体翻译，但仍需与`@Mapping`进行配合使用。
>
> 如果开发者只想使用整体翻译功能，那么只要不指定translator属性即可。

**框架支持使用组合注解，开发者可以自行封装@Mapping以提高开发效率。**

```java
/**
 * 用在model类中翻译字段，标记了该注解的字段会与字段被FieldTranslationBuilder解析为FieldTranslationEvent对象
 * 然后 processor会将 FieldTranslationEvent 交给 FieldTranslationHandler处理
 *
 * @see com.superkele.translation.core.metadata.FieldTranslationBuilder
 * @see com.superkele.translation.core.processor.FieldTranslationHandler
 */
@Inherited
@Target({ElementType.FIELD,ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Mapping {

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
     * 执行时机
     */
    TranslateTiming timing() default TranslateTiming.AFTER_RETURN;

    /**
     * 映射策略，是单个处理还是批量处理
     */
    MappingStrategy strategy() default MappingStrategy.SINGLE;

    /**
     * 映射对象的属性值，处理后传递给翻译器
     * 替代原先的mapper字段
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

## 翻译器的选择 

`translator`

`@Mapping`使用translator属性来使用对应的翻译器

例如

```java
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
```

```java
    private Integer typeId;

    @Mapping(translator = "getTypeById", mappers = @Mapper("typeId"), receive = "typeName")
    private String typeName;
```

该 typeName 字段将会调用`Type getTypeById(Integer id)`方法，然后将返回值填充。

## 翻译策略 

`strategy`

```java
/**
 * 映射策略
 */
public enum MappingStrategy {
    /**
     * 单条处理
     */
    SINGLE,
    /**
     * 批量处理
     */
    BATCH,
    /**
     * 自定义处理
     */
    DIY

}
```

### 单体翻译 

`SINGLE`

当指定`@Mapping`的`strategy()`属性为`MappingStrategy.SINGLE`时，将在处理每个对象的时候单独为其执行一次翻译。

```java
@Data
public class DictVo {

    private String dictType;

    private Integer dictCode;

    @Mapping(translator = "getDictValue", mappers = @Mapper({"dictType", "dictCode"}))
    private String dictValue;

    @Translator("getDictValue")
    public static String convertToValue(@TransMapper String dictType, @TransMapper Integer dictCode) {
        switch (dictType) {
            case "sex":
                switch (dictCode) {
                    case 0:
                        return "男";
                    case 1:
                        return "女";
                    default:
                        return "未知";
                }
            case "status":
                switch (dictCode) {
                    case 0:
                        return "正常";
                    case 1:
                        return "封禁";
                    default:
                        return "未知";
                }
        }
        return "未知";
    }
}

```

> 框架有类似事务缓存的概念，如果开发者开启了该缓存，则在一次对象处理中，缓存其翻译器的执行结果，如果调用翻译器的方法参数完全相同，将会复用结果，以减少IO次数。

例如，在上述例子中，如果有一个size为10的DictVo，该翻译器将执行10次。

### 批量翻译 

`BATCH`

当指定`@Mapping`的`strategy()`属性为`MappingStrategy.BATCH`时，当在处理一个集合时，将收集齐所有的mapper参数然后组成一个集合(也可以是数组，Set等其他类型的参数，开发者可以编写paramHandler进行个性化处理)。然后传递给翻译器，**只进行一次翻译器的调用**。

例如：

```java
//翻译类
@Data
public class Order {

    private Integer id;

    @Mapping(translator = "getOrdersByIds",
            strategy = MappingStrategy.BATCH,
            mappers = @Mapper("id"),
            receive = "orderNo")
    private String orderNo;

    @Mapping(translator = "getOrdersByIds",
            strategy = MappingStrategy.BATCH,
            mappers = @Mapper("id"),
            receive = "createTime")
    private String createTime;

}
		//增强的方法
        @TranslationExecute(type = Order.class)
        public List<Order> getMainOrder() {
            return IntStream.range(1, 10)
                    .mapToObj(i -> {
                        Order order = new Order();
                        order.setId(i);
                        return order;
                    })
                    .collect(Collectors.toList());
        }

//测试类
    @Test
    public void test_one_mapper_sync_batch_process() {
        List<Order> mainOrder = service.getMainOrder();
        mainOrder.forEach(order -> {
            Assert.assertNotNull(order.getOrderNo());
            Assert.assertNotNull(order.getCreateTime());
        });
    }
```

在开了debug模式后，打印结果为

![image-20240701025616973](./assets/image-20240701025616973.png)

可以看到，结合翻译器的事务缓存，只执行了一次翻译器调用。

## 结果处理器 

`resultHandler`

使用全类名 或者 @ + `BeanName` 调用

在批量处理时，翻译器的返回结果可能是单个对象，可能是List，Array，Set，Map等多种类型的结果。

结果处理器负责处理翻译结果，然后分发给每个对象。

开发者可以自行更换结果处理器。

```java
/**
 * 结果处理器，将结果映射成我们需要的样子并接收
 *
 * @param <T> 原结果
 * @param <R> 处理后的结果
 * @param <S> 接收时传递给翻译对象的结果
 */
public interface ResultHandler<T, R, S> {


    /**
     * 结果处理
     *
     * @param result   翻译的结果
     * @param groupKey 批量翻译时的分组依据
     * @return
     */
    R handle(T result, String[] groupKey);

    /**
     * 结果选择
     *
     * @param processResult #handle 处理后的结果
     * @param index         对象的索引
     * @param mapperKey     对象的映射key数组 例如@Mapping(mapper={"spuId","createTime"})，则会选取spuId和createTime两个属性的值
     * @return
     */
    S map(R processResult, int index,Object source,Object[] mapperKey);
}

```

### 默认实现

**框架默认的DefaultResultHandler会尝试配合groupKey属性，将结果通过streamApi转化为Map，然后在分发阶段根据mappers属性配对分发。**

```java
    @Translator("getBookList")
    public List<Book> getBookList(List<Integer> ids) {
        return ids.stream()
                .map(id -> {
                    Book book = new Book();
                    book.setId(id);
                    book.setBookName(bookFactory().get(id));
                    return book;
                })
                .collect(Collectors.toList());
    }

    @Translator("getBookArr")
    public Book[] getBookArr(List<Integer> ids) {
        return ids.stream()
                .map(id -> {
                    Book book = new Book();
                    book.setId(id);
                    book.setBookName(bookFactory().get(id));
                    return book;
                })
                .toArray(Book[]::new);
    }

    @Translator("getBookSet")
    public Set<Book> getBookSet(List<Integer> ids) {
        return ids.stream()
                .map(id -> {
                    Book book = new Book();
                    book.setId(id);
                    book.setBookName(bookFactory().get(id));
                    return book;
                })
                .collect(Collectors.toSet());
    }

    @Translator("getBookMap")
    public Map<Integer, Book> getBookMap(List<Integer> ids) {
        return ids.stream()
                .map(id -> {
                    Book book = new Book();
                    book.setId(id);
                    book.setBookName(bookFactory().get(id));
                    return book;
                })
                .distinct()
                .collect(Collectors.toMap(Book::getId, x -> x));
    }


```

对应的实体类写法

```java
@Data
public class BookVo {

    private Integer id;

    @Mapping(translator = "getBookList", mappers = @Mapper(value = "id"), groupKey = "id",receive = "bookName")
    private String bookName0;

    @Mapping(translator = "getBookArr", mappers = @Mapper(value = "id"), groupKey = "id",receive = "bookName")
    private String bookName1;

    @Mapping(translator = "getBookSet", mappers = @Mapper(value = "id"), groupKey = "id",receive = "bookName")
    private String bookName2;

    @Mapping(translator = "getBookMap", mappers = @Mapper(value = "id"), groupKey = "id",receive = "bookName")
    private String bookName3;
}

```

### 自定义实现结果处理

```java
@Component
public class BookResultHandler implements ResultHandler<List<Book>, Map<Integer, Book>, Book> {
    @Override
    public Map<Integer, Book> handle(List<Book> result, String[] groupKey) {
        Map<Integer, Book> map = result.stream()
                .distinct()
                .collect(Collectors.toMap(Book::getId, x -> x));
        return map;
    }

    @Override
    public Book map(Map<Integer, Book> processResult, int index, Object source, Object[] mapperKey) {
        return processResult.get(mapperKey[0]);
    }
}
```

```java
@Data
public class BookVo3 {

    private Integer id;

    @Mapping(translator = "getBookList",
            strategy = MappingStrategy.BATCH,
            mappers = @Mapper(value = "id"),
            resultHandler = "@bookResultHandler",
            receive = "bookName")
    private String bookName;
}
```

## 结果分组依据 

`groupKey`

分发依据，**框架默认的DefaultResultHandler会尝试配合groupKey属性，将结果通过streamApi转化为Map，然后在分发阶段根据mappers属性配对分发。**

例如，结果是List<User>，User中有userId属性，然后可以指定groupKey为userId，

结果分配器将调用User的getUserId方法对List<User>进行分组。

当然，开发者自定义ResultHandler的时候，对类型是可见的，所以未必会使用到该字段。

## 翻译映射字段 

`mappers`

> 关于mapper和`@TransMapper`，详细请看[生成翻译器篇](genTranslator?#transmapper)

`@Mapping`的`mappers`属性，用于获取对象的属性，然后使用paramHandler处理后，逐一填充至翻译器中为`mapper`的参数中。

`mappers`属性由`@Mapper`数组组成。

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

    /**
     * 记录属性数组，获取属性后将用ParamHandler进行处理
     * @return
     */
    String[] value() default "";

    /**
     * 参数处理器
     * 使用 全类名 指定该参数的实现类
     * 也可使用 @ + `beanName` 的方式指定
     * @see com.superkele.translation.core.mapping.ParamHandler
     */
    String paramHandler() default "";
}
```

`@Mapper`的`value`用来记录属性数组，当获取属性后，使用paramHandler处理后传递给翻译器。当然，**框架默认的paramHandler可以为您处理绝大多数情况，包括参数类型转换，自动组成List，Array，Set等等。**

`value`支持**嵌套使用**，例如next.next.val

```java
class Node{
	Node next;
	int val;
}
```

### 参数处理器

`paramHandler`

```java
/**
 * 将 翻译中的mapper参数 转化为实际需要的参数
 *
 * @param <S> source，传递的mapper参数
 * @param <T> target，翻译器实际要求的参数类型
 */
public interface ParamHandler<S, T> {

    /**
     * 对单个参数处理
     *
     * @param param       原参数
     * @param targetClazz 目标参数的类
     * @param types       目标参数的泛型
     * @return translator的实际需求参数
     */
    T wrapper(S param, Class<S> sourceClazz, Class<T> targetClazz, Class[] types) throws ParamHandlerException;

    /**
     * 对于集合翻译情况，存在通过列表查询列表的情况
     * 调用该方法，将所有的参数组装成一个列表然后传参
     *
     * @param params      集合内mapper参数对于的值的集合
     * @param targetClazz 目标参数的类
     * @param types       目标参数的泛型
     */
    T wrapperBatch(List<S> params, Class<S> sourceClazz, Class<T> targetClazz, Class[] types) throws ParamHandlerException;
}
```

### 单体翻译

当映射策略为单体翻译时，框架会为每一个翻译属性各执行一次翻译器。

在收集到单个对象的mapper属性后，框架会调用`ParamHandler`的`wrapper`方法，来包装mapper参数，使其成为翻译器真正需要的参数。

例如，这是框架的默认实现，可以将参数转为List，Arr，Set，或是其他类。

开发者可以自行参阅以下代码实现ParamHandler。

```java
public class DefaultParamHandler implements ParamHandler<Object, Object> {

    @Override
    public Object wrapper(Object param, Class<Object> sourceClazz, Class<Object> targetClazz, Class[] types) throws ParamHandlerException {
        if (sourceClazz.isAssignableFrom(targetClazz)) {
            return param;
        }
        if (targetClazz.isArray()) {
            Class<?> componentType = targetClazz.getComponentType();
            Object array = Array.newInstance(componentType, 1);
            if (param == null || sourceClazz.isAssignableFrom(componentType)) {
                Array.set(array, 0, param);
            } else {
                Array.set(array, 0, Convert.convert(componentType, param));
            }
            return array;
        } else if (ArrayList.class.isAssignableFrom(targetClazz)) {
            ArrayList arrayList = new ArrayList(1);
            param = predictAndProcess(param, types);
            arrayList.add(param);
            return arrayList;
        } else if (HashSet.class.isAssignableFrom(targetClazz)) {
            HashSet<Object> set = new HashSet<>();
            param = predictAndProcess(param, types);
            set.add(param);
            return set;
        } else if (LinkedList.class.isAssignableFrom(targetClazz)) {
            LinkedList<Object> linkedList = new LinkedList<>();
            param = predictAndProcess(param, types);
            linkedList.add(param);
            return linkedList;
        } else {
            return Convert.convert(targetClazz, param);
        }
    }

    /**
     * 判断泛型类型是否匹配并转换
     *
     * @param param
     * @param types
     * @return
     */
    private Object predictAndProcess(Object param, Class[] types) {
        if (types != null && types.length > 0 && param != null) {
            if (param != null && !types[0].isInstance(param)) {
                param = Convert.convert(types[0], param);
            }
        }
        return param;
    }
	//....省略其他方法

}

```

### 批量翻译

在批量翻译时，框架会调用`ParamHandler`的`wrapperBatch`方法，此时，框架会收集起批量处理对象的mapper，然后一起处理。

例如，下面代码中，假如返回体是List<Animal3>，框架将会把所有对象的id参数收集起来包装成 List<Integer>，然后使用wrapperBatch处理，变成List，Array或者Set，或是其他开发者需要的复杂参数。



```java
@Data
public class Animal3 {

    static int counter = 0;

    private Integer id;

    private Integer typeId = ++counter;

    @Mapping(translator = "getByTypeIdList", mappers = @Mapper(value = "typeId"))
    private String typeName1;


    @Mapping(translator = "getByTypeIdList",strategy = MappingStrategy.BATCH, mappers = @Mapper(value = "typeId"))
    private String typeName2;

    @Mapping(translator = "getByTypeIdArr",strategy = MappingStrategy.BATCH, mappers = @Mapper(value = "typeId"))
    private String typeName3;

    @Mapping(translator = "getByTypeIdSet",strategy = MappingStrategy.BATCH, mappers = @Mapper(value = "typeId"))
    private String typeName4;
}
```

### 其他示例

```java
@Data
public class Animal2 {

    private String var1;

    private String var2;

    @Mapping(translator = "getNameByVar1AndVar2", mappers = @Mapper(value = {"var1", "var2"}))
    private String var3;

    @Mapping(translator = "getNameByVar1AndVar2", mappers = {
            @Mapper(value = "var1"),
            @Mapper(value = "var2", paramHandler = "@stringToListParamHandler")}
    )
    private String var4;


    @Translator("getNameByVar1AndVar2")
    public static String getNameByVar1AndVar2(@TransMapper Integer var1,@TransMapper List<String> var2) {
        return var1 + "=>" + var2;
    }
}
```

## 翻译条件补充

`other`

> 关于mapper和`@TransMapper`，详细请看[生成翻译器篇](genTranslator?#transmapper)

`@Mapping`使用`other`属性来接收常量数组，然后逐一填充至翻译器的非mapper的参数中。

other属性会尝试变为方法中需要的类型。

## 选择结果填充

`receive`

当您希望获取翻译结果中的某个属性时，

可以使用`@Mapping`的receive属性

**【重要】**调用同一翻译器，且mapper，ohter等参数一致的情况下，如果该翻译器已经执行且获取到了结果，则不会重复执行。

例如

```java

    @Mapping(translator = "getOrdersByIds",
            strategy = MappingStrategy.BATCH,
            mappers = @Mapper("id"),
            receive = "orderNo")
    private String orderNo;

    @Mapping(translator = "getOrdersByIds",
            strategy = MappingStrategy.BATCH,
            mappers = @Mapper("id"),
            receive = "createTime")
    private String createTime;

```

orderNo会接收getOrdersByIds翻译器翻译结果处理后，分配给它的Order对象的orderNo属性，createTime会接收对应createTime属性。

支持嵌套调用，例如 next.next.val

```java
class Node{
	Node next;
	int val;
}
```

## 翻译执行时机

`timing`

`@Mapping`的timing属性可以决定该字段翻译的时机

```java
public enum TranslateTiming {

    AFTER_RETURN,
    JSON_SERIALIZE,
    NO_EXECUTE;
}
```

`AFTER_RETURN`将在processor处理后立马赋值（可以使用spring依赖注入提前翻译处理，也可以使用注解，交由aop处理）。

`JSON_SERIALIZE`将在对象json序列化的执行。

`NO_EXECUTE`则为忽略该字段。

## 不为空时是否翻译

`notNullMapping`

`@Mapping`的notNullMapping注解可以决定，当该属性的值已经不为空的时候，是否调用翻译器填充值

true：不为空也翻译填充

## 翻译排序

`sort`

有时候，我们需要拿到一个值之后，在映射另一个值

例如，每个员工有部门，数据表中记录了员工的部门Id。这个时候，我们想获取该员工所属的部门名称，就可以先翻译出员工的部门Id，再根据部门Id渲染出部门名称。

`@Mapping`的sort属性可以控制翻译器的执行顺序，从小到大依次执行

```java
@Data
public class ProductVo2 {

    private Integer productId;

    private String productName;

    @Mapping(translator = "getTypeId", mappers = @Mapper("productId"), sort = 0)
    private Integer typeId;

    @Mapping(translator = "getTypeById", mappers = @Mapper("typeId"), sort = 1, receive = "typeName")
    private String typeName;
}
```

测试

```java
    /**
     * 测试 排序映射是否满足要求
     * @see com.superkele.demo.processor.ProductVo2
     */
    @Test
    @Repeat(10)
    public void testSortMapping() {
        ProductVo2 productVo = new ProductVo2();
        int productId = new Random().nextInt(100);
        productVo.setProductId(productId);
        productVo.setProductName("商品" + productId);
        process(productVo);
        Assert.assertNotNull(productVo.getTypeId());
        Assert.assertNotNull(productVo.getTypeName());
    }
```

结果

![image-20240701031843630](./assets/image-20240701031843630.png)

## 异步翻译

`async`

当一些字段没有关联性时，即可使用`@Mapping`的**async**属性开启异步翻译，不过需要先配置一个异步翻译器使用的线程池。

```java
    @Bean
    public TranslationAutoConfigurationCustomizer translationAutoConfigurationCustomizer() {
        return config -> {
            config.setThreadPoolExecutor(threadPoolExecutor);
        };
    }
```

然后我们，我们就可以实现异步翻译了，异步翻译仍然会遵循按sort排序翻译。

不过如果开发者希望在异步翻译执行后再翻译，可以使用[回调翻译](#回调翻译（推荐异步翻译使用）)。

## 回调翻译

`after`

`@Mapping`的`after`属性，记录的是属性数组，可以强制该字段在其他字段翻译执行后再翻译，将不再遵守sort的顺序。

如果需要希望控制在指定的**几个属性翻译填充后**再执行，可以使用该属性。

举个例子

```java
@Data
public class ProductVo3 {

    private Integer productId;

    private String productName;

    @Mapping(translator = "getTypeId", async = true, mappers = @Mapper("productId"), sort = 0)
    private Integer typeId;

    @Mapping(translator = "getTypeById",  async = true,after = "typeId", mappers = @Mapper("typeId"),receive = "typeName")
    private String typeName;

    @Mapping(translator = "getCurrentTime", async = true,sort = 0)
    private String currentTime;


    @Translator("getCurrentTime")
    public static String currentTime() {
        return DateUtil.now();
    }
}

```

上述代码中，typeId字段和currentTime字段将同时开始翻译，之后执行typeName的翻译。

**慎重使用，在不支持虚拟线程的传统Servlet应用中，异步容易导致线程池资源耗空，使得并发量下降，若IO消耗不大，不必使用异步。**

## 空属性异常处理器

`nullPointerHandler`

`@Mapping`的nullPointerHandler将会处理翻译过程中，**属性值为空导致的空指针异常**。

**注意**，并不是所有的空指针异常都会处理

例如，当mapper="user.id"，user为空导致了id属性获取失败，这时，将会使用nullPointerHandler进行处理。

框架已经提供了两种策略可供选择

```java
public class DefaultNullPointerExceptionHandler implements NullPointerExceptionHandler {
    @Override
    public void handle(NullPointerException exception) {
        throw exception;
    }
}
```

```java
public class IgnoreNullPointerExceptionHandler implements NullPointerExceptionHandler {
    @Override
    public void handle(NullPointerException exception) {
        return;
    }
}

```

默认会抛出异常，如果采用IgnoreNullPointerExceptionHandler将会忽略该异常然后不再注入。

## 关联翻译

`@RefTranslation`

> 此为v1.3.0后增加的功能，它使用`@RefTranslation`来实现相关功能，会对类中的某个属性进行整体翻译，但仍需与`@Mapping`进行配合使用。
>
> 只在timing为AFTER_RETURN时生效
>
> 如果开发者只想使用整体翻译功能，那么只要不指定translator属性即可。

```java
/**
 * 关联翻译项，如果一个类中的某个属性不能简单的使用@Mapping进行翻译，
 * 而需要使用整体的翻译，则可以使用该注解
 * 使用同 @TranslationExecute
 */
@Inherited
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RefTranslation {

    /**
     * 指定是那个类型的
     *
     * @return
     */
    Class<?> type() default Object.class;

    /**
     * 指定翻译某个字段
     */
    String field() default "";

    /**
     * 异步处理翻译(只作用在list返回体中)
     */
    boolean async() default false;

    /**
     * 解包返回体为List的情况 用来应对List<List<T>>,Map 等情况
     */
    Class<? extends TranslationUnpackingHandler> listTypeHandler() default DefaultTranslationTypeHandler.class;

}
```

具体的使用方法类似于`执行翻译篇`中的`@TranslationExecute`注解，这里不做赘述。

直接使用上使用案例，开发者一看便知

```java
@Data
public class ProductVo5 {

    private Integer productId;

    private String productName;

    private Integer typeId;

    @Mapping(translator = "getTypeById", mappers = @Mapper("typeId"), receive = "typeName")
    private String typeName;

    @RefTranslation
    @Mapping
    private ProductVo5 child;
}

```



