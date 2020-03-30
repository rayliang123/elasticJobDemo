# elasticJobDemo
elasticJob分布式调度


增加elastic-job-spring-boot-starter的Maven依赖
目前最新版本1.0.1

第一步添加仓库地址：

<repositories>
		<repository>
		    <id>jitpack.io</id>
		    <url>https://jitpack.io</url>
		</repository>
</repositories>
第二步添加依赖：

<dependency>
	    <groupId>com.rayliang</groupId>
	    <artifactId>elastic-job-spring-boot-starter</artifactId>
	    <version>1.0.5</version>
</dependency>
增加Zookeeper注册中心的配置
elasticjob.zk.serverLists=192.168.10.47:2181
elasticjob.zk.namespace=cxytiandi_job2
Zookeeper配置的前缀是elasticJob.zk

开启Elastic-Job自动配置
开启自动配置只需要在Spring Boot的启动类上增加@EnableElasticJob注解

import java.util.concurrent.CountDownLatch;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.builder.SpringApplicationBuilder;
import com.cxytiandi.elasticjob.annotation.EnableElasticJob;

/**
 * ElasticJob Spring Boot集成案例
 * 
 * @author rayliang
 * 
 * @about http://rayliang.com/about
 *
 */
@SpringBootApplication
@EnableElasticJob
public class ElasticjobApplication {
	
	public static void main(String[] args) {
		 SpringApplication.run(ElasticjobApplication.class, args);
	}
	
}
配置任务
@ElasticJobConf(name = "MySimpleJob", cron = "* * * * * ?",jobParameter = "aaa",shardingTotalCount = 3,
        shardingItemParameters = "0=0,1=1", description = "简单任务", eventTraceRdbDataSource = "datasource",distributedListener ="com.example.elasticjob.listener.MyElasticJobListener" )
public class SimpleJobTest implements SimpleJob {
    @Override
    public void execute(ShardingContext shardingContext) {

        System.out.println(String.format("------Thread ID: %s, 任务总片数: %s, " +
                        "当前分片项: %s.当前参数: %s," +
                        "当前任务名称: %s.当前任务参数: %s"
                ,
                Thread.currentThread().getId(),
                shardingContext.getShardingTotalCount(),
                shardingContext.getShardingItem(),
                shardingContext.getShardingParameter(),
                shardingContext.getJobName(),
                shardingContext.getJobParameter()
        ));

    }
}
任务的配置只需要在任务类上增加一个ElasticJobConf注解，注解中有很多属性，这些属性都是任务的配置，详细的属性配置请查看ElasticJobConf

到此为止，我们就快速的使用注解发布了一个任务，DataflowJob和ScriptJob的使用方式一样。



事件追踪功能使用
事件追踪功能在注解中也只需要配置eventTraceRdbDataSource=你的数据源 就可以使用了，数据源用什么连接池无限制，唯一需要注意的一点是你的数据源必须在spring-boot-elastic-job-starter之前创建，因为spring-boot-elastic-job-starter中依赖了你的数据源，下面我以druid作为连接池来进行讲解。

引入druid的Spring Boot Starter,GitHub地址：druid-spring-boot-starter

<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid-spring-boot-starter</artifactId>
	<version>1.1.2</version>
</dependency>
配置连接池属性：

spring.datasource.druid.log.url=jdbc:mysql://localhost:3306/event_log
spring.datasource.druid.log.username=root
spring.datasource.druid.log.password=123456
spring.datasource.druid.log.driver-class-name=com.mysql.jdbc.Driver
然后在项目中定义一个配置类，配置连接池，手动配置的原因是连接池可以在elastic-job-starter之前被初始化。

@Configuration
public class BeanConfig {
	
	/**
	 * 任务执行事件数据源
	 * @return
	 */
	@Bean("datasource")
	@ConfigurationProperties("spring.datasource.druid.log")
	public DataSource dataSourceTwo(){
	    return DruidDataSourceBuilder.create().build();
	}
	
}

然后在注解中增加数据源的配置即可：

@ElasticJobConf(name = "MySimpleJob", cron = "0/10 * * * * ?", 
	shardingItemParameters = "0=0,1=1", description = "简单任务", eventTraceRdbDataSource = "datasource")
application.properties中配置任务信息
使用注解是比较方便，但很多时候我们需要不同的环境使用不同的配置，测试环境跟生产环境的配置肯定是不一样的，当然你也可以在发布之前将注解中的配置调整好然后发布。

为了能够让任务的配置区分环境，还可以在属性文件中配置任务的信息，当属性文件中配置了任务的信息，优先级就比注解中的高。

首先还是在任务类上加@ElasticJobConf(name = "MySimpleJob")注解，只需要增加一个name即可，任务名是唯一的。

剩下的配置都可以在属性文件中进行配置，格式为elasticJob.任务名.配置属性=属性值

elastic.job.MySimpleJob.cron=0/10 * * * * ?
elastic.job.MySimpleJob.overwrite=true
elastic.job.MySimpleJob.shardingTotalCount=1
elastic.job.MySimpleJob.shardingItemParameters=0=0,1=1
elastic.job.MySimpleJob.jobParameter=test
elastic.job.MySimpleJob.failover=true
elastic.job.MySimpleJob.misfire=true
elastic.job.MySimpleJob.description=simple job
elastic.job.MySimpleJob.monitorExecution=false
elastic.job.MySimpleJob.listener=com.cxytiandi.job.core.MessageElasticJobListener
elastic.job.MySimpleJob.jobExceptionHandler=com.cxytiandi.job.core.CustomJobExceptionHandler
elastic.job.MySimpleJob.disabled=true

分布式场景中仅单一节点执行的监听

若作业处理数据库数据，处理完成后只需一个节点完成数据清理任务即可。此类型任务处理复杂，需同步分布式环境下作业的状态同步，提供了超时设置来避免作业不同步导致的死锁，请谨慎使用。

/**
 * @ClassName ElasticJobListener
 * @Description
 * @Author RayLiang
 * @Date 2020/3/26 8:57
 * @Version 1.0
 **/
public class MyElasticJobListener extends AbstractDistributeOnceElasticJobListener {


    public MyElasticJobListener(long startedTimeoutMilliseconds, long completedTimeoutMilliseconds) {
        super(startedTimeoutMilliseconds, completedTimeoutMilliseconds);
    }

    @Override
    public void doBeforeJobExecutedAtLastStarted(ShardingContexts shardingContexts) {
        System.out.println("listener 总片数----"+shardingContexts.getShardingTotalCount());
    }

    @Override
    public void doAfterJobExecutedAtLastCompleted(ShardingContexts shardingContexts) {
            System.out.println("listener 总片数----"+shardingContexts.getShardingTotalCount());
    }
}
ElasticJobConf注解中的distributedListener属性为定义的监听器，记得定义为全路径类。
