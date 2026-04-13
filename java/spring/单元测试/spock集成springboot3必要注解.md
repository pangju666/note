在测试实例类上添加

```java
@ActiveProfiles("dev-test")
@ContextConfiguration(classes = Application.class, loader = SpringBootContextLoader.class)
class XXXSpec extends Specification {
}
```

若需要测试controller则还需要添加

**@WebAppConfiguration**