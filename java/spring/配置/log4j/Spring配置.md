
```xml
<dependency>  
    <groupId>org.springframework.boot</groupId>  
	<artifactId>spring-boot-starter</artifactId>
    <exclusions>  
       <exclusion>  
          <groupId>org.springframework.boot</groupId>  
          <artifactId>spring-boot-starter-logging</artifactId>  
       </exclusion>  
    </exclusions>  
</dependency>

<dependency>  
    <groupId>org.springframework.boot</groupId>  
    <artifactId>spring-boot-starter-log4j2</artifactId>  
</dependency>  

<!-- 如果使用xml格式配置文件，则不需要加这个 -->
<dependency>  
    <groupId>com.fasterxml.jackson.dataformat</groupId>  
    <artifactId>jackson-dataformat-yaml</artifactId>  
</dependency>
```


```yaml
logging:  
  config: classpath:log4j2/local.yml
```