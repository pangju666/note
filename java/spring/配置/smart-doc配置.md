# 插件配置

```xml
<plugin>
    <groupId>com.ly.smart-doc</groupId>
    <artifactId>smart-doc-maven-plugin</artifactId>
    <version>3.0.9</version>
    <configuration>
       <configFile>${basedir}/src/main/resources/smart-doc.json</configFile>
       <includes>
          <!-- 使用了mybatis-plus的Page分页需要include所使用的源码包 -->
          <include>com.baomidou:mybatis-plus-extension</include>
          <!-- 使用了mybatis-plus的IPage分页需要include mybatis-plus-core-->
          <include>com.baomidou:mybatis-plus-core</include>
       </includes>
       <projectName>古探长发票管理服务</projectName>
    </configuration>
    <executions>
       <execution>
          <phase>package</phase>
       </execution>
    </executions>
</plugin>
```

文件名：smart-doc.json

文件位置：src/main/resources

```json
{
  "outPath": "target/smart-doc",
  "packageFilters": "cn.gtzhang.personnel.web.controller.*",
  "projectName": "古探长人员服务",
  "appToken": "5465575b137f40dfb33f0802f8f845b5",
  "openUrl": "http://swagger.int/api",
  "replace": true,
  "tornaDebug": true,
  "framework": "spring",
  "showValidation": true,
  "debugEnvName": "本地环境",
  "debugEnvUrl": "http://gtz-personnel-api.int"
}
```

# 踩坑笔记

**RequestHeader、RequestParam、PathVariable、RequestPart**

```java
/** 接口文档可以正确显示 app 为请求头参数 */
@GetMapping("/mini-program/openid")
public String getOpenid(@RequestParam(value = "code") @NotBlank(message = "code不可为空") String code,
                   @RequestHeader(value = "app") String appName) {
    return wechatService.getMiniProgramOpenid(code, appName);
}
```

```java
/** 接口文档会错误显示 appName 为请求头参数 */
@GetMapping("/mini-program/openid")
public String getOpenid(@RequestParam(name = "code") @NotBlank(message = "code不可为空") String code,
                   @RequestHeader(name = "app") String appName) {
    return wechatService.getMiniProgramOpenid(code, appName);
}
```