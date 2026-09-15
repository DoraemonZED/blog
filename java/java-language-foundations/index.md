> 本文面向希望系统掌握现代 Java 的开发者。示例以当前长期支持版本的通用语法为基础；涉及新语法时，应以项目实际 JDK 和对应 JEP 为准。

## 1. 类型系统与对象模型

Java 是静态、强类型语言。基本类型直接保存值，引用类型保存对象引用。写业务代码时最重要的不是背诵八种基本类型，而是明确三个边界：`null` 只适用于引用、数值转换可能丢失精度、对象相等应区分 `==` 与 `equals`。

类封装状态和行为，接口描述能力，继承表达“是一种”，组合表达“拥有”。业务系统应优先组合：继承会把父类实现细节暴露给子类，而构造器注入接口更容易测试和替换。

```java
public interface PricePolicy {
    BigDecimal calculate(Order order);
}

public final class OrderService {
    private final PricePolicy policy;

    public OrderService(PricePolicy policy) {
        this.policy = Objects.requireNonNull(policy);
    }
}
```

## 2. 泛型与类型擦除

泛型把类型错误提前到编译期。`List<String>` 并不是运行时独立于 `List<Integer>` 的新类型，大多数泛型信息会被擦除，因此不能直接 `new T()`，也不能可靠地用 `instanceof List<String>`。

记住 PECS：只读取的参数使用 `? extends T`，只写入的参数使用 `? super T`。公共 API 尽量返回清晰的领域类型，不要让通配符扩散到业务代码。

## 3. 异常与资源管理

受检异常适合调用者能够恢复的情况；运行时异常适合编程错误、非法状态或无法在当前层恢复的问题。不要捕获 `Exception` 后静默忽略，也不要丢失原始 cause。

实现 `AutoCloseable` 的资源应使用 try-with-resources。异常信息应包含操作对象和必要上下文，但不得记录密码、Token、身份证号等敏感数据。

## 4. 注解、反射与动态代理

注解本身不执行逻辑，它只是元数据。框架通过编译期处理器或运行时反射读取注解。反射适合框架边界，不适合散布在核心业务中：它削弱静态检查，也可能带来可访问性和性能问题。

JDK 动态代理基于接口；需要代理具体类时，框架通常使用字节码生成。理解这一点有助于排查 Spring AOP 中自调用不生效、`final` 方法无法增强等问题。

## 5. Lambda、Stream 与 Optional

Lambda 适合传递行为，Stream 适合声明式的数据变换。流是一次性的，操作分为惰性的中间操作和触发执行的终止操作。不要在 `map`、`filter` 中修改外部共享状态，也不要因为“看起来更函数式”而把简单循环写成难懂的链。

`Optional` 主要用于返回值表达“可能没有”，通常不应作为实体字段、方法参数或集合元素。集合没有结果时优先返回空集合。

## 6. 现代 Java 建模能力

- `record` 适合不可变数据载体，会生成访问器、`equals`、`hashCode` 和 `toString`。
- sealed 类型可以显式限制允许的子类型，适合有限状态集合。
- 模式匹配减少类型判断后的强制转换。
- 文本块改善 SQL、JSON 等多行文本可读性，但仍应使用参数化 SQL。

## 7. 模块、包与可见性

包解决命名与访问边界，模块系统进一步声明依赖和导出。大多数 Spring Boot 服务仍以构建工具的模块划分为主，但库开发、桌面应用和强封装场景可考虑 JPMS。无论是否使用 `module-info.java`，都应让领域模块保持单向依赖，禁止循环引用。

## 学习检查表

- 能解释值相等、引用相等和哈希契约。
- 能正确选择接口、组合、继承和不可变对象。
- 能解释泛型擦除和 PECS。
- 能设计异常边界并安全关闭资源。
- 能说明注解为何能够被框架识别。
- 能判断 Stream 是否比循环更清晰。
- 能使用 record、sealed 类型表达领域模型。

## 参考资料

- [Java Language Specification](https://docs.oracle.com/javase/specs/)
- [OpenJDK JEP 索引](https://openjdk.org/jeps/0)
- [Java API 文档](https://docs.oracle.com/en/java/javase/)

