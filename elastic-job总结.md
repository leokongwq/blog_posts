## elastic-job 是什么？

还是看官方文档介绍：<http://elasticjob.io/docs/elastic-job-lite/00-overview/>

## elastic-job 实现分析

### 架构

![Elastic-Job-Lite Architecture](http://elasticjob.io/docs/elastic-job-lite/img/architecture/elastic_job_lite.png)

从上图可以看到 elastic-job 的实现主要依赖的外部组件是zookeeper。 用zookeeper来实现分布式任务的协调及相关任务信息的保存。具体任务的调度触发还是依赖core模块里面的Quartz API提供的能力。

### spring 集成分析

```java
<reg:zookeeper id="regCenter"
                   server-lists="${elastic.job.zk.host}"
                   namespace="viptrade-job"
                   base-sleep-time-milliseconds="1000"
                   max-sleep-time-milliseconds="3000"
                   max-retries="3"
                   session-timeout-milliseconds="5000"
                   connection-timeout-milliseconds="3000"
/>
<job:simple id="mySimpleElasticJob" class="com.leokongwq.elasticjob.elasticjobdemo.job.MySimpleElasticJob" 
    registry-center-ref="regCenter"
    cron="0/10 * * * * ?"
    job-parameter="name=sky;age=21"
    sharding-total-count="3"
    sharding-item-parameters="0=A,1=B,2=C"/>    
```

从上面的配置容易看出， elastic-job 是采用spring提供的命名空间扩展能力进行集成的。

具体是通过如下的配置和类实现：

```xml
http\://www.dangdang.com/schema/ddframe/reg/reg.xsd=META-INF/namespace/reg.xsd
http\://www.dangdang.com/schema/ddframe/job/job.xsd=META-INF/namespace/job.xsd

http\://www.dangdang.com/schema/ddframe/reg=io.elasticjob.lite.spring.reg.handler.RegNamespaceHandler
http\://www.dangdang.com/schema/ddframe/job=io.elasticjob.lite.spring.job.handler.JobNamespaceHandler
```



`RegNamespaceHandler` 来负责解析 `reg`命名空间配置。`JobNamespaceHandler` 来解析`job`命名空间配置

#### 注册中心

目前只支持基于zookeeper的注册中心，核心的类是`ZookeeperRegistryCenter`。

支持的核心配置如下：

- serverLists  zookeeper节点列表，逗号分隔
- namespace 任务的命名空间， 同时也是Curator操作的命名空间，所有的操作都是相对于这个命名空间的。
- baseSleepTimeMilliseconds maxSleepTimeMilliseconds maxRetries 这三个都是底层Curator的重试策略`ExponentialBackoffRetry`配置属性
- sessionTimeoutMilliseconds zookeeper 回话超时时间
- connectionTimeoutMilliseconds zookeeper 连接超时时间
- digest zookeeper认证信息

#### job

不同的job定义是通过不同的Handler类处理的。

SimpleJobBeanDefinitionParser 处理 simple 命名空间的任务定义

DataflowJobBeanDefinitionParser 处理 dataflow命名空间的任务定义

ScriptJobBeanDefinitionParser 处理 script命名空间的任务定义

所有job 在spring容器中最终生成的 Bean 类型 都是 `SpringJobScheduler` 并通过`init`方法进行初始化。



### zookeeper 目录结构

/jobname

/jobname/systemtime/current 保存当前时间

/jobname/config  job 配置信息

/jobname/guarantee/started/0  # 分布式环境下，任务同时开始

/jobname/guarantee/stoped/0  # 分布式环境下，任务同时结束

/jobname/servers/ip #曾经注册了可以执行job的机器. 数据节点的值为空字符串表示正常，disabled表示不可用

/jobname/instances/ip@-@pid #表示存活的实例，节点值为空字符串

/jobname/sharding/0/instance # 节点的值是instanceId eg. 10.3.9.7@-@4256

/jobname/sharding/0/running # 表示分片运行状态

/jobname/sharding/0/misfire # 表示任务被错过执行标志，第二次调度时，第一次的任务还没有执行完成。

/jobname/sharding/0/disabled # 表示分片被禁用

/jobname/sharding/0/failover # 表示执行分片failover， 节点的值是执行失败分片的实例id,即 instanceId

/jobname/learder/sharding/necessary # 表示Job是否需要分片， 分片完成后被删除

/jobname/learder/sharding/processing # 表示Job正在进行分片， 分片完成后被删除

/jobname/learder/election/latch # 表示进行选主的根节点

/jobname/learder/election/instance # 表示被选为master的实例，节点的值是master实例的instanceId

/jobname/learder/failover/items/0 # 表示需要failover的分片



### Quartz 如何调度任务？

#### QuartzSchedulerThread

```java QuartzSchedulerThread
JobRunShell shell = null;
try {
    shell = qsRsrcs.getJobRunShellFactory().createJobRunShell(bndle);
    shell.initialize(qs);
} catch (SchedulerException se) {
    qsRsrcs.getJobStore().triggeredJobComplete(triggers.get(i), bndle.getJobDetail(), CompletedExecutionInstruction.SET_ALL_JOB_TRIGGERS_ERROR);
    continue;
}

if (qsRsrcs.getThreadPool().runInThread(shell) == false) {
    // this case should never happen, as it is indicative of the
    // scheduler being shutdown or a bug in the thread pool or
    // a thread pool being used concurrently - which the docs
    // say not to do...
    getLog().error("ThreadPool.runInThread() return false!");
    qsRsrcs.getJobStore().triggeredJobComplete(triggers.get(i), bndle.getJobDetail(), CompletedExecutionInstruction.SET_ALL_JOB_TRIGGERS_ERROR);
}
```

#### JobFactory

`JobFactory`负责Job的创建。 一般是通过`JobDetail`指的class属性，通过反射API进行创建。

```java
public class SimpleJobFactory implements JobFactory {

    private final Logger log = LoggerFactory.getLogger(getClass());
    
    protected Logger getLog() {
        return log;
    }
    
    public Job newJob(TriggerFiredBundle bundle, Scheduler Scheduler) throws SchedulerException {

        JobDetail jobDetail = bundle.getJobDetail();
        Class<? extends Job> jobClass = jobDetail.getJobClass();
        try {
            if(log.isDebugEnabled()) {
                log.debug(
                    "Producing instance of Job '" + jobDetail.getKey() + 
                    "', class=" + jobClass.getName());
            }
            
            return jobClass.newInstance();
        } catch (Exception e) {
            SchedulerException se = new SchedulerException(
                    "Problem instantiating class '"
                            + jobDetail.getJobClass().getName() + "'", e);
            throw se;
        }
    }
}
```

#### JobRunShell 

`JobRunShell `  提供Job执行环境，真正负责Job的执行。

```java JobRunShell
// 任务初始化
public void initialize(QuartzScheduler sched)
        throws SchedulerException {
        this.qs = sched;

        Job job = null;
        JobDetail jobDetail = firedTriggerBundle.getJobDetail();

        try {
            //负责Job的创建， 通过反射API创建
            job = sched.getJobFactory().newJob(firedTriggerBundle, scheduler);
        } catch (SchedulerException se) {
            sched.notifySchedulerListenersError(
                    "An error occured instantiating job to be executed. job= '"
                            + jobDetail.getKey() + "'", se);
            throw se;
        } catch (Throwable ncdfe) { // such as NoClassDefFoundError
            SchedulerException se = new SchedulerException(
                    "Problem instantiating class '"
                            + jobDetail.getJobClass().getName() + "' - ", ncdfe);
            sched.notifySchedulerListenersError(
                    "An error occured instantiating job to be executed. job= '"
                            + jobDetail.getKey() + "'", se);
            throw se;
        }
		// 创建Job执行上下文
        this.jec = new JobExecutionContextImpl(scheduler, firedTriggerBundle, job);
}

// 任务执行
public void run() {
	qs.addInternalSchedulerListener(this);

    try {
        OperableTrigger trigger = (OperableTrigger) jec.getTrigger();
        JobDetail jobDetail = jec.getJobDetail();

        do {
			// 删除一部分代码
            long startTime = System.currentTimeMillis();
            long endTime = startTime;

            // execute the job
            try {
                log.debug("Calling execute on job " + jobDetail.getKey());
                // job 开始执行
                job.execute(jec);
                endTime = System.currentTimeMillis();
            } catch (JobExecutionException jee) {
                endTime = System.currentTimeMillis();
                jobExEx = jee;
                getLog().info("Job " + jobDetail.getKey() +
                              " threw a JobExecutionException: ", jobExEx);
            } catch (Throwable e) {
                endTime = System.currentTimeMillis();
                getLog().error("Job " + jobDetail.getKey() +
                               " threw an unhandled Exception: ", e);
                SchedulerException se = new SchedulerException(
                    "Job threw an unhandled exception.", e);
                qs.notifySchedulerListenersError("Job ("
                                                 + jec.getJobDetail().getKey()
                                                 + " threw an exception.", se);
                jobExEx = new JobExecutionException(se, false);
            }
			// 删除部分代码
        } while (true);
    } finally {
        qs.removeInternalSchedulerListener(this);
    }
}
```

#### LiteJob

```java
/**
 * Lite调度作业.
 *
 * @author zhangliang
 */
public final class LiteJob implements Job {
    
    @Setter
    private ElasticJob elasticJob;
    
    @Setter
    private JobFacade jobFacade;
    
    @Override
    public void execute(final JobExecutionContext context) throws JobExecutionException {
        JobExecutorFactory.getJobExecutor(elasticJob, jobFacade).execute();
    }
}
```

`LiteJob` 实现了Quartz 中表示任务的接口`Job` ， 通过`LiteJob`类，elastic-job实现了和Quartz的整合。

### 核心类

#### JobInstance & JobRegistry

JobRegistry 表示job注册中心，在Job创建时 以job的名称为key， JobInstance为Value保存到一个Map中。

JobInstance 本质上是一个字符串，来唯一标识一个Job

```java
JobScheduler
JobRegistry.getInstance().addJobInstance(liteJobConfig.getJobName(), new JobInstance());

JobRegistry

private Map<String, JobInstance> jobInstanceMap = new ConcurrentHashMap<>();
/**
* 添加作业实例.
*
* @param jobName 作业名称
* @param jobInstance 作业实例
*/
public void addJobInstance(final String jobName, final JobInstance jobInstance) {
    jobInstanceMap.put(jobName, jobInstance);
}
```



#### GuaranteeService

保证分布式任务全部开始和结束状态的服务。

一个多片任务，通过该类实现所有分片同时开始，等待所有分片都完成。

##### 注册开始分片：

如果一个Job分为三片，那么就会在zookeeper上创建三个不同的`持久化`节点

```java
job 开始
/命名空间/job名称/guarantee/started/1
/命名空间/job名称/guarantee/started/2
/命名空间/job名称/guarantee/started/3

job 结束
/命名空间/job名称/guarantee/completed/1
/命名空间/job名称/guarantee/completed/2
/命名空间/job名称/guarantee/completed/3
```

#### LeaderService

`LeaderService`是Job在zookeeper上主节点服务类，主要作用如下：

1. Job主节点选举
2. 判断当前节点是否是主节点
3. 判断Job是否有主节点
4. 删除主节点

选举主节点底层是通过Curator提供的master选举功能。创建的节点路径是`Job名称/leader/election/latch/latch-*`  , * 表示1,2,3...(临时顺序节点)

并写入当前Job执行实例的`instanceId`

说白了就是操作`jobName/leader`及其子节点的工具类

#### ServerService

`ServerService`类用来操作`jobName/servers`及其子节点。主要功能如下：

1. 创建执行Job的服务节点`jobName`/servers/ip；如果该节点可用，则保存的数据为空字符串，不可用保存的数据为`DISABLED`
2. 判断是否还有可用的Job执行节点。其实就是判断`jobName`/servers/ip`节点存在并且保存的值是空字符串，并且节点`jobName/instances/ip`存在。

#### InstanceService

`InstanceService`用来操作节点`jobName/instances`。主要功能如下：

1. 标识Job实例上线。其实就是创建一个节点`jobName/instances/instanceId`
2. 删除instance节点

#### ListenerManager

`ListenerManager`用来管理所有elastic-job的各种`Listener`或其他的`ListenerManager`

```java
/**
 * 作业注册中心的监听器管理者.
 * 
 * @author zhangliang
 */
public final class ListenerManager {
    
    private final JobNodeStorage jobNodeStorage;
    
    private final ElectionListenerManager electionListenerManager;
    
    private final ShardingListenerManager shardingListenerManager;
    
    private final FailoverListenerManager failoverListenerManager;
    
    private final MonitorExecutionListenerManager monitorExecutionListenerManager;
    
    private final ShutdownListenerManager shutdownListenerManager;
    
    private final TriggerListenerManager triggerListenerManager;
    
    private final RescheduleListenerManager rescheduleListenerManager;
	/**
	* 任务执行前或执行后的监听器， 在分布式环境下，确保只通知一次。
	*/
    private final GuaranteeListenerManager guaranteeListenerManager;
    
    private final RegistryCenterConnectionStateListener regCenterConnectionStateListener;
    
    public ListenerManager(final CoordinatorRegistryCenter regCenter, final String jobName, final List<ElasticJobListener> elasticJobListeners) {
        jobNodeStorage = new JobNodeStorage(regCenter, jobName);
        electionListenerManager = new ElectionListenerManager(regCenter, jobName);
        shardingListenerManager = new ShardingListenerManager(regCenter, jobName);
        failoverListenerManager = new FailoverListenerManager(regCenter, jobName);
        monitorExecutionListenerManager = new MonitorExecutionListenerManager(regCenter, jobName);
        shutdownListenerManager = new ShutdownListenerManager(regCenter, jobName);
        triggerListenerManager = new TriggerListenerManager(regCenter, jobName);
        rescheduleListenerManager = new RescheduleListenerManager(regCenter, jobName);
        guaranteeListenerManager = new GuaranteeListenerManager(regCenter, jobName, elasticJobListeners);
        regCenterConnectionStateListener = new RegistryCenterConnectionStateListener(regCenter, jobName);
    }
    
    /**
     * 开启所有监听器.
     */
    public void startAllListeners() {
        electionListenerManager.start();
        shardingListenerManager.start();
        failoverListenerManager.start();
        monitorExecutionListenerManager.start();
        shutdownListenerManager.start();
        triggerListenerManager.start();
        rescheduleListenerManager.start();
        guaranteeListenerManager.start();
        jobNodeStorage.addConnectionStateListener(regCenterConnectionStateListener);
    }
}
```

#### ShardingContext

分片上下文。

```java
public final class ShardingContext {
    
    /**
     * 作业名称.
     */
    private final String jobName;
    
    /**
     * 作业任务ID.
     */
    private final String taskId;
    
    /**
     * 分片总数.
     */
    private final int shardingTotalCount;
    
    /**
     * 作业自定义参数.
     * 可以配置多个相同的作业, 但是用不同的参数作为不同的调度实例.
     */
    private final String jobParameter;
    
    /**
     * 分配于本作业实例的分片项.
     */
    private final int shardingItem;
    
    /**
     * 分配于本作业实例的分片参数.
     */
    private final String shardingParameter;
}    
```

#### JobExecutorFactory

```
/**
 * 作业执行器工厂.
 *
 * @author zhangliang
 */
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class JobExecutorFactory {
    
    /**
     * 获取作业执行器.
     *
     * @param elasticJob 分布式弹性作业
     * @param jobFacade 作业内部服务门面服务
     * @return 作业执行器
     */
    @SuppressWarnings("unchecked")
    public static AbstractElasticJobExecutor getJobExecutor(final ElasticJob elasticJob, final JobFacade jobFacade) {
        if (null == elasticJob) {
            return new ScriptJobExecutor(jobFacade);
        }
        if (elasticJob instanceof SimpleJob) {
            return new SimpleJobExecutor((SimpleJob) elasticJob, jobFacade);
        }
        if (elasticJob instanceof DataflowJob) {
            return new DataflowJobExecutor((DataflowJob) elasticJob, jobFacade);
        }
        throw new JobConfigurationException("Cannot support job type '%s'", elasticJob.getClass().getCanonicalName());
    }
}
```

#### AbstractElasticJobExecutor

`AbstractElasticJobExecutor`是Job执行的核心类。它有如下三个子类，分别对应不同种类的任务

1. SimpleJobExecutor
2. DataflowJobExecutor
3. ScriptJobExecutor

### 作业初始化

所有配置的Job 最终都是生成一个调度对象`JobScheduler`进行任务的触发调度。

```java
/**
* 初始化作业.
*/
public void init() {
	//更新配置信息
    LiteJobConfiguration liteJobConfigFromRegCenter = schedulerFacade.updateJobConfiguration(liteJobConfig);
//设置Job当前的分片总数			JobRegistry.getInstance().setCurrentShardingTotalCount(liteJobConfigFromRegCenter.getJobName(), liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getShardingTotalCount());
    // 创建Job调度控制器
	JobScheduleController jobScheduleController = new JobScheduleController(
                createScheduler(), createJobDetail(liteJobConfigFromRegCenter.getTypeConfig().getJobClass()), liteJobConfigFromRegCenter.getJobName());
	//添加作业调度控制器
    JobRegistry.getInstance().registerJob(liteJobConfigFromRegCenter.getJobName(), jobScheduleController, regCenter);
	//注册作业启动信息， 这个步骤非常重要
    schedulerFacade.registerStartUpInfo(!liteJobConfigFromRegCenter.isDisabled());
// 调度Job		jobScheduleController.scheduleJob(liteJobConfigFromRegCenter.getTypeConfig().getCoreConfig().getCron());
}
```

#### 1.冲突检查&更新配置

每个Job的配置信息在 zookeeper上的节点路径是`/命名空间/job名称/config` 格式是一个json字符串。

如果配置信息里面保存的Job `class名称`和本地配置的`class`名称不同，此时elastic-job会提升冲突`JobConfigurationException`。

```java
private void checkConflictJob(final LiteJobConfiguration liteJobConfig) {
    	//查询zookeeper上的配置信息
    	Optional<LiteJobConfiguration> liteJobConfigFromZk = find();
    
        if (liteJobConfigFromZk.isPresent() && !liteJobConfigFromZk.get().getTypeConfig().getJobClass().equals(liteJobConfig.getTypeConfig().getJobClass())) {
            throw new JobConfigurationException("Job conflict with register center. The job '%s' in register center's class is '%s', your job class is '%s'", 
                    liteJobConfig.getJobName(), liteJobConfigFromZk.get().getTypeConfig().getJobClass(), liteJobConfig.getTypeConfig().getJobClass());
        }
}
```

如果Job配置的属性`overwrite=true`，则用本地的配置信息更新zookeeper上的配置信息。

#### 2. 启动所有监听器

```
/**
 * 开启所有监听器.
 */
public void startAllListeners() {
    electionListenerManager.start(); 
    shardingListenerManager.start(); 
    failoverListenerManager.start(); 
    monitorExecutionListenerManager.start(); // 
    shutdownListenerManager.start();
    triggerListenerManager.start();
    rescheduleListenerManager.start();
    guaranteeListenerManager.start();
    jobNodeStorage.addConnectionStateListener(regCenterConnectionStateListener);
}
```

##### ElectionListener

选主监听器，当master奔溃或上下线job服务器或服务器被禁用，进行选主

##### ShardingListenerManager

job分片数调整，或job服务器上下线时触发分片调整（任务下次执行时，真正进行调整，listener只是设置需要调整分片标志）

当前的实现逻辑如下：

```java
class ListenServersChangedJobListener extends AbstractJobListener {
        
        @Override
        protected void dataChanged(final String path, final Type eventType, final String data) {
            if (!JobRegistry.getInstance().isShutdown(jobName) && (isInstanceChange(eventType, path) || isServerChange(path))) {
                shardingService.setReshardingFlag();
            }
        }
        // 是`jobName/instances`开头的路径，并且不是数据更新事件，
        // 那么就必然是 NODE_ADD, 或 NODE_REMOVE 事件了
        private boolean isInstanceChange(final Type eventType, final String path) {
            return instanceNode.isInstancePath(path) && Type.NODE_UPDATED != eventType;
        }
        
        private boolean isServerChange(final String path) {
            return serverNode.isServerPath(path);
        }
}
```



##### FailoverListenerManager

1. job执行中奔溃， 如果设置了Job的failover配置，那么进行Job的故障转移

2. 如果将job的failover配置调整为false，那么删除job所有分片的failover节点。 `jobName/sharding/0/failover`

##### MonitorExecutionListenerManager

1. 如果将job的配置项`monitorExecution`设置为false，那么删除Job所有分片的running节点。`jobName/sharding/0/running`

##### ShutdownListenerManager

1. 如果`jobName/sharding/0/instance` 节点被删除，那么该实例不再参与任务调度。
2. 关闭Monitor Socket,  如果是master，则放弃master身份。关闭 job对应的Quartz调度器。
3. 释放缓存
4. 删除内存中的状态数据

##### TriggerListenerManager

1. 将`jobName/instances/instanceid` 节点的值设置为 `trigger`, 并且当前Job不处于运行中，则直接触发Job的执行。

##### RescheduleListenerManager

1. 当修改任务的cron表达式时， 触发任务重新调度。 

##### GuaranteeListenerManager

1. 当节点`jobName/guarantee/started`节点被删除时，通知对应的Listener.
2. 当节点`jobName/guarantee/completed`节点被删除时，通知对应的Listener.

##### RegistryCenterConnectionStateListener

zk连接状态监听器。

1. 当断开连接 或 连接出去挂起状态，暂时任务调度。` jobScheduleController.pauseJob();`
2. 连接重新建立时，上线`server`和`instance` , 清除分片的`running`状态。恢复job调度。

#### 3.选取主节点

```
leaderService.electLeader();
/**
 * 选举主节点.
 */
public void electLeader() {
    log.debug("Elect a new leader now.");
    jobNodeStorage.executeInLeader(LeaderNode.LATCH, new 	LeaderElectionExecutionCallback());
    log.debug("Leader election completed.");
}
```

主节点选取成功后，获得master角色的实例，将它的实例id写入`jobName/leader/election/instance`节点

####  4.上线服务器和实例

```
serverService.persistOnline(enabled);
instanceService.persistOnline(); 
```

`/jobName/servers/ip` 数据设置为空字符串

`/jobName/instances/instanceid` 数据设置为空字符串

#### 5.设置需要分片信息

```jav
shardingService.setReshardingFlag();
```

设置需要分片标识`/jobName/leader/sharding/necessary` 

#### 6.打开监控端口

```
monitorService.listen();
```

#### 7. 启动分布式状态不一致修复服务

```
if (!reconcileService.isRunning()) {
    reconcileService.startAsync();
}
```

### 作业执行

elastic-job 中作业的执行是通过不同的`JobExecutor`来实现的。核心逻辑都封装在父类`AbstractElasticJobExecutor`中。下面我们主要以`SimpleJobExecutor`来分析Job的执行流程。

#### SimpleJobExecutor

```java
public final class SimpleJobExecutor extends AbstractElasticJobExecutor {
    
    private final SimpleJob simpleJob;
    
    public SimpleJobExecutor(final SimpleJob simpleJob, final JobFacade jobFacade) {
        super(jobFacade);
        this.simpleJob = simpleJob;
    }
    
    @Override
    protected void process(final ShardingContext shardingContext) {
        simpleJob.execute(shardingContext);
    }
}
```

#### Job 执行入口 AbstractElasticJobExecutor

```java  AbstractElasticJobExecutor.execute
/**
* 执行作业.
*/
public final void execute() {
    try {
        // 检查执行环境，主要是检查时间误差是否在允许的时间范围内。
        jobFacade.checkJobExecutionEnvironment();
    } catch (final JobExecutionEnvironmentException cause) {
        jobExceptionHandler.handleException(jobName, cause);
    }
    // 构造Job执行上下文对象，非常重要，后面专门分析
    ShardingContexts shardingContexts = jobFacade.getShardingContexts();
    if (shardingContexts.isAllowSendJobEvent()) {
        jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_STAGING, String.format("Job '%s' execute begin.", jobName));
    }
    // 如果Job仍处于执行状态，则设置missfire标志
    if (jobFacade.misfireIfRunning(shardingContexts.getShardingItemParameters().keySet())) {
        if (shardingContexts.isAllowSendJobEvent()) {
            jobFacade.postJobStatusTraceEvent(shardingContexts.getTaskId(), State.TASK_FINISHED, String.format(
                "Previous job '%s' - shardingItems '%s' is still running, misfired job will start after previous job completed.", jobName, 
                shardingContexts.getShardingItemParameters().keySet()));
        }
        return;
    }
    try {
        // 通知 ElasticJobListener 
        jobFacade.beforeJobExecuted(shardingContexts);
        //CHECKSTYLE:OFF
    } catch (final Throwable cause) {
        //CHECKSTYLE:ON
        jobExceptionHandler.handleException(jobName, cause);
    }
    // 执行 job
    execute(shardingContexts, JobExecutionEvent.ExecutionSource.NORMAL_TRIGGER);
    // 如果开启了misfire功能，则再次Job
    while (jobFacade.isExecuteMisfired(shardingContexts.getShardingItemParameters().keySet())) {
        jobFacade.clearMisfire(shardingContexts.getShardingItemParameters().keySet());
        execute(shardingContexts, JobExecutionEvent.ExecutionSource.MISFIRE);
    }
    // 如果有任务分片执行失败了，并且启用了failover功能，则重新执行失败了的分片
    jobFacade.failoverIfNecessary();
    try {
        // 通知 ElasticJobListener
        jobFacade.afterJobExecuted(shardingContexts);
        //CHECKSTYLE:OFF
    } catch (final Throwable cause) {
        //CHECKSTYLE:ON
        jobExceptionHandler.handleException(jobName, cause);
    }
}
```

####  LiteJobFacade.getShardingContexts

获取Job执行的上下文信息

```
@Override
public ShardingContexts getShardingContexts() {
	boolean isFailover = configService.load(true).isFailover();
	//如果开始了failover功能
    if (isFailover) {
    	// 获取分配到本机的failover分片
        List<Integer> failoverShardingItems = failoverService.getLocalFailoverItems();
        log.debug("failover sharding items is {}", failoverShardingItems);
        // 如果有分配给本实例执行的failover分片，则执行。
        // 这里有个疑问， 本次任务调度，执行上次调度其他实例执行失败的分片， 那该实例本次调度
        // 需要执行的任务还如何执行呢?
        if (!failoverShardingItems.isEmpty()) {
            return executionContextService.getJobShardingContext(failoverShardingItems);
        }
    }
    // 如果需要，进行job分片处理
    shardingService.shardingIfNecessary();
    // 获取分配到本实例的分片，遍历job的所有分片，找出instance节点值和
    // 当前job实例instanceId相同的分片
    List<Integer> shardingItems = shardingService.getLocalShardingItems();
    // 如果开启了failover功能，去除failover分片
    if (isFailover) {
        shardingItems.removeAll(failoverService.getLocalTakeOffItems());
    }
    // 去除处于disabled状态的分片
	shardingItems.removeAll(executionService.getDisabledItems(shardingItems));
	
	return executionContextService.getJobShardingContext(shardingItems);
}
```



#### shardingService.shardingIfNecessary

```java shardingService.shardingIfNecessary()
/**
     * 如果需要分片且当前节点为主节点, 则作业分片.
     * 
     * <p>
     * 如果当前无可用节点则不分片.
     * </p>
     */
public void shardingIfNecessary() {
    List<JobInstance> availableJobInstances = instanceService.getAvailableJobInstances();
    // 存在 jobName/leader/sharding/necessary 并且 有在线的实例
    if (!isNeedSharding() || availableJobInstances.isEmpty()) {
        return;
    }
    // 判断自己是否是主节点 或 等待选举完成后再次判断
    // 只有主节点才能进行分片
    if (!leaderService.isLeaderUntilBlock()) {
        // 非主节点，则等待主节点完成分片操作
        blockUntilShardingCompleted();
        return;
    }
    // 等待正在执行的Job执行完成后，再执行分片调整。
    waitingOtherJobCompleted();
    LiteJobConfiguration liteJobConfig = configService.load(false);
    int shardingTotalCount = liteJobConfig.getTypeConfig().getCoreConfig().getShardingTotalCount();
    log.debug("Job '{}' sharding begin.", jobName);
    jobNodeStorage.fillEphemeralJobNode(ShardingNode.PROCESSING, "");
    resetShardingInfo(shardingTotalCount);
    JobShardingStrategy jobShardingStrategy = JobShardingStrategyFactory.getStrategy(liteJobConfig.getJobShardingStrategyClass());
    jobNodeStorage.executeInTransaction(new PersistShardingInfoTransactionExecutionCallback(jobShardingStrategy.sharding(availableJobInstances, jobName, shardingTotalCount)));
    log.debug("Job '{}' sharding complete.", jobName);
}

/**
 * 分片完成后的回调处理逻辑
*/
@RequiredArgsConstructor
class PersistShardingInfoTransactionExecutionCallback implements TransactionExecutionCallback {

    private final Map<JobInstance, List<Integer>> shardingResults;

    @Override
    public void execute(final CuratorTransactionFinal curatorTransactionFinal) throws Exception {
        for (Map.Entry<JobInstance, List<Integer>> entry : shardingResults.entrySet()) {
            for (int shardingItem : entry.getValue()) {
                curatorTransactionFinal.create().forPath(jobNodePath.getFullPath(ShardingNode.getInstanceNode(shardingItem)), entry.getKey().getJobInstanceId().getBytes()).and();
            }
        }
 // 删除分片相关节点，Follower 节点不再等待，继续执行Job      curatorTransactionFinal.delete().forPath(jobNodePath.getFullPath(ShardingNode.NECESSARY)).and();
        curatorTransactionFinal.delete().forPath(jobNodePath.getFullPath(ShardingNode.PROCESSING)).and();
    }
}
```

#### ExecutionContextService.getJobShardContext

```
/**
 * 获取当前作业服务器分片上下文.
 * 
 * @param shardingItems 分片项, 有可能是只包含failover节点的分片项
 * @return 分片上下文
 */
public ShardingContexts getJobShardingContext(final List<Integer> shardingItems) {
    LiteJobConfiguration liteJobConfig = configService.load(false);
    // 去除还处于执行状态的分片
    removeRunningIfMonitorExecution(liteJobConfig.isMonitorExecution(), shardingItems);
    log.debug("After remove running sharding item, result is {}", shardingItems);
    if (shardingItems.isEmpty()) {
        return new ShardingContexts(buildTaskId(liteJobConfig, shardingItems), liteJobConfig.getJobName(), liteJobConfig.getTypeConfig().getCoreConfig().getShardingTotalCount(), 
                liteJobConfig.getTypeConfig().getCoreConfig().getJobParameter(), Collections.<Integer, String>emptyMap());
    }
    // 构造分片参数map， key是分片下标， values是分片参数
    Map<Integer, String> shardingItemParameterMap = new ShardingItemParameters(liteJobConfig.getTypeConfig().getCoreConfig().getShardingItemParameters()).getMap();
    
    log.info("sharding Item parameters : {}", shardingItemParameterMap);
    // 返回值
    return new ShardingContexts(buildTaskId(liteJobConfig, shardingItems), liteJobConfig.getJobName(), liteJobConfig.getTypeConfig().getCoreConfig().getShardingTotalCount(),
            liteJobConfig.getTypeConfig().getCoreConfig().getJobParameter(), getAssignedShardingItemParameterMap(shardingItems, shardingItemParameterMap));
}
```

### failover实现逻辑

failover指的是在Job执行中，突然宕机，elastic-job 将中断的Job重新调度到其他可用实例的过程。有一下几点需要注意：

1. 开启failover功能。
2. Job可以调度的实例大于1。
3. Job执行结果具有幂等性，否则对业务会有影响。

#### failover 监听

```java
class JobCrashedJobListener extends AbstractJobListener {

    /**
         * 监听 jobName/instances/IP@-@PID 节点的 remove事件，进行failover
         * note: 每个任务都会监听 job实例 crash 的事件, 如果job正在执行中，那么failover是正常的。
         * 但是有一种情况是: job已经执行完成了，此时重新调度执行就有问题了。
         * @param path zk 事件发生的path
         * @param eventType {@link Type}
         * @param data  zk 事件发生的path 节点的数据
         */
    @Override
    protected void dataChanged(final String path, final Type eventType, final String data) {
        // 开启failover特性， 并且是 instance奔溃触发的zk事件
        if (isFailoverEnabled() && Type.NODE_REMOVED == eventType && instanceNode.isInstancePath(path)) {
            String jobInstanceId = path.substring(instanceNode.getInstanceFullPath().length() + 1);
            if (jobInstanceId.equals(JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId())) {
                return;
            }
            log.info("Job instance '{}' with path '{}' crashed! will trigger failover.", jobInstanceId, path);

            List<Integer> failoverItems = failoverService.getFailoverItems(jobInstanceId);
            if (!failoverItems.isEmpty()) {
                for (int each : failoverItems) {
                    // 创建节点 jobName/leader/failover/items/each 
                    failoverService.setCrashedFailoverFlag(each);
                    // 重新调度分片
                    failoverService.failoverIfNecessary();
                }
            } else {
                // 获取分配给该实例的Job分片信息, 设置failover相关节点，进行failover
                // 此处需要理解的是：
                // 1. 每个Job都有该监听器，每个Job会处理自己的failover逻辑
                // 2. 分片后的Job, 每个分片自己处理自己负责分片的failover逻辑
                for (int each : shardingService.getShardItemsForFailover(jobInstanceId)) {
                    failoverService.setCrashedFailoverFlag(each);
                    failoverService.failoverIfNecessary();
                }
            }
        }
    }
}
```

#### ShardingService获取需要重新调度的分片

```java
/**
 * 获取作业运行实例的分片项集合.
 *
 * @param jobInstanceId 作业运行实例主键
 * @return 作业运行实例的分片项集合
*/
public List<Integer> getShardingItems(final String jobInstanceId) {
    JobInstance jobInstance = new JobInstance(jobInstanceId);
    // 判断Job Crash 的实例ip 对应得服务器是否可用
    // 1. 该server是可用状态
    // 2. 该server还有在线的instance
    if (!serverService.isAvailableServer(jobInstance.getIp())) {
        return Collections.emptyList();
    }
    List<Integer> result = new LinkedList<>();
    int shardingTotalCount = configService.load(true).getTypeConfig().getCoreConfig().getShardingTotalCount();
    for (int i = 0; i < shardingTotalCount; i++) {
        if (jobInstance.getJobInstanceId().equals(jobNodeStorage.getJobNodeData(ShardingNode.getInstanceNode(i)))) {
            result.add(i);
        }
    }
    return result;
}

/**
 * 判断作业服务器是否可用.
 * 
 * @param ip 作业服务器IP地址
 * @return 作业服务器是否可用
*/
public boolean isAvailableServer(final String ip) {
    return isEnableServer(ip) && hasOnlineInstances(ip);
}

/**
     * 判断服务器是否启用.
     *
     * @param ip 作业服务器IP地址
     * @return 服务器是否启用
     */
public boolean isEnableServer(final String ip) {
    return !ServerStatus.DISABLED.name().equals(jobNodeStorage.getJobNodeData(serverNode.getServerNode(ip)));
}

private boolean hasOnlineInstances(final String ip) {
    for (String each : jobNodeStorage.getJobNodeChildrenKeys(InstanceNode.ROOT)) {
        if (each.startsWith(ip)) {
            return true;
        }
    }
    return false;
}
```

上面的代码其实是有问题的，具体分析如下：

1. 突然挂掉的Job实例所在的服务器可能是没有Down机的，并且也不会被Disable。
2. 在实践中中，一个server我们只会启动一个Job实例进程，如果该进程挂掉了，那么该Ip上就没有可用的调度实例了。
3. 由1和2可以得出结论，getShardingItems 返回的结果集肯定是空，所有failover功能其实是失效的（除非一个服务器启动多个Job实例）。

这个是在实践中得出的结论，并且[github上有多人反馈这个问题]( https://github.com/elasticjob/elastic-job-lite/issues/669 )，至今未修复。（好死不死，我就是在本机启动2个Job实例来测试failover功能的，没有问题啊）。只能自己fork了一个版本进行修复。



#### FailoverServer 进行failover

```java
/**
* 如果需要失效转移, 则执行作业失效转移.
*/
public void failoverIfNecessary() {
    if (needFailover()) {
        jobNodeStorage.executeInLeader(FailoverNode.LATCH, new FailoverLeaderExecutionCallback());
    }
}

/**
 * 是否需要进行failover
 * @return true 表示需要，false 表示不需要
*/
private boolean needFailover() {
    return jobNodeStorage.isJobNodeExisted(FailoverNode.ITEMS_ROOT) && !jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).isEmpty()
        && !JobRegistry.getInstance().isJobRunning(jobName);
}

class FailoverLeaderExecutionCallback implements LeaderExecutionCallback {

    @Override
    public void execute() {
        if (JobRegistry.getInstance().isShutdown(jobName) || !needFailover()) {
            return;
        }
        // 为什么是get(0)呢？ 原因在于每个分片都在不同的实例上面
        // 但是也有例外啊！ 分片数大于实例数也是可能的。 但是没有理由这么部署。
        int crashedItem = Integer.parseInt(jobNodeStorage.getJobNodeChildrenKeys(FailoverNode.ITEMS_ROOT).get(0));
        log.debug("Failover job '{}' begin, crashed item '{}'", jobName, crashedItem);
        jobNodeStorage.fillEphemeralJobNode(FailoverNode.getExecutionFailoverNode(crashedItem), JobRegistry.getInstance().getJobInstance(jobName).getJobInstanceId());
        jobNodeStorage.removeJobNodeIfExisted(FailoverNode.getItemsNode(crashedItem));
        // TODO 不应使用triggerJob, 而是使用executor统一调度
        JobScheduleController jobScheduleController = JobRegistry.getInstance().getJobScheduleController(jobName);
        if (null != jobScheduleController) {
            jobScheduleController.triggerJob();
        }
    }
}
```



## elastic-job-console 

### 实现

通过Jetty实现的restful API。

### job状态

-  crash ： 如果job没有可用的instance实例，则为此状态
- disabled：如果job所有的server节点保存的值都是`disabled`，则为此状态
- 分片调整中：

### server 操作

#### 获取服务器总个数

##### API

/api/servers/count 

##### 实现逻辑

通过遍历每个Job节点下的`servers`子节点，获取所有的server ip ，进行计数。

#### 获取服务器简明信息

##### API

/api/servers

##### 实现逻辑

遍历根目录，获取去重后的server IP，统计每个server上注册的Job数量，统计每个Job的运行实例个数。

#### 失效server

##### 作用

下线或禁用该服务器节点

##### API

POST /api/servers/10.110.114.42/disable

##### 实现逻辑

将`jobName/servers/ip` 节点的数据设置为`disabled`

#### 启用server

##### 作用

上线线或启用该服务器节点

##### API

DELETE /api/servers/10.110.114.42/disable

##### 实现逻辑

遍历所有job节点，将`jobName/servers/ip` 节点的数据设置为空字符串。 

#### 终止server

##### 作用

下线该服务器节点，意味着所有在该Server节点执行的job实例都不会执行了。

##### API

POST /api/servers/10.110.114.42/shutdown

##### 实现逻辑

遍历所有job节点，删除`jobName/instances/ip@-@pid` 节点。 

#### 清理server

##### 作用

下线该服务器节点，意味着所有在该Server节点执行的job实例都不会执行了。

##### API

DELETE  /api/servers/10.110.114.42

##### 实现逻辑

遍历所有job节点，删除`jobName/servers/ip` 节点。 

#### 获取该服务器下Job注册的简明详情

##### 作用 

获取该服务器上注册的Job的简明信息

##### API

/api/servers/10.110.114.42/jobs?order=asc&offset=0&limit=10&_=1568597724872

##### 实现逻辑

通过遍历每个`job`节点下的`servers`子节点（`jobName/servers/ip`），判断servers子节点的ip是否和参数指定的ip相同，如果相同则命中，构造job的名称，状态(`jobName/servers/ip`节点数据，取值如果是非`disabled`则就是正常)，运行实例（`jobname/instances/ip@-@xxxxxx`）个数信息。

#### 禁用作业

##### 作用 

禁用作业

##### API

POST /api/servers/10.110.114.42/jobs/{jobName}/disable

##### 实现逻辑

将`/jobName/servers`下指定IP的子节点（`jobName/servers/ip`）的数据设置为`disabled`

#### 启用作业

##### 作用 

启用作业

##### API

DELETE /api/servers/10.110.114.42/jobs/{jobName}/disable

##### 实现逻辑

将`/jobName/servers`下指定IP的子节点（`jobName/servers/ip`）的数据设置为空字符串

#### 清理Job

##### 作用

下线该服务器节点，意味着所有在该Server节点执行的job实例都不会执行了。

##### API

DELETE  /api/servers/10.110.114.42/jobs/{jobName}

##### 实现逻辑

删除`jobName/instances/ip@-@pid` 节点。 删除`jobName/servers/ip` 节点。

### Job操作

#### Job 数量

##### 作用

获取Job的数量

##### API

GET /api/jobs/count

##### 实现逻辑

遍历所有`namespace`下的jobName节点，累加。

#### Job 简明信息列表

##### 作用

获取Job简要列表信息

##### API

GET /api/jobs

##### 实现逻辑

遍历所有jobName节点，构造job简明信息。

`jobName/config`节点的数据是json格式的Job配置信息。

#### 触发Job

##### 作用

触发作业立刻执行

##### API

get /api/jobs/{jobName}/trigger

##### 实现

通过将节点`jobName/instances/`下的每个子节点`instanceId`的数据修改为`TRIGGER`来触发job的执行。 

因为执行job的机器会监听`jobName/instances/instanceId`节点的数据变化，然后通过quartz API来触发job的执行。前提是job当前没有执行。后续的版本可能改为堆积式执行。



#### GET 修改job信息

##### API

get /api/jobs/config/commonDutAutoRenewSmsRemindElasticJob

##### 实现



#### PUT 修改job信息

##### API

PUT /api/jobs/config

##### 实现



#### 获取Job分片信息

##### 作用

获取job的详细分片信息

##### API

GET /api/jobs/{jobName}/sharding

##### 实现逻辑

返回数据如下：

```json
[
    {
        "item": 0,
        "serverIp": "10.110.27.147",
        "instanceId": "8535",
        "status": "SHARDING_FLAG",
        "failover": false
    }
]
```



遍历`jobName/sharding`节点下的所有子节点(名称从0开始)，获取每个分片执行实例`jobName/sharding/0/instance `   的实例id,

`jobName/sharding/0/disabled` 是否禁用，`jobName/sharding/0/running` 是否执行。

判断节点`jobName/instances/instanceId` 是否存，不存在表示分片error。

failover 值的判断逻辑为：判断节点`jobName/sharding/0/failover`是否存，存在表示可以failover。

status 值的判断逻辑为：如果`jobName/sharding/0/disabled`节点存在，则为`DISABLED`的。

如果`jobName/sharding/0/running`节点存在，则为`RUNNING`的。

如果`jobName/instances/instanceId` `节点不存在，则为`SHARDING_FLAG`的，表示需要分片。

其它情况表示待调度执行。