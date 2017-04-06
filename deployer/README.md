# com.alibaba.otter.canal.deployer.CanalLauncher

    public static void main(String[] args) throws Throwable
        创建new Properties()实例并加载canal.conf系统属性对应的文件，设置给properties变量       // canal.conf系统属性默认等于classpath:canal.properties
        设置controller变量等于new CanalController(properties)实例
        执行controller.start()
        添加关闭钩子
            执行controller.stop()

# com.alibaba.otter.canal.deployer.CanalController

    static ServerRunningData                    serverData;
    static Map<String, ServerRunningMonitor>    runningMonitors

    public CanalController(Properties properties)
        创建new InstanceConfig()实例，设置给globalInstanceConfig属性                                                  // 初始化全局Config
            使用properties参数中canal.instance.global.mode属性值，设置mode属性
            使用properties参数中canal.instance.global.lazy属性值，设置lazy属性
            使用properties参数中canal.instance.global.spring.xml属性值，设置springXml属性
        设置instanceConfigs属性等于new ConcurrentHashMap<String, InstanceConfig>()实例                               // 初始化默认启动destination
        获取properties参数中canal.destinations属性值，使用,分隔，并进行遍历
            创建new InstanceConfig(globalInstanceConfig)实例，添加destination和其到instanceConfigs属性中
                使用properties参数中canal.instance.{destination}.mode属性值，设置mode属性
                使用properties参数中canal.instance.{destination}.lazy属性值，设置lazy属性
                使用properties参数中canal.instance.{destination}.spring.xml属性值，设置springXml属性
        执行CanalServerWithEmbedded.instance().setInstanceConfigs(instanceConfigs)

        使用properties参数中canal.zkServers属性值，设置给zkServers变量，如果变量不等于null，创建ZK客户端，设置给zkclientx属性
            使用ZK客户端创建/otter/canal/destinations永久节点，设置节点值等于true
            使用ZK客户端创建/otter/canal/cluster永久节点，设置节点值等于true

        使用properties参数中canal.id属性值，设置cid属性
        使用properties参数中canal.ip属性值，设置ip属性，如果属性ip等于null，使用本机地址作为ip属性
        使用properties参数中canal.port属性值，设置port属性
        设置serverData静态变量等于new ServerRunningData(cid, ip + ":" + port)实例
        初始化runningMonitors静态变量            // Map<String, ServerRunningMonitor>，类似LoadingCache，key不存在，自动根据key创建实例
            public ServerRunningMonitor apply(final String destination)
                创建new ServerRunningMonitor(serverData)实例，设置给runningMonitor变量
                执行runningMonitor.setDestination(destination)
                执行runningMonitor.setZkClient(zkclientx)
                创建new ServerRunningListener()实例，实现相关方法，设置给listener变量，并执行runningMonitor.setListener(listener)
                    public void processActiveEnter()
                        执行CanalServerWithEmbedded.instance().start(destination)
                    public void processActiveExit()
                        执行CanalServerWithEmbedded.instance().stop(destination)
                    public void processStart()
                        如果zkclientx属性不等于null
                            创建临时节点/otter/canal/destinations/{destination}/cluster/{ip:port}
                            添加ZK状态监听器，当会话过期创建新的会话时，创建临时节点/otter/canal/destinations/{destination}/cluster/{ip:port}
                    public void processStop()
                        如果zkclientx属性不等于null
                            删除临时节点/otter/canal/destinations/{destination}/cluster/{ip:port}

        使用properties参数中canal.auto.scan属性值，设置autoScan属性
        如果autoScan属性等于true
            创建InstanceAction类型的实例，设置给defaultAction变量
                public void start(String destination)                                                           // 运行时检测到新destination，执行启动
                    获取instanceConfigs属性中destination对应的实例，设置给config变量
                    如果config变量等于null，创建new InstanceConfig(globalInstanceConfig)实例，添加destination和config到instanceConfigs属性中
                        使用properties参数中canal.instance.{destination}.mode属性值，设置mode属性
                        使用properties参数中canal.instance.{destination}.lazy属性值，设置lazy属性
                        使用properties参数中canal.instance.{destination}.spring.xml属性值，设置springXml属性
                    如果config.lazy等于false
                        执行runningMonitors.get(destination).start()
                public void stop(String destination)                                                            // 运行时检测到文件不存在或者目录被删除，执行停止
                    从instanceConfigs属性中移除destination，如果返回的实例不等于null
                        执行CanalServerWithEmbedded.instance().stop(destination)
                        执行runningMonitors.get(destination).stop()
                public void reload(String destination)                                                          // 运行时检测到文件内容发生变化
                    执行stop(destination)
                    执行start(destination)
            初始化instanceConfigMonitors属性     // Map<InstanceMode, InstanceConfigMonitor>，类似LoadingCache，key不存在，自动根据key创建实例
                public InstanceConfigMonitor apply(InstanceMode mode)
                    如果config.mode等于SPRING                                                                  // Spring方式，定时识别目录变化，控制启动和停止
                        使用properties参数中canal.auto.scan.interval属性值，设置scanInterval变量
                        使用properties参数中canal.conf.dir属性值，设置rootDir变量
                        SpringInstanceConfigMonitor monitor = new SpringInstanceConfigMonitor();
                        monitor.setScanIntervalInSecond(scanInterval);
                        monitor.setDefaultAction(defaultAction);
                        monitor.setRootConf(rootDir)
                        return monitor
                    否则，抛异常UnsupportedOperationException(unknow mode)
    public void start() throws Throwable
        如果zkclientx属性不等于null
            创建/otter/canal/cluster/{ip:port}临时节点
            添加ZK状态监听器，当会话过期创建新的会话时，创建/otter/canal/cluster/{ip:port}临时节点
        执行CanalServerWithEmbedded.instance().start()
        遍历instanceConfigs属性
            设置destination变量等于entry.key，config变量等于entry.value
            如果config.lazy等于false
                执行runningMonitors.get(destination).start()
            如果autoScan属性等于true
                执行instanceConfigMonitors.get(config.mode).register(destination, defaultAction)
        如果autoScan属性等于true
            执行instanceConfigMonitors.get(globalInstanceConfig.mode).start()
            遍历instanceConfigMonitors属性
                执行monitor.start()
    public void stop() throws Throwable
        执行CanalServerWithEmbedded.instance().stop()
        如果autoScan属性等于true
            遍历instanceConfigMonitors属性
                执行monitor.stop()
        遍历runningMonitors
           执行runningMonitor.stop()
        删除/otter/canal/cluster/{ip:port}临时节点

## com.alibaba.otter.canal.deployer.monitor.SpringInstanceConfigMonitor
    public void setScanIntervalInSecond(long scanIntervalInSecond)
    public void setDefaultAction(InstanceAction defaultAction)
    public void setRootConf(String rootConf)

    public void start()                         // 定时检测指定目录下的所有子目录，进行动态启动、停止、重启操作
        设置running属性等于true
        启动定时任务，时间间隔为scanIntervalInSecond秒
            遍历rootConf目录的子目录（目录名不等于spring），设置destination等于目录名称
                添加destination到currentInstanceNames变量中
                如果目录下包含instance.properties文件且actions.containsKey(destination)等于false
                    执行defaultAction.start(destination)
                    执行actions.put(destination, defaultAction)
                否则如果actions.containsKey(destination)等于true
                    如果目录下不包含instance.properties文件
                        执行actions.remove(destination).stop(destination)
                    否则判断instance.properties内容是否发生变化，如果发生变化
                        执行actions.get(destination).reload(destination)
            找出actions不存在于currentInstanceNames中的，进行遍历，设置destination变量等于当前元素
                执行actions.remove(destination).stop(destination)
    public void stop()
        设置running属性等于false
        关闭定时任务
        执行actions.clear()
    public void register(String destination, InstanceAction action)
        如果action等于null，设置action参数等于defaultAction属性，添加destination参数和action参数到actions属性中

# com.alibaba.otter.canal.common.zookeeper.running.ServerRunningMonitor
    public void setDestination(String destination)
    public void setZkClient(ZkClientx zkClient)
    public void setListener(ServerRunningListener listener)

    private BooleanMutex mutex = new BooleanMutex();                // 类似Semaphore，令牌数始终介于0或1，继承AQS，对外暴露setStatus方法，set(false)表示setStatus(0)，set(true)表示setStatus(1)，get表示acquire()

    public ServerRunningMonitor(ServerRunningData serverData)               // 对destination的leader选举，确保只有leader对应的CanalInstance实例的在运行（running等于true），以及保证可用性
        设置serverData属性等于serverData参数
        设置dataListener属性等于new IZkDataListener()实例
            public void handleDataChange(String dataPath, Object data) throws Exception                      // 节点数据发生变化
                将data参数解序列化为ServerRunningData类型，设置给runningData变量
                如果runningData.address不等于serverData.address                                                // 主节点不是当前实例
                    执行mutex.set(false)
                否则如果runningData.active等于false                                                            // 主节点是当前实例，但是释放了（没有相关操作设置active等于false）
                    设置release属性等于true
                    获取/otter/canal/destinations/{destination}/running节点值，解序列化为ServerRunningData类型，设置给eventData变量和activeData属性
                    如果eventData.address等于serverData.address
                        删除/otter/canal/destinations/{destination}/running节点
                        执行mutex.set(false)
                        执行listener.processActiveExit()
                设置activeData属性等于runningData
            public void handleDataDeleted(String dataPath) throws Exception                                  // 节点被删除
                执行mutex.set(false)
                如果release属性等于false并且activeData属性不等于null并且activeData.address等于serverData.address  // 主节点对应的实例执行release()后重新参与选举
                    如果running属性等于true
                        执行mutex.set(false)
                        如果/otter/canal/destinations/{destination}/running节点存在且有节点值，解序列化节点值，设置给activeData属性
                        否则
                            创建/otter/canal/destinations/{destination}/running临时节点，设置节点值等于序列化serverData属性
                            设置activeData属性等于serverData属性
                            执行listener.processActiveEnter()
                            执行mutex.set(true)
                否则                                                                                         // 1.主节点对应的实例执行stop()，主节点被删除，其他节点重新选举；2.主节点的active变成了false，触发节点值改变事件，主节点被删除，原来的主节点重新参与选举（没有相关操作设置active等于false）
                    延迟5秒钟执行任务
                        如果running属性等于true
                            执行mutex.set(false)
                            如果/otter/canal/destinations/{destination}/running节点存在且有节点值，解序列化节点值，设置给activeData属性
                            否则
                                创建/otter/canal/destinations/{destination}/running临时节点，设置节点值等于序列化serverData属性
                                设置activeData属性等于serverData属性
                                执行listener.processActiveEnter()
                                执行mutex.set(true)
    public void start()
        设置running属性等于true
        执行listener.processStart()
        如果zkClient属性不等于null
            执行zkClient.subscribeDataChanges("/otter/canal/destinations/{destination}/running", dataListener)
            执行mutex.set(false)
            如果/otter/canal/destinations/{destination}/running节点存在且有节点值，解序列化节点值，设置给activeData属性
            否则
                创建/otter/canal/destinations/{destination}/running临时节点，设置节点值等于序列化serverData属性
                设置activeData属性等于serverData属性
                执行listener.processActiveEnter()
                执行mutex.set(true)
        否则
            执行listener.processActiveEnter()
    public void stop()
        设置running属性等于false
        如果zkClient属性不等于null
            执行zkClient.unsubscribeDataChanges("/otter/canal/destinations/{destination}/running", dataListener)
            获取/otter/canal/destinations/{destination}/running节点值，解序列化为ServerRunningData类型，设置给eventData变量和activeData属性
            如果eventData.address等于serverData.address
                删除/otter/canal/destinations/{destination}/running节点
                执行mutex.set(false)
                执行listener.processActiveExit()
        否则
            执行listener.processActiveExit()
        执行listener.processStop()
    public void release()                           // 暂无相关代码执行此方法
        如果zkClient属性不等于null
            获取/otter/canal/destinations/{destination}/running节点值，解序列化为ServerRunningData类型，设置给eventData变量和activeData属性
            如果eventData.address等于serverData.address
                删除/otter/canal/destinations/{destination}/running节点
                执行mutex.set(false)
                执行listener.processActiveExit()
        否则
            执行listener.processActiveExit()

# com.alibaba.otter.canal.server.embedded.CanalServerWithEmbedded
    public void setInstanceConfigs(Map<String, InstanceConfig> instanceConfigs)

    public void start()             // instanceConfigs是懒加载，同时start(final String destination)和stop(String destination)由ServerRunningMonitor进行leader选举控制
        设置running属性等于true
        初始化canalInstances               // Map<String, CanalInstance>，类似LoadingCache，key不存在，自动根据key创建实例
            public CanalInstance apply(String destination)
                获取instanceConfigs属性中destination对应的实例，设置给config变量，如果config变量等于null，抛异常CanalServerException(can't find destination)
                如果config.mode等于SPRING
                    创建new ClassPathXmlApplicationContext(config.springXml)实例，设置给beanFactory变量
                    添加canal.instance.destination和destination到系统属性中

                    try {
                        SpringCanalInstanceGenerator instanceGenerator = new SpringCanalInstanceGenerator();
                        instanceGenerator.setBeanFactory(beanFactory);
                        return instanceGenerator.generate(destination)
                    } finally {
                        从系统属性中移除canal.instance.destination
                    }
                否则，抛异常UnsupportedOperationException(unknow mode)
    public void stop()
        设置running属性等于false
        遍历canalInstances属性
            执行entry.value.stop()
    public void start(final String destination)
        执行canalInstances.get(destination).start()
    public void stop(String destination)
        从canalInstances属性中移除destination实例，如果返回的实例不等于null，执行canalInstance.stop()
    public void subscribe(ClientIdentity clientIdentity) throws CanalServerException                        // 远程调用，添加对destination的订阅
        设置canalInstance变量等于canalInstances.get(clientIdentity.destination)
        执行canalInstance.metaManager.start()
        执行canalInstance.metaManager.subscribe(clientIdentity)
        设置position变量等于canalInstance.metaManager.getCursor(clientIdentity)
        如果position变量等于null
            设置position变量等于canalInstance.eventStore.getFirstPosition()                                   // 如果put过，获取最近一次的ack位置
            如果position变量不等于null
                执行canalInstance.metaManager.updateCursor(clientIdentity, position)
        执行canalInstance.subscribeChange(clientIdentity)                                                    // 重置filter（要看底层是否支持）
    public void unsubscribe(ClientIdentity clientIdentity) throws CanalServerException                      // 远程调用，删除对destinatio的订阅
        设置canalInstance变量等于canalInstances.get(clientIdentity.destination)
        执行canalInstance.metaManager.unsubscribe(clientIdentity)
    public Message get(ClientIdentity clientIdentity, int batchSize) throws CanalServerException            // 远程调用，获取事件，订阅和订阅不能做到单组（相同clientId）并行消费，多组消费需要看store是否支持，目前客户端clientId默认为1001
        设置canalInstance变量等于canalInstances.get(clientIdentity.destination)
        如果canalInstance变量等于null或者canalInstance.running属性等于false，抛异常CanalServerException
        如果canalInstance.metaManager.hasSubscribe(clientIdentity)等于false，抛异常CanalServerException
        synchronized (canalInstance) {                                                                      // 保持meta和store一致
            设置positionRanges变量等于canalInstance.metaManager.getLastestBatch(clientIdentity)
            如果positionRanges变量不等于null                                                                  // 上一次get操作会执行ack删除上一次get记录，如果此次不等于null，表示之前执行过getWithoutAck操作后没有执行ack/rollback
                抛异常CanalServerException
            设置start变量等于canalInstance.metaManager.getCursor(clientIdentity)
            设置events变量等于canalInstance.eventStore.tryGet(start, batchSize)
            如果events.events等于null或者个数等于0
                返回new Message(-1, new ArrayList<Entry>())实例
            否则
                设置batchId等于canalInstance.metaManager.addBatch(clientIdentity, events.positionRange)
                执行ack(clientIdentity, batchId)
                返回new Message(batchId, Lists.transform(events.getEvents(), new Function<Event, Entry>() {
                                           public Entry apply(Event input) {
                                               return input.getEntry();
                                           }
                                        }))实例
        }
    public void ack(ClientIdentity clientIdentity, long batchId) throws CanalServerException                // 用于单次get或多次getWithoutAck场景，如果是多次getWithoutAck，则调用ack的顺序应该一致
        设置canalInstance变量等于canalInstances.get(clientIdentity.destination)
        如果canalInstance变量等于null或者canalInstance.running属性等于false，抛异常CanalServerException
        如果canalInstance.metaManager.hasSubscribe(clientIdentity)等于false，抛异常CanalServerException
        设置positionRanges变量等于canalInstance.metaManager.removeBatch(clientIdentity, batchId)
        如果positionRanges变量等于null                                                                        // 此次ack删除必须存在，如果不存在，表示之前执行过ack或get或rollback操作
            抛异常CanalServerException
        如果positionRanges.ack不等于null
            执行canalInstance.metaManager.updateCursor(clientIdentity, positionRanges.ack)
        执行canalInstance.eventStore.ack(positionRanges.end)
    public void rollback(ClientIdentity clientIdentity) throws CanalServerException                         // 远程调用，适合单次或多次getWithoutAck场景
        设置canalInstance变量等于canalInstances.get(clientIdentity.destination)
        如果canalInstance变量等于null或者canalInstance.running属性等于false，抛异常CanalServerException
        如果canalInstance.metaManager.hasSubscribe(clientIdentity)等于false，退出方法
        synchronized (canalInstance) {                                                                      // 保持meta和store一致
            执行canalInstance.metaManager.clearAllBatchs(clientIdentity)                                     // 清空getWithoutAck范围记录
            执行canalInstance.eventStore.rollback()
        }

## com.alibaba.otter.canal.instance.spring.SpringCanalInstanceGenerator
    public void setBeanFactory(BeanFactory beanFactory)

    public CanalInstance generate(String destination)
        如果beanFactory中不包含beanName等于destination的实现，设置destination参数等于"instance"，返回beanName等于destination参数的实现

# 简介

    创建/otter/canal/destinations永久节点，设置节点值等于true
    创建创建/otter/canal/cluster永久节点，设置节点之等于true
    创建/otter/canal/cluster/{ip:port}临时节点
    开启一个定时任务，时间间隔为scanIntervalInSecond，
        检测rootConf目录下的子目录下是否存在新增的instance.properties文件，进行动态启动、停止、重启操作
    创建/otter/canal/destinations/{destination}/cluster/{ip:port}临时节点
    根据/otter/canal/destinations/{destination}/running节点，进行选举，确保只有一台机器作为master（每个destination的所有节点中，只有一个节点的CanalInstance实例在运行，其他节点都作为backup存在）
        选举方式
            启动
                对/otter/canal/destinations/{destination}/running节点添加数据改变监听器

                如果/otter/canal/destinations/{destination}/running节点存在，设置activeData属性等于解序列化的节点值
                否则
                    设置/otter/canal/destinations/{destination}/running节点值等于serverData属性的序列化形式
                    设置activeData属性等于serverData属性
                    启动对应的CanalInstance实例
            停止
                设置running属性等于false
                删除对/otter/canal/destinations/{destination}/running节点添加的数据改变监听器
                如果/otter/canal/destinations/{destination}/running节点值解序列化后等于serverData属性
                    删除/otter/canal/destinations/{destination}/running节点
                    停止对应的CanalInstance实例
            释放
                如果/otter/canal/destinations/{destination}/running节点值解序列化后等于serverData属性
                    删除/otter/canal/destinations/{destination}/running节点
                    停止对应的CanalInstance实例
            监听器
                数据改变
                    设置activeData属性等于解序列化后的数据
                节点删除
                    如果activeData属性等于serverData属性
                        如果/otter/canal/destinations/{destination}/running节点存在，设置activeData属性等于解序列化的节点值
                        否则
                            设置/otter/canal/destinations/{destination}/running节点值等于serverData属性的序列化形式
                            设置activeData属性等于serverData属性
                            启动对应的CanalInstance实例
                    否则
                        延迟5秒钟执行任务
                            如果running属性等于true
                                如果/otter/canal/destinations/{destination}/running节点存在，设置activeData属性等于解序列化的节点值
                                否则
                                    设置/otter/canal/destinations/{destination}/running节点值等于serverData属性的序列化形式
                                    设置activeData属性等于serverData属性
                                    启动对应的CanalInstance实例


    客户端可以对CanalInstance进行订阅和解订阅
    客户端可以对CanalInstance执行get、getWithoutAck/ack/rollback操作
        get操作只支持单组消费
        如果执行了多次getWithoutAck，则执行ack的顺序必须一致
        rollback对单次getWithoutAck和多次getWithoutAck场景起作用

    default-instance
        EntryEventSink通常情况下，接收一个完整的事物事件列表，中间可能包含心跳事件，其中的逻辑会过滤掉心跳事件，并存储到eventStore中
        MemoryEventStoreWithBuffer: 类似RingBuffer机制，
        PeriodMixedMetaManager/ZooKeeperMetaManager用于控制client端的offset，默认情况下，每个一秒钟检查是否发生变化，然后更新一次offset
        MysqlDetectingTimeTask/HeartBeatHAController控制是否进行主从库切换
        EventTransactionBuffer用于聚合单挑事件成一个事物的作用，依赖于bufferSize，如果一个事件的长度大于bufferSize，则无法一次聚合
        FailbackLogPositionManager/MemoryLogPositionManager/MetaLogPositionManager用于记录每个事物的结束位置，主要靠内存
        MysqlEventParser:
            内部启动一个单独的线程，根据已有配置信息获取startPosition，然后对mysql执行dump请求，并解析binlog成事件，添加到transactionBuffer中

