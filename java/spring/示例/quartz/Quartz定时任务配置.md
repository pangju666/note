```java
@Profile({"prod", "test"})
@Slf4j
@SpringBootConfiguration
public class QuartzConfig {
    @Bean("thumbnailStatusCleanJobDetail")
    public JobDetail thumbnailStatusCleanJobDetail() {
       return JobBuilder.newJob(ThumbnailStatusCleanJob.class)
          .withIdentity("thumbnailStatusClean")
          .withDescription("低访问频率缩略图状态清除")
          .storeDurably()
          .build();
    }

    @Bean("compressFileCleanJobDetail")
    public JobDetail compressFileCleanJobDetail() {
       return JobBuilder.newJob(CompressFileCleanJob.class)
          .withIdentity("compressFileClean")
          .withDescription("压缩文件清理")
          .storeDurably()
          .build();
    }

    @Bean
    public Trigger thumbnailStatusCleanTrigger(@Qualifier("thumbnailStatusCleanJobDetail") JobDetail jobDetail) throws ParseException {
       return TriggerBuilder.newTrigger()
          .forJob(jobDetail)
          .withIdentity("thumbnailStatusCleanTrigger")
          .withSchedule(CronScheduleBuilder.cronScheduleNonvalidatedExpression("0 0 3 ? * SUN"))
          .build();
    }

    @Bean
    public Trigger compressFileCleanTrigger(@Qualifier("compressFileCleanJobDetail") JobDetail jobDetail) throws ParseException {
       return TriggerBuilder.newTrigger()
          .forJob(jobDetail)
          .withIdentity("compressFileCleanTrigger")
          .withSchedule(CronScheduleBuilder.cronScheduleNonvalidatedExpression("0 0 3 ? * SUN"))
          .build();
    }
}
```