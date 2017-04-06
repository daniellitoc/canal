# 参考文档

    http://dev.mysql.com/doc/internals/en/client-server-protocol.html
    http://dev.mysql.com/doc/internals/en/binary-log.html

# com.alibaba.otter.canal.common.alarm.LogAlarmHandler

    public void sendAlarm(String destination, String msg)                               // 打印报警信息到日志
        记录日志destination:{destination}[{msg}]

# com.alibaba.otter.canal.common.zookeeper.ZkClientx

    public static ZkClientx getZkClient(String servers)                                 // 参数：canal.zkServers，默认值：127.0.0.1:2181                            // 创建ZK客户端。

# com.alibaba.otter.canal.sink.entry.EntryEventSink
    public void setEventStore(CanalEventStore<Event> eventStore)                        // com.alibaba.otter.canal.store.memory.MemoryEventStoreWithBuffer

    public EntryEventSink()
        添加new HeartBeatEntryEventHandler()实例到handlers属性中
    public void start()
        设置running属性等于true
        遍历handlers属性
            执行handler.start()
    public void stop()
        设置running属性等于false
        遍历handlers属性
            执行handler.stop()
    public void interrupt()
    public boolean sink(List<CanalEntry.Entry> entrys, InetSocketAddress remoteAddress, String destination) throws CanalSinkException, InterruptedException     // 添加事件
        设置events变量等于new ArrayList<Event>()实例
        遍历entrys参数
            创建new Event(new LogIdentity(remoteAddress, -1L), entry)实例，设置给event变量，并添加到events变量中
            如果entry.entryType等于ROWDATA，设置hasRowData变量等于true
            如果entry.entryType等于HEARTBEAT，设置hasHeartBeat变量等于true
        如果hasRowData变量等于true或者hasHeartBeat变量等于true
            返回doSink(events)
        否则
            如果events元素个数大于0
                // 事件类型包含4中：TRANSACTIONBEGIN、ROWDATA、TRANSACTIONEND、HEARTBEAT，如果hasRowData和hasHeartBeat等于false，则events中只包含TRANSACTIONBEGIN和TRANSACTIONEND类型的事件
                // 基于上一次事件的执行时间和当前时间差是否大于5 * 1000或最近空事物个数是否大于8192，来决定是否放过一次空的事务头和尾
                设置currentTimestamp变量等于events.get(0).entry.header.executeTime
                如果currentTimestamp和lastEmptyTransactionTimestamp的差值后的绝对值大于5 * 1000或者lastEmptyTransactionCount++大于8192
                    设置lastEmptyTransactionCount属性等于0
                    设置lastEmptyTransactionTimestamp属性等于currentTimestamp变量
                    返回doSink(events)
            返回true
    protected boolean doSink(List<Event> events)                                        // 过滤心跳事件以及确保eventStore.tryPut(events)成功
        遍历handlers属性
            设置events参数等于handler.before(events)
        do {
            如果eventStore.tryPut(events)返回true
                遍历handlers属性
                    设置events参数等于handler.after(events)
                返回true
            否则
                睡眠10ms
            遍历handlers属性
                设置events参数等于handler.retry(events)
        } while (running && !Thread.interrupted());
        返回false

## com.alibaba.otter.canal.sink.entry.HeartBeatEntryEventHandler

     public void start()
     public void stop()
     public List<Event> before(List<Event> events)                                      // 过滤心跳事件
        过滤掉events中event.entry.entryType等于HEARTBEAT的Event类型实例
        返回过滤后的列表
     public List<Event> retry(List<Event> events)
        返回events
     public List<Event> after(List<Event> events)
        返回events

## com.alibaba.otter.canal.store.memory.MemoryEventStoreWithBuffer
    public void setBufferSize(int bufferSize)                                           // 参数：canal.instance.memory.buffer.size，默认值：16384
    public void setBufferMemUnit(int bufferMemUnit)                                     // 参数：canal.instance.memory.buffer.memunit，默认值：1024
    public void setBatchMode(BatchMode batchMode)                                       // 参数：canal.instance.memory.batch.mode，默认值：MEMSIZE
    public void setDdlIsolation(boolean ddlIsolation)                                   // 参数：canal.instance.get.ddl.isolation，默认值：false。是否将ddl单条返回

    public MemoryEventStoreWithBuffer()                                                 // 实现上类似于RingBuffer，是纯内存方式的存储，目前的api不支持多组（不同的clientId）消费（需要共用get和ack）

    public void start() throws CanalStoreException
        设置entries属性等于new Event[bufferSize]
    public void stop() throws CanalStoreException
        设置entries属性等于null
    public boolean tryPut(List<Event> data) throws CanalStoreException
        如果data等于null或者个数等于0，返回true
        根据当前发送者的位置，判断距离消费者位置是否有data.size()个空位可插入，如果没有，返回false
        否则
            遍历data，插入event到entries中
            返回true
    public Events<Event> get(Position start, int batchSize, long timeout, TimeUnit unit) throws InterruptedException, CanalStoreException
        根据当前消费者位置，判断距离发送者的位置是否有batchSize个非空位可读取，如果没有，最多等待unit.toNanos(timeout)纳秒，如果中间有put操作，重新检测
        如果有足够非空位可读取
            遍历读取到的event事件列表，如果ddlIsolation属性等于true并且isDdl(entry.header.eventType)等于true
                如果当前event事件是列表中第一个，则返回只包含当前元素的Events实例
                否则，返回当前事件之前的所有event事件
            返回读取到的event实例列表
        如果达到超时，有多少取多少并返回
    public void ack(Position position)
        设置消费者等于新的地址
    public void rollback()
        设置消费者位置等于上次消费的位置
    private boolean isDdl(EventType type)
        如果类型等于ALTER、CREATE、ERASE、RENAME、TRUNCATE、CINDEX、DINDEX，返回true，否则返回false
    public LogPosition getFirstPosition() throws CanalStoreException                    // 获取最近一次的ack位置
        如果之前put过元素，且未ack过，返回第一个Event的LogPosition
        如果之前put过元素，且ack过，但是ack小于put，返回ack位置 + 1的Event的LogPosition
        如果之前put过元素，且ack过，且ack等于put，返回ack位置的Event的LogPosition
        如果之前未put过，返回null
    public LogPosition getLatestPosition() throws CanalStoreException                   // 获取最近一次的put位置
        如果之前put过元素，返回put位置的Event的LogPosition
        如果之前未put过，返回null

# com.alibaba.otter.canal.instance.spring.CanalInstanceWithSpring
    public void setDestination(String destination)                                      // 参数：canal.instance.destination
    public void setEventSink(CanalEventSink<List<CanalEntry.Entry>> eventSink)          // com.alibaba.otter.canal.sink.entry.EntryEventSink
    public void setEventStore(CanalEventStore<Event> eventStore)                        // com.alibaba.otter.canal.store.memory.MemoryEventStoreWithBuffer
    public void setAlarmHandler(CanalAlarmHandler alarmHandler)                         // com.alibaba.otter.canal.common.alarm.LogAlarmHandler
    public void setEventParser(CanalEventParser eventParser)                            // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser
    public void setMetaManager(CanalMetaManager metaManager)                            // com.alibaba.otter.canal.meta.PeriodMixedMetaManager

    public void start()
        执行metaManager.start()
        执行alarmHandler.start()
        执行eventStore.start()
        执行eventSink.start()
        如果eventParser属性是AbstractEventParser类型
            执行((AbstractEventParser) eventParser).getLogPositionManager().start()
        如果eventParser属性是MysqlEventParser类型
            设置haController变量等于((MysqlEventParser) eventParser).getHaController()
            如果haController变量是HeartBeatHAController类型
                执行((HeartBeatHAController) haController).setCanalHASwitchable((MysqlEventParser) eventParser)
            执行haController.start()
        执行eventParser.start()
        设置clientIdentitys变量等于metaManager.listAllSubscribeInfo(destination)，并进行遍历    // 返回的是排好序的（从小到大），因此这里实际依赖于最新的有filter的ClientIdentify
            执行subscribeChange(clientIdentity)
    public boolean subscribeChange(ClientIdentity identity)                             // 有新的订阅，可能改变filter（还要看底层支不支持，按现在理解的配置，不支持），用来过滤库/表
        如果identity.filter属性不等于null
            执行((AbstractEventParser) eventParser).setEventFilter(new AviaterRegexFilter(identity.filter))
    public void stop()
        执行eventParser.stop()
        如果eventParser属性是AbstractEventParser类型
            执行((AbstractEventParser) eventParser).getLogPositionManager().stop()
        如果eventParser属性是MysqlEventParser类型
            执行(((MysqlEventParser) eventParser).getHaController()).stop()
        执行eventSink.stop()
        执行eventStore.stop()
        执行metaManager.stop()
        执行alarmHandler.stop()

## com.alibaba.otter.canal.meta.PeriodMixedMetaManager
    public void setPeriod(long period)                                                  // 参数：canal.zookeeper.flush.period，默认值：1000
    public void setZooKeeperMetaManager(ZooKeeperMetaManager zooKeeperMetaManager)      // com.alibaba.otter.canal.meta.ZooKeeperMetaManager

    public void start()
        执行zooKeeperMetaManager.start()
        初始化destinations属性                       // Map<String, List<ClientIdentity>>，类似LoadingCache，key不存在，自动根据key创建实例。    维护订阅关系
            public List<ClientIdentity> apply(String destination)
                返回zooKeeperMetaManager.listAllSubscribeInfo(destination)
        初始化cursors属性                            // Map<ClientIdentity, Position>，类似LoadingCache，key不存在，自动根据key创建实例          维护消费位置
            public Position apply(ClientIdentity clientIdentity)
                返回zooKeeperMetaManager.getCursor(clientIdentity)
        初始化batches属性                            // Map<ClientIdentity, MemoryClientIdentityBatch>，类似LoadingCache，key不存在，自动根据key创建实例
            public MemoryClientIdentityBatch apply(ClientIdentity clientIdentity)
                设置batches变量等于MemoryClientIdentityBatch.create(clientIdentity)
                设置positionRanges变量等于zooKeeperMetaManager.listAllBatchs(clientIdentity)，并进行遍历
                    执行batches.addPositionRange(entry.getValue(), entry.getKey())
                返回batches变量
        设置updateCursorTasks属性等于Collections.synchronizedSet(new HashSet<ClientIdentity>())                                           // 维护消费位置
        启动定时任务，时间间隔为period毫秒                                                                                                   // 维护消费位置
            创建updateCursorTasks属性快照，并进行遍历
                执行zooKeeperMetaManager.updateCursor(clientIdentity, cursors.get(clientIdentity))
                执行updateCursorTasks.remove(clientIdentity)
    public void stop()
        执行cursors.clear()
        遍历batches属性
            执行entry.value().clearPositionRanges()
        执行zooKeeperMetaManager.stop()
        停止定时任务
        执行destinations.clear()
        执行batches.clear()
    public synchronized List<ClientIdentity> listAllSubscribeInfo(String destination) throws CanalMetaManagerException                  // 维护订阅关系
        返回destinations.get(destination)
    public void subscribe(final ClientIdentity clientIdentity) throws CanalMetaManagerException                                         // 维护订阅关系
        设置clientIdentitys变量等于destinations.get(clientIdentity.destination)
        如果clientIdentitys.contains(clientIdentity)等于true
            执行clientIdentitys.remove(clientIdentity)
        执行clientIdentitys.add(clientIdentity)
        异步执行zooKeeperMetaManager.subscribe(clientIdentity)
    public void unsubscribe(final ClientIdentity clientIdentity) throws CanalMetaManagerException                                       // 维护订阅关系
        设置clientIdentitys变量等于destinations.get(clientIdentity.destination)
        如果clientIdentitys不等于null并且clientIdentitys.contains(clientIdentity)等于true
            执行clientIdentitys.remove(clientIdentity)
        异步执行zooKeeperMetaManager.unsubscribe(clientIdentity)
    public Position getCursor(ClientIdentity clientIdentity) throws CanalMetaManagerException                                           // 维护消费位置
        返回cursors.get(clientIdentity)
    public void updateCursor(ClientIdentity clientIdentity, Position position) throws CanalMetaManagerException                         // 维护消费位置
        执行updateCursorTasks.add(clientIdentity)
        执行cursors.put(clientIdentity, position)
    public PositionRange getLastestBatch(ClientIdentity clientIdentity) throws CanalMetaManagerException                                // 获取最近一次的getWithoutAck执行记录
        返回batches.get(clientIdentity).getLastestPositionRange()
    public Long addBatch(ClientIdentity clientIdentity, PositionRange positionRange) throws CanalMetaManagerException                   // 添加get或getWithoutAck执行记录
        执行batches.get(clientIdentity).addPositionRange(positionRange)
    public PositionRange removeBatch(ClientIdentity clientIdentity, Long batchId) throws CanalMetaManagerException                      // 删除get或getWithoutAck执行记录，需要验证batchId是最久的
        返回batches.get(clientIdentity).removePositionRange(batchId)
    public void clearAllBatchs(ClientIdentity clientIdentity) throws CanalMetaManagerException                                          // 清理getWithoutAck执行记录
        执行batches.get(clientIdentity).clearPositionRanges()

### com.alibaba.otter.canal.meta.ZooKeeperMetaManager
    public void setZkClientx(ZkClientx zkClientx)                                       // com.alibaba.otter.canal.common.zookeeper.ZkClientx

    public void start()
    public void stop()
    public List<ClientIdentity> listAllSubscribeInfo(String destination) throws CanalMetaManagerException       // 返回destination的所有订阅者（目前客户端只有一个clientId等于1001的），如果有多个，订阅者之间依赖clientId自增来保证新旧
        获取/otter/canal/destinations/{destination}节点的所有子节点
        如果没有子节点，返回new ArrayList<ClientIdentity>()
        否则
            设置clientIdentities变量等于Lists.newArrayList()
            将所有子节点转换为Short类型，并按照从小到大进行排序，设置给clientIds变量，并进行遍历
                获取/otter/canal/destinations/{destination}/{clientId}/filter节点值
                如果节点值不等于null
                    执行clientIdentities.add(new ClientIdentity(destination, clientId, filter))
                否则
                    执行clientIdentities.add(new ClientIdentity(destination, clientId, null))
            返回clientIdentities变量
    public Position getCursor(ClientIdentity clientIdentity) throws CanalMetaManagerException                   // 获取clientIdentity消费位置
        获取/otter/canal/destinations/{clientIdentity.destination}/{clientIdentity.clientId}/cursor节点值
        如果节点值不等于null
            将节点值解序列化为Position类型并返回
        返回null
    public void updateCursor(ClientIdentity clientIdentity, Position position) throws CanalMetaManagerException // 更新clientIdentity消费位置
        设置/otter/canal/destinations/{clientIdentity.destination}/{clientIdentity.clientId}/cursor节点值等于position参数序列化后的byte[]
    public void subscribe(ClientIdentity clientIdentity) throws CanalMetaManagerException                       // 新增订阅
        创建/otter/canal/destinations/{clientIdentity.destination}/{clientIdentity.clientId}节点
        如果clientIdentity.filter不等于null
            设置/otter/canal/destinations/{clientIdentity.destination}/{clientIdentity.clientId}/filter节点值等于clientIdentity.filter序列化后的byte[]
    public void unsubscribe(ClientIdentity clientIdentity) throws CanalMetaManagerException                     // 解除订阅
        删除/otter/canal/destinations/{clientIdentity.destination}/{clientIdentity.clientId}节点
    public Map<Long, PositionRange> listAllBatchs(ClientIdentity clientIdentity)                                // 目前没有相关调用添加mark节点
        获取/otter/canal/destinations/{clientIdentity.destination}/{clientIdentity.clientId}/mark节点的所有子节点
        如果没有子节点，返回Maps.newHashMap()
        否则
            设置positionRanges变量等于Maps.newLinkedHashMap()
            将所有子节点转换为Long类型，并按照从小到大进行排序，设置给batchIds变量，并进行遍历
                获取/otter/canal/destinations/{clientIdentity.destination}/{clientIdentity.clientId}/mark/{batchId}节点值
                如果节点值等于null
                    重新执行listAllBatchs(clientIdentity)并返回
                否则
                    添加batchId和将节点值解序列化为PositionRange类型的实例到positionRanges变量
            返回positionRanges变量

### com.alibaba.otter.canal.meta.MemoryMetaManager.MemoryClientIdentityBatch
    // 用于维护某个clientIdentity的get/ack/rollback操作的范围记录，主要针对getWithoutAck + ack场景有起到检测的意义，需要保证顺序依次执行，支持多getWithoutAck + 多ack，单次get场景无实际意义

## com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser
    public void setSlaveId(long slaveId)                                                        // 参数：canal.instance.mysql.slaveId，默认值：1234             // 模拟MySQL从库Connection的slaveId
    public void setReceiveBufferSize(int receiveBufferSize)                                     // 参数：canal.instance.network.receiveBufferSize，默认值：16384    // MysqlConnection的接收buffer大小
    public void setSendBufferSize(int sendBufferSize)                                           // 参数：canal.instance.network.sendBufferSize，默认值：16384   // MysqlConnection的发送buffer大小
    public void setDefaultConnectionTimeoutInSeconds(int defaultConnectionTimeoutInSeconds)     // 参数：canal.instance.network.soTimeout，默认值：30           // MysqlConnection的执行操作的超时时间

### com.alibaba.otter.canal.parse.inbound.mysql.MysqlConnection
    public void setCharset(Charset charset)
    public void setSlaveId(long slaveId)

    public MysqlConnection(InetSocketAddress address, String username, String password, byte charsetNumber, String defaultSchema)
        设置connector属性等于new MysqlConnector(address, username, password, charsetNumber, defaultSchema)实例

    public MysqlConnection fork()
        设置connection变量等于new MysqlConnection()实例
        执行connection.setCharset(charset);
        执行connection.setSlaveId(slaveId);
        执行connection.setConnector(connector.fork());
        返回connection变量
    public void reconnect() throws IOException
        执行connector.reconnect()
    public boolean isConnected()
        返回connector.isConnected()
    public void connect() throws IOException
        执行connector.connect()
    public void disconnect() throws IOException
        执行connector.disconnect()
    public ResultSetPacket query(String cmd) throws IOException
        返回new MysqlQueryExecutor(connector).query(cmd)
    public void update(String cmd) throws IOException
        返回new MysqlUpdateExecutor(connector).update(cmd)
    public BinlogFormat getBinlogFormat()                                                       // STATEMENT/ROW/MIXED（http://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_binlog_format）
        如果binlogFormat属性等于null
            执行query("show variables like 'binlog_format'")返回ResultSetPacket实例               // 假设为ROW
            验证必须包含2个列值
            获取第二个列值，处理成BinlogFormat枚举类型，设置给binlogFormat属性
        返回binlogFormat属性
    public BinlogImage getBinlogImage()                                                         // FULL/MINIMAL/NOBLOB（http://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_binlog_row_image）
        如果binlogImage属性等于null
            执行query("show variables like 'binlog_row_image'")返回ResultSetPacket实例            // 默认为FULL
            验证必须包含2个列值
            获取第二个列值，处理成BinlogImage枚举类型，设置给binlogImage属性
        返回binlogImage属性
    public void dump(String binlogfilename, Long binlogPosition, SinkFunction func) throws IOException                                                          // 解码MySQL全部事件（http://dev.mysql.com/doc/internals/en/binlog-event-type.html、http://dev.mysql.com/doc/internals/en/event-classes-and-types.html）及MariaDB全部事件
        执行update("set wait_timeout=9999999")
        执行update("set net_write_timeout=1800")
        执行update("set net_read_timeout=1800")
        执行update("set names 'binary'")
        执行update("set @master_binlog_checksum= '@@global.binlog_checksum'")
        执行update("SET @mariadb_slave_capability='4'")

        根据binlogfilename、binlogPosition、slaveId信息创建COM_BINLOG_DUMP（http://dev.mysql.com/doc/internals/en/com-binlog-dump.html）
        根据COM_BINLOG_DUMP生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html），发送Packets给channel
        执行connector.setDumping(true)

        设置fetcher变量等于new DirectLogFetcher(connector.receiveBufferSize)
        执行fetcher.start(connector.getChannel())
        设置decoder变量等于new LogDecoder(0, 164)             // 0: UNKNOWN_EVENT   164: ENUM_END_EVENT
        设置context变量等于new LogContext()
        while (fetcher.fetch()) {                                                                          // false结束，异常抛出，true继续
            执行decoder.decode(fetcher, context)返回LogEvent类型的实例，设置给event变量
            如果event变量等于null，抛异常CanalParseException(parse failed);
            执行func.sink(event)，如果返回false
                退出循环
        }
    public void dump(long timestamp, SinkFunction func) throws IOException
        抛异常NullPointerException(Not implement yet)
    public void seek(String binlogfilename, Long binlogPosition, SinkFunction func) throws IOException                                                          // 解码以下4种事件，其他事件全部按照UnknownEvent处理
        执行update("set wait_timeout=9999999")
        执行update("set net_write_timeout=1800")
        执行update("set net_read_timeout=1800")
        执行update("set names 'binary'")
        执行update("set @master_binlog_checksum= '@@global.binlog_checksum'")
        执行update("SET @mariadb_slave_capability='4'")

        根据binlogfilename、binlogPosition、slaveId信息创建COM_BINLOG_DUMP（http://dev.mysql.com/doc/internals/en/com-binlog-dump.html）
        根据COM_BINLOG_DUMP生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html），发送Packets给channel
        执行connector.setDumping(true)

        设置fetcher变量等于new DirectLogFetcher(connector.receiveBufferSize)
        执行fetcher.start(connector.getChannel())
        设置decoder变量等于new LogDecoder()
        执行decoder.handle(4);                                // ROTATE_EVENT
        执行decoder.handle(15);                               // FORMAT_DESCRIPTION_EVENT
        执行decoder.handle(2);                                // QUERY_EVENT
        执行decoder.handle(16);                               // XID_EVENT
        设置context变量等于new LogContext()
        while (fetcher.fetch()) {                                                                           // false结束，异常抛出，true继续
            执行decoder.decode(fetcher, context)返回LogEvent类型的实例，设置给event变量
            如果event变量等于null，抛异常CanalParseException(parse failed);
            执行func.sink(event)，如果返回false
                退出循环
        }

#### com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector
    public void setReceiveBufferSize(int receiveBufferSize)
    public void setSendBufferSize(int sendBufferSize)
    public void setSoTimeout(int soTimeout)
    public void setDumping(boolean dumping)

    public MysqlConnector(InetSocketAddress address, String username, String password, byte charsetNumber, String defaultSchema)
        赋值参数到各个属性
    public MysqlConnector fork()
        设置connector变量等于new MysqlConnector()实例
        执行connector.setCharsetNumber(charsetNumber);
        执行connector.setDefaultSchema(defaultSchema);
        执行connector.setAddress(address);
        执行connector.setPassword(password);
        执行connector.setUsername(username);
        执行connector.setReceiveBufferSize(receiveBufferSize);
        执行connector.setSendBufferSize(sendBufferSize);
        执行connector.setSoTimeout(soTimeout);
        返回connector变量
    public void reconnect() throws IOException
        执行disconnect();
        执行connect();
    public boolean isConnected()
        如果channel属性不等于null并且channel.isConnected()等于true，返回true，否则返回false
    public void connect() throws IOException
        如果connected执行cas操作从false变成true成功
            try {
                设置channel属性等于SocketChannel.open()实例
                执行channel.socket().setKeepAlive(true);
                执行channel.socket().setReuseAddress(true);
                执行channel.socket().setSoTimeout(soTimeout);
                执行channel.socket().setTcpNoDelay(true);
                执行channel.socket().setReceiveBufferSize(receiveBufferSize);
                执行channel.socket().setSendBufferSize(sendBufferSize);
                执行channel.connect(address)

                从channel读取字节生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html）
                如果packets.payload的第一个字节是(byte) 0xff（http://dev.mysql.com/doc/internals/en/packet-ERR_Packet.html）
                    根据payload生成ERR_Packet，抛异常IOException(handshake exception...)
                否则如果packets.payload的第一个字节是(byte) 0xfe（http://dev.mysql.com/doc/internals/en/packet-EOF_Packet.html）
                    抛异常IOException(Unexpected EOF packet at handshake phase.)
                否则如果packets.payload的第一个字节小于0
                    抛异常IOException(unpexpected packet with field_count=...)

                根据packets.payload生成Handshake（http://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::Handshake）
                设置connectionId属性等于handshake.threadId

                根据charsetNumber、username、password、defaultSchema等信息创建HandshakeResponse（http://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::HandshakeResponse）
                根据HandshakeResponse生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html），发送Packets给channel

                从channel读取字节生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html）
                验证packets.payload不等于null
                如果packets.payload的第一个字节是(byte) 0xff（http://dev.mysql.com/doc/internals/en/packet-ERR_Packet.html）
                    根据payload生成ERR_Packet，抛异常IOException(Error When doing Client Authentication...)
                否则如果packets.payload的第一个字节是(byte) 0xfe（http://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::AuthSwitchRequest）
                    根据password等信息创建AuthSwitchResponse（http://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::AuthSwitchResponse）
                    根据AuthSwitchResponse生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html），发送Packets给channel
                    从channel读取字节生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html）
                        如果packets.payload的第一个字节是(byte) 0xff（http://dev.mysql.com/doc/internals/en/packet-ERR_Packet.html）
                            根据payload生成ERR_Packet，抛异常IOException(Error When doing Client Authentication...)
                        否则如果packets.payload的第一个字节不是0x00（http://dev.mysql.com/doc/internals/en/packet-OK_Packet.html）
                            throw new IOException(unpexpected packet with field_count=...);
                否则如果packets.payload的第一个字节小于0
                    抛异常IOException(unpexpected packet with field_count=...)
            } catch (Exception e) {
                执行disconnect()
                抛异常IOException(connect...);
            }
    public void disconnect() throws IOException
        如果connected执行cas操作从true变成false成功
            try {
                如果channel属性不等于null，执行channel.close()
            } catch (Exception e) {
                抛异常IOException(disconnect...);
            }
            如果dumping等于true并且connectionId大于0
                MysqlConnector connector = null;
                try {
                    设置connector变量等于fork()
                    执行connector.connect()
                    执行new MysqlUpdateExecutor(connector).update("KILL CONNECTION " + connectionId);
                } catch (Exception e) {
                    抛异常IOException(KILL DUMP...);
                } finally {
                    如果connector变量不等于null，执行connector.disconnect()
                }
                设置dumping等于false

#### com.alibaba.otter.canal.parse.driver.mysql.MysqlQueryExecutor

    public MysqlQueryExecutor(MysqlConnector connector)
        如果connector.isConnected()等于false，抛异常RuntimeException(should execute connector.connect() first)
        设置channel属性等于connector.channel
    public ResultSetPacket query(String queryString) throws IOException
        创建COM_QUERY（http://dev.mysql.com/doc/internals/en/com-query.html），设置queryString属性等于queryString参数
        根据COM_QUERY生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html），发送Packets给channel
        从channel读取字节生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html）
        如果packets.payload的第一个字节小于0
            根据payload生成ERR_Packet（http://dev.mysql.com/doc/internals/en/packet-ERR_Packet.html），抛异常IOException(...)
        根据packets.payload生成Resultset（http://dev.mysql.com/doc/internals/en/com-query-response.html#packet-ProtocolText::Resultset）并返回

#### com.alibaba.otter.canal.parse.driver.mysql.MysqlUpdateExecutor

    public MysqlUpdateExecutor(MysqlConnector connector)
        如果connector.isConnected()等于false，抛异常RuntimeException(should execute connector.connect() first)
        设置channel属性等于connector.channel
    public OKPacket update(String updateString) throws IOException
        创建COM_QUERY（http://dev.mysql.com/doc/internals/en/com-query.html），设置queryString属性等于updateString参数
        根据COM_QUERY生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html），发送Packets给channel
        从channel读取字节生成Packets（http://dev.mysql.com/doc/internals/en/mysql-packet.html）
        如果packets.payload的第一个字节小于0
            根据payload生成ERR_Packet（http://dev.mysql.com/doc/internals/en/packet-ERR_Packet.html），抛异常IOException(...)
        根据packets.payload生成OK_Packet（http://dev.mysql.com/doc/internals/en/packet-OK_Packet.html）并返回

#### com.alibaba.otter.canal.parse.inbound.mysql.dbsync.DirectLogFetcher

    public DirectLogFetcher(final int initialCapacity)
        设置buffer属性等于new byte[initialCapacity]实例
        设置factor属性等于2.0f
    public void start(SocketChannel channel) throws IOException
        设置channel属性等于channel参数
    public boolean fetch() throws IOException
        // 这里还是看源码吧

#### com.taobao.tddl.dbsync.binlog.LogDecoder
    // 这里还是看源码吧

#### com.taobao.tddl.dbsync.binlog.LogContext
    // 这里还是看源码吧

## com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser
    public void setAlarmHandler(CanalAlarmHandler alarmHandler)                                 // com.alibaba.otter.canal.common.alarm.LogAlarmHandler

    public void setDetectingEnable(boolean detectingEnable)                                     // 参数：canal.instance.detecting.enable，默认值：false         // 是否开启心跳检查
    public void setDetectingSQL(String detectingSQL)                                            // 参数：canal.instance.detecting.sql                         // 心跳检测被执行sql
    public void setDetectingIntervalInSeconds(Integer detectingIntervalInSeconds)               // 参数：canal.instance.detecting.interval.time，默认值：5      // 心跳检查频率
    public void setHaController(CanalHAController haController)                                 // com.alibaba.otter.canal.parse.ha.HeartBeatHAController
    public void setMasterInfo(AuthenticationInfo masterInfo)                                    // 属性：address（canal.instance.master.address）属性：username（canal.instance.dbUsername:retl）属性：password（canal.instance.dbPassword:retl）属性：defaultDatabaseName（canal.instance.defaultDatabaseName:retl）
    public void setStandbyInfo(AuthenticationInfo standbyInfo)                                  // 属性：address（canal.instance.standby.address）属性：username（canal.instance.dbUsername:retl）属性：password（canal.instance.dbPassword:retl）属性：defaultDatabaseName（canal.instance.defaultDatabaseName:retl）
    public void setMasterPosition(EntryPosition masterPosition)                                 // 属性：journalName（canal.instance.master.journal.name）属性：position（canal.instance.master.position）属性：timestamp（canal.instance.master.timestamp）
    public void setStandbyPosition(EntryPosition standbyPosition)                               // 属性：journalName（canal.instance.standby.journal.name）属性：position（canal.instance.standby.position）属性：timestamp（canal.instance.standby.timestamp）
    protected void startHeartBeat(ErosaConnection connection)                                                                                                   // 开启心跳检测任务
        设置lastEntryTime属性等于0
        如果timer属性等于null，创建new Timer(destination + runningInfo.address, true)实例设置给timer属性
        如果heartBeatTimerTask属性等于null
            如果detectingEnable属性等于true并且detectingSQL属性不等于空
                创建new MysqlDetectingTimeTask((MysqlConnection) connection.fork())实例，设置给heartBeatTimerTask属性
            否则
                创建new TimerTask()实例，设置给heartBeatTimerTask属性
                    public void run()
                         try {
                            如果exception属性等于null或者lastEntryTime属性大于0                                                                                     // 不是刚启动就失败场景
                                如果当前时间减去lastEntryTime属性的时间差除以1000，得到的值大于等于detectingIntervalInSeconds属性
                                    Header.Builder headerBuilder = Header.newBuilder();
                                    headerBuilder.setExecuteTime(now);
                                    Entry.Builder entryBuilder = Entry.newBuilder();
                                    entryBuilder.setHeader(headerBuilder.build());
                                    entryBuilder.setEntryType(EntryType.HEARTBEAT);
                                    Entry entry = entryBuilder.build();
                                    执行consumeTheEventAndProfilingIfNecessary(Arrays.asList(entry))                                                            // 仅仅是发个heartbeat事件，无其他特殊处理
                         } catch (Throwable e) {
                         }
            执行timer.schedule(heartBeatTimerTask, detectingIntervalInSeconds * 1000L, detectingIntervalInSeconds * 1000L)
    public void doSwitch()                                                                                                                                     // 切换连接库
        如果runningInfo属性等于masterInfo属性，设置newRunningInfo变量等于standbyInfo属性，否则设置newRunningInfo变量等于masterInfo属性
        如果runningInfo属性和newRunningInfo变量相等                                                                                                               // 如果主库和从库信息一样，不切换
            退出
        否则如果newRunningInfo变量等于null                                                                                                                       // 如果没有配置从库信息，报警
            执行alarmHandler.sendAlarm(destination, "no standby config, just do nothing, will continue try")
        否则                                                                                                                                                   // 替换主从信息，重新启动
            执行stop()
            执行alarmHandler.sendAlarm(destination, "try to ha switch...")
            设置runningInfo属性等于newRunningInfo变量
            执行start()

### com.alibba.otter.canal.parse.inbound.mysql.MysqlEventParser.MysqlDetectingTimeTask

    public MysqlDetectingTimeTask(MysqlConnection mysqlConnection)
        设置mysqlConnection属性等于mysqlConnection参数
        设置reconnect属性等于false
    public void run()                                                   // 执行SQL，如果成功通知haController的onSuccess方法，如果失败，通知haController的onFailed方法
        try
            如果reconnect属性等于true
                reconnect属性等于false
                执行mysqlConnection.reconnect()
            否则如果mysqlConnection.isConnected()等于false
                执行mysqlConnection.connect()
            设置开始时间
            如果detectingSQL以select或show或explain或desc开头
                执行mysqlConnection.query(detectingSQL)
            否则
                执行mysqlConnection.update(detectingSQL)
            统计执行时间，设置给costTime变量
            执行((HeartBeatCallback) haController).onSuccess(costTime)
        catch (Throwable e)
            执行((HeartBeatCallback) haController).onFailed(e)
            设置reconnect属性等于true

### com.alibaba.otter.canal.parse.ha.HeartBeatHAController
    public void setDetectingRetryTimes(int detectingRetryTimes)                         // 参数：canal.instance.detecting.retry.threshold，默认值：3
    public void setSwitchEnable(boolean switchEnable)                                   // 参数：canal.instance.detecting.heartbeatHaEnable，默认值：false
    public void setCanalHASwitchable(CanalHASwitchable canalHASwitchable)               // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser

    public void start()
    public void stop()
    public void onSuccess(long costTime)                                // 重置连续失败次数为0
        设置failedTimes属性等于0
    public void onFailed(Throwable e)                                   // 如果开启了高可用，并且连续失败次数超过detectingRetryTimes，指定parser的doSwitch方法
        设置failedTimes属性自身加1
            如果failedTimes属性大于detectingRetryTimes属性
                如果switchEnable属性等于true
                    执行eventParser.doSwitch()
                    设置failedTimes属性等于0

## com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser
    public void setTransactionSize(int transactionSize)                                         // 参数：canal.instance.transaction.size，默认值：1024
    public void setEventSink(CanalEventSink<List<CanalEntry.Entry>> eventSink)                  // com.alibaba.otter.canal.store.memory.MemoryEventStoreWithBuffer  // 这是通过Spring byName设置的
    public void setLogPositionManager(CanalLogPositionManager logPositionManager)               // com.alibaba.otter.canal.parse.index.FailbackLogPositionManager

    public MysqlEventParser()
        初始化transactionBuffer属性，创建new EventTransactionBuffer(new TransactionFlushCallback() {
                                           public void flush(List<CanalEntry.Entry> transaction) throws InterruptedException {
                                               执行consumeTheEventAndProfilingIfNecessary(transaction)，设置给successed变量
                                               如果running属性等于false，退出
                                               如果successed变量等于false，抛异常CanalParseException(consume failed!)                                               // eventSink执行stop()方法导致不成功

                                               执行buildLastTransactionPosition(transaction)，设置给position变量
                                               如果position变量不等于null                                                                                          // 如果有类型等于TRANSACTIONEND的，记录位置
                                                    执行logPositionManager.persistLogPosition(MysqlEventParser.this.destination, position)
                                           }
                                       })实例

    public void start() throws CanalParseException
        设置runningInfo属性等于masterInfo属性
        设置running属性等于true
        执行transactionBuffer.setBufferSize(transactionSize)
        执行transactionBuffer.start()
    public void stop() throws CanalParseException
        设置running属性等于false
        执行transactionBuffer.stop()
    protected boolean consumeTheEventAndProfilingIfNecessary(List<CanalEntry.Entry> entrys) throws CanalSinkException, InterruptedException                     // 添加entrys到sink
        执行eventSink.sink(entrys, runningInfo.address, destination)，设置给result变量
        如果consumedEventCount.incrementAndGet()小于0
            执行consumedEventCount.set(0)
        返回result变量
    protected LogPosition buildLastTransactionPosition(List<CanalEntry.Entry> entries)                                                                          // 返回类型等于TRANSACTIONEND的位置
        从后往前遍历entries参数
            如果entry.entryType等于TRANSACTIONEND
                执行buildLastPosition(entry)并返回
    protected LogPosition buildLastPosition(CanalEntry.Entry entry)
        设置position变量等于new EntryPosition()实例
        设置position变量的journalName属性等于entry.header.logfileName
        设置position变量的position属性等于entry.header.logfileOffset
        设置position变量的timestamp属性等于entry.header.executeTime

        设置logPosition变量等于new LogPosition()实例
        设置logPosition变量的position属性等于position变量
        设置logPosition变量的identity属性等于new LogIdentity(runningInfo.address, -1L)实例
        返回logPosition

### com.alibaba.otter.canal.parse.inbound.EventTransactionBuffer

    // 针对一次dump解析出的所有事件，起到聚合单条成一个事物的作用，依赖bufferSize，如果bufferSize小于事物个数，可能导致事物开始和事物结束不在一次flush中
    // 非线程安全
    // 添加元素时，如果不是INSERT/UPDATE/DELETE时，执行flush()，如果类型是TRANSACTIONBEGIN，先flush()在添加，如果类型是TRANSACTIONEND，先添加在flush()
    // 其他添加情况根据buffer是否满了（环中，put追上flush）进行flush()
    public EventTransactionBuffer(TransactionFlushCallback flushCallback)

### com.alibaba.otter.canal.parse.index.FailbackLogPositionManager
    public void setPrimary(CanalLogPositionManager primary)                             // com.alibaba.otter.canal.parse.index.MemoryLogPositionManager
    public void setFailback(CanalLogPositionManager failback)                           // com.alibaba.otter.canal.parse.index.MetaLogPositionManager

    public void start()
        执行primary.start()
        执行failback.start()
    public void stop()
        执行primary.stop()
        执行failback.stop()
    public void persistLogPosition(String destination, LogPosition logPosition) throws CanalParseException              // 记录事物结束位置，主失败用从
        try {
            执行primary.persistLogPosition(destination, logPosition);
        } catch (CanalParseException e) {
            执行failback.persistLogPosition(destination, logPosition);
        }
    public LogPosition getLatestIndexBy(String destination)                                                             // 读取上一个事物的结束位置或上一次最旧的ack位置，主不存在用从
        执行primary.getLatestIndexBy(destination)返回实例，设置给logPosition变量
        如果logPosition变量等于null
            返回failback.getLatestIndexBy(destination)
        否则返回logPosition变量

### com.alibaba.otter.canal.parse.index.MemoryLogPositionManager

    public void start()
        设置positions属性等于new ConcurrentHashMap<String, LogPosition>()
    public void stop()
        执行positions.clear()
    public void persistLogPosition(String destination, LogPosition logPosition)                                         // 添加K/V到map中
        添加destination和logPosition到positions属性中
    public LogPosition getLatestIndexBy(String destination)                                                             // 根据K从map中返回
        执行positions.get(destination)并返回

### com.alibaba.otter.canal.parse.index.MetaLogPositionManager
    public void setMetaManager(CanalMetaManager metaManager)                            // com.alibaba.otter.canal.meta.PeriodMixedMetaManager

    public void start()
        执行metaManager.start()
    public void stop()
        执行metaManager.stop()
    public void persistLogPosition(String destination, LogPosition logPosition)                                         // 空实现
    public LogPosition getLatestIndexBy(String destination)                                                             // 返回的是排好序的（从小到大），最终返回最旧的
        设置result变量等于null
        执行metaManager.listAllSubscribeInfo(destination)返回List<ClientIdentity>类型的实例，设置给clientIdentitys变量
        如果clientIdentitys变量不等于null且各数大于0
            遍历clientIdentitys变量
                执行metaManager.getCursor(clientIdentity)返回LogPosition类型的实例，设置给position变量
                如果position变量等于null，继续下一次循环
                如果result变量等于null
                    设置result变量等于position变量
                否则
                    设置result变量等于CanalEventUtils.min(result, position)
        返回result变量

#### com.alibaba.otter.canal.store.helper.CanalEventUtils

    public static LogPosition min(LogPosition position1, LogPosition position2)             // 按名称和位置，以及时间比较，返回小的
        如果position1.identity等于position2.identity
            如果position1.position.journalName大于position2.position.journalName
                返回position2
            否则如果position1.position.journalName小于position2.position.journalName
                返回position1
            否则
                如果position1.position.position大于position2.position.position
                    返回position2
                否则
                    返回position1
        否则
            如果position1.position.timestamp大于position2.position.timestamp
                返回position2
            否则
                返回position1

## com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser
    public void setDestination(String destination)                                              // 参数：canal.instance.destination
    public void setEventFilter(CanalEventFilter eventFilter)                                    // new AviaterRegexFilter(canal.instance.filter.regex:.*\..*)           // 多次set没意义，只有第一次有意义，用于过滤库/表的
    public void setEventBlackFilter(CanalEventFilter eventBlackFilter)                          // new AviaterRegexFilter(canal.instance.filter.black.regex, false)     // 前后文没有set多次情况，和eventFilter作用一样，前者是不匹配返回null，后者是匹配返回null
    public void setConnectionCharset(String connectionCharset)                                  // 参数：canal.instance.connectionCharset，默认值：UTF-8
    public void setFilterQueryDcl(boolean filterQueryDcl)                                       // 参数：canal.instance.filter.query.dcl，默认值：false
    public void setFilterQueryDml(boolean filterQueryDml)                                       // 参数：canal.instance.filter.query.dml，默认值：false
    public void setFilterQueryDdl(boolean filterQueryDdl)                                       // 参数：canal.instance.filter.query.ddl，默认值：false
    public void setFilterRows(boolean filterRows)                                               // 参数：canal.instance.filter.rows，默认值：false
    public void setFilterTableError(boolean filterTableError)                                   // 参数：canal.instance.filter.table.error，默认值：false

    public void setFallbackIntervalInSeconds(int fallbackIntervalInSeconds)                     // 参数：canal.instance.fallbackIntervalInSeconds，默认值：60
    public void setSupportBinlogFormats(String formatStrs)                                      // 参数：canal.instance.binlog.format
        使用,分隔，并转换为BinlogFormat数组，设置给supportBinlogFormats属性
    public void setSupportBinlogImages(String imageStrs)                                        // 参数：canal.instance.binlog.image
        使用,分隔，并转换为BinlogImage数组，设置给supportBinlogImages属性

    AtomicBoolean needTransactionPosition = new AtomicBoolean(false)

    public void start() throws CanalParseException
        设置binlogParser属性等于new LogEventConvert()实例
        如果eventFilter属性不等于null并且是AviaterRegexFilter类型
            执行binlogParser.setNameFilter((AviaterRegexFilter) eventFilter)
        如果eventBlackFilter属性不等于null并且是AviaterRegexFilter类型
            执行binlogParser.setNameBlackFilter((AviaterRegexFilter) eventBlackFilter)
        执行binlogParser.setCharset(connectionCharset);
        执行binlogParser.setFilterQueryDcl(filterQueryDcl);
        执行binlogParser.setFilterQueryDml(filterQueryDml);
        执行binlogParser.setFilterQueryDdl(filterQueryDdl);
        执行binlogParser.setFilterRows(filterRows);
        执行binlogParser.setFilterTableError(filterTableError);
        执行binlogParser.start()

        设置parseThread属性等于new Thread()实例，重写run方法
            public void run()
                while (running) {
                    ErosaConnection erosaConnection = null
                    try {
                        // 33表示utf8_general_ci（http://dev.mysql.com/doc/internals/en/character-set.html）
                        设置erosaConnection变量等于new MysqlConnection(runningInfo.address, runningInfo.username, runningInfo.password, (byte) 33, runningInfo.defaultDatabaseName)实例
                        执行erosaConnection.getConnector().setReceiveBufferSize(receiveBufferSize)
                        执行erosaConnection.getConnector().setSendBufferSize(sendBufferSize)
                        执行erosaConnection.getConnector().setSoTimeout(defaultConnectionTimeoutInSeconds * 1000)
                        执行erosaConnection.setCharset(connectionCharset)
                        执行erosaConnection.setSlaveId(this.slaveId)

                        执行startHeartBeat(erosaConnection)                                                              // 开启心跳检测任务
                        执行preDump(erosaConnection)                                                                     // 校验连接有效且支持BinlogFormat和BinlogImage，设置表结构加载器给binlogParser
                        执行erosaConnection.connect()
                        执行findStartPosition(erosaConnection)返回EntryPosition类型的实例，设置给startPosition变量           // 获取到开始位置
                        如果startPosition变量等于空，抛异常CanalParseException(can't find start position for...)
                        执行erosaConnection.reconnect()

                        创建new SinkFunction<EVENT>()实例，设置给sinkHandler变量
                            public boolean sink(EVENT event) {
                                try {
                                    设置entry变量等于binlogParser.parse(event);
                                    如果running属性等于false
                                        返回false
                                    如果entry变量不等于null
                                        设置exception属性等于null
                                        执行transactionBuffer.add(entry)                                                // 添加进去
                                        设置lastEntryTime属性等于系统当前时间戳                                             // 更新添加时间为当前时间
                                    返回running属性
                                } catch (TableIdNotFoundException e) {
                                    抛异常e;
                                } catch (Exception e) {
                                    抛异常CanalParseException(e)
                                }
                            }
                        如果startPosition.journalName等于null并且startPosition.timestamp不等于null
                            执行erosaConnection.dump(startPosition.timestamp, sinkHandler)                               // 不支持，抛异常NullPointerException("Not implement yet")
                        否则
                            执行erosaConnection.dump(startPosition.journalName, startPosition.position, sinkHandler)     // 执行mysql dump
                    } catch (TableIdNotFoundException e) {
                        设置exception属性等于e参数
                        对needTransactionPosition属性执行cas操作，从false设置为true                                         // 重新找位置，会返回当前开始位置之前的事物开始位置
                    } catch (Throwable e) {
                        设置exception属性等于e参数
                        如果running等于false
                            如果e不是ClosedByInterruptException类型的异常且e.getCause()不是ClosedByInterruptException类型的异常
                                抛异常CanalParseException(dump address...)
                        否则
                            执行alarmHandler.sendAlarm(destination, e.getMessage())
                    } finally {
                        执行Thread.interrupted()
                        如果connection不是MysqlConnection类型，抛异常CanalParseException(Unsupported connection type...)
                        如果metaConnection属性不等于null，try { 执行metaConnection.disconnect() } catch (IOException e) {}            // 关闭用于获取表结构的连接
                        如果erosaConnection变量不等于null，
                            try {
                                执行erosaConnection.disconnect()
                            } catch (IOException e) {
                                如果running等于false
                                    抛异常CanalParseException(disconnect address...)
                            }
                    }
                    执行eventSink.interrupt()
                    执行transactionBuffer.reset()
                    执行binlogParser.reset()                          // 清空已加载过的表结构信息
                    如果running等于true，睡眠10000 - 20000毫秒
                }
        执行parseThread.setUncaughtExceptionHandler(handler)          // 打印日志，避免线程因异常而终止
        执行parseThread.setName("destination = " + destination + " , address = " + runningInfo.address + " , EventParser")
        执行parseThread.start()
    public void stop() throws CanalParseException
        如果metaConnection属性不等于null，执行metaConnection.disconnect()
        如果tableMetaCache属性不等于null，执行tableMetaCache.clearTableMeta()
        终止并等待parseThread线程结束
        执行eventSink.interrupt()
        执行binlogParser.stop()
    protected void preDump(ErosaConnection connection)                                                                  // 校验连接有效且支持BinlogFormat和BinlogImage，设置表结构加载器给binlogParser
        如果connection参数不是MysqlConnection类型，抛异常CanalParseException(Unsupported connection type...)
        如果binlogParser属性不等于null并且是LogEventConvert类型的实例
            设置metaConnection属性等于connection.fork()
            执行metaConnection.connect()，如果出异常，抛new CanalParseException(e)
            如果supportBinlogFormats属性不等于null并且个数大于0
                验证metaConnection.getBinlogFormat()必须是其中的一个属性，否则抛异常CanalParseException(Unsupported BinlogFormat...)
            如果supportBinlogImages属性不等于null并且个数大于0
                验证metaConnection.getBinlogImage()必须是其中的一个属性，否则抛异常CanalParseException(Unsupported BinlogImage...)
            设置tableMetaCache属性等于new TableMetaCache(metaConnection)
            执行((LogEventConvert) binlogParser).setTableMetaCache(tableMetaCache)
    protected EntryPosition findStartPosition(ErosaConnection connection) throws IOException
        执行findStartPositionInternal(connection)方法返回EntryPosition类型的实例，设置给startPosition变量
        如果needTransactionPosition属性等于true               // 上一次调用findStartPosition(ErosaConnection connection)返回的位置有有问题，需要找到他前面的事物开始位置
            执行findTransactionBeginPosition(connection, startPosition)返回Long类型的实例，设置给preTransactionStartPosition变量
            如果preTransactionStartPosition变量和startPosition.getPosition()不等
                执行startPosition.setPosition(preTransactionStartPosition)
            设置needTransactionPosition属性等于false          // 只针对这次有效，下次不走这套逻辑了
        返回startPosition变量
    private Long findTransactionBeginPosition(ErosaConnection mysqlConnection, EntryPosition entryPosition) throws IOException      // 如果entryPosition后边的第一个事件类型不是TRANSACTIONBEGIN或TRANSACTIONEND，找到entryPosition前面最近的事件类型等于TRANSACTIONBEGIN的位置
        设置reDump变量等于false
        执行mysqlConnection.reconnect()
        创建new SinkFunction<LogEvent>()实例，设置给sinkFunction方法
            public boolean sink(LogEvent event) {
                try {
                    执行binlogParser.parse(event)，设置给entry变量，如果entry变量等于null，返回true
                    如果entry.entryType等于TRANSACTIONBEGIN或TRANSACTIONEND
                        返回false
                    否则
                        设置reDump变量等于true
                        返回false
                } catch (Exception e) {
                    设置reDump变量等于true
                    返回false
                }
            }
        执行mysqlConnection.seek(entryPosition.journalName, entryPosition.position, sinkFunction)     // 检测position后边的第一个不等于null的事件如果不是TRANSACTIONBEGIN或TRANSACTIONEND，标记需要从头dump，以及解析失败也需要从头dump
        如果reDump变量等于true                                                                         // 需要从头dump，找出在entryPosition前面最近的TRANSACTIONBEGIN的事件位置
            设置preTransactionStartPosition变量等于0
            执行mysqlConnection.reconnect()
            创建new SinkFunction<LogEvent>()实例，设置给sinkFunction方法
                public boolean sink(LogEvent event) {
                    try {
                        执行binlogParser.parse(event)，设置给entry变量，如果entry变量等于null，返回true
                        如果entry.entryType等于TRANSACTIONBEGIN并且entry.header.logfileOffset小于entryPosition.position
                            设置preTransactionStartPosition变量等于entry.header.logfileOffset           // 找到最接近entryPosition.position的TRANSACTIONBEGIN的事件位置

                        如果entry.header.logfileOffset大于等于entryPosition.position
                            返回false                                                                 // 如果超过了，就没必要再找了
                    } catch (Exception e) {
                        返回false
                    }
                    返回running属性
                }
            执行mysqlConnection.seek(entryPosition.journalName, 4L, sinkFunction)                     // 从日志文件头部开始dump
            如果preTransactionStartPosition变量大于entryPosition.position                              // 超了抛异常
                抛异常CanalParseException(preTransactionStartPosition greater than startPosition from zk or localconf, maybe lost data)
            返回preTransactionStartPosition                                                           // 返回定位到的
        否则
            返回entryPosition.position;
    protected EntryPosition findStartPositionInternal(ErosaConnection connection)
        执行logPositionManager.getLatestIndexBy(destination)返回LogPosition类型的实例，设置给logPosition变量                // 读取上一个事物的结束位置或上一次最旧的ack位置
        如果logPosition变量等于null                       // 要么是第一次，要么是节点都被清了（没有订阅者）
            设置entryPosition变量等于null
            如果masterInfo不等于null并且masterInfo.address和mysqlConnection.getConnector().getAddress()相等               // 如果当前是master，先用master的
                设置entryPosition变量等于masterPosition属性
            否则如果standbyInfo不等于null并且standbyInfo.address和mysqlConnection.getConnector().getAddress()相等          // 如果当前是slave，先用slave的
                设置entryPosition变量等于standbyPosition属性
            如果entryPosition变量等于null，执行findEndPosition(mysqlConnection)设置给entryPosition变量                     // 如果都没设置，用最新的
            如果entryPosition.journalName等于null                                                                       // 手动配置场景（用了master或slave的，配置没指定journalName）
                如果entryPosition.timestamp不等于null并且大于0                                                               // 指定了时间，就按时间找
                    执行findByStartTimeStamp(mysqlConnection, entryPosition.getTimestamp())得到实例并返回
                否则                                                                                                       // 连时间也没指定，只能按最新的来
                    执行findEndPosition(mysqlConnection)并返回
            否则
                如果entryPosition.position不等于null并且大于0                                                            // 有可能是最新的，也可能是手动配置的，反正有位置了，就返回位置
                    返回entryPosition变量
                否则                                                                                                   // 手动配置场景（用了master或slave的，指定journalName，没指定位置）
                    设置specificLogFilePosition变量等于null
                    如果entryPosition.timestamp不等于null并且大于0                                                        // 但是指定时间了，用指定的时间查找指定文件，如果没找到，设置位置等于头并返回，否则，返回找到的
                        执行findEndPosition(mysqlConnection)返回EntryPosition类型的实例，设置给endPosition变量
                        如果endPosition变量不等于null
                            执行findAsPerTimestampInSpecificLogFile(mysqlConnection, entryPosition.timestamp, endPosition, entryPosition.journalName)返回EntryPosition类型的实例，设置给specificLogFilePosition变量
                    如果specificLogFilePosition变量等于null
                        执行entryPosition.setPosition(4L)，并返回entryPosition变量
                    否则
                        返回specificLogFilePosition变量
        否则
            如果logPosition.identity.sourceAddress等于mysqlConnection.getConnector().getAddress()
                返回logPosition.position
            否则                                                                                     // 如果不等，说明发生了切换，这个时候，按照上一次的处理时间减去fallbackIntervalInSeconds开始找
                使用logPosition.position.timestamp减去fallbackIntervalInSeconds * 1000，设置给newStartTimestamp变量
                执行findByStartTimeStamp(mysqlConnection, newStartTimestamp)并返回
    private EntryPosition findByStartTimeStamp(MysqlConnection mysqlConnection, Long startTimestamp)    // 遍历所有已知的binglog文件，找到位于startTimestamp之前的事物开始事件位置或事物结束位置的下一个事件位置，前提是不能超过目前的最新文件
        设置endPosition变量等于findEndPosition(mysqlConnection)
        设置startPosition变量等于findStartPosition(mysqlConnection)
        设置startSearchBinlogFile变量等于endPosition.journalName
        boolean shouldBreak = false;
        while (running && !shouldBreak) {
            try {
                设置entryPosition变量等于findAsPerTimestampInSpecificLogFile(mysqlConnection, startTimestamp, endPosition, startSearchBinlogFile)
                如果entryPosition变量不等于null
                    返回entryPosition变量                                           // 找到了直接返回
                否则
                    如果startPosition.journalName和startSearchBinlogFile相等         // 已经目前已知的最旧的文件了，还是没找到，只能返回null了
                        设置shouldBreak等于true
                    否则
                        设置binlogSeqNum变量等于startSearchBinlogFile.substring(startSearchBinlogFile.indexOf(".") + 1)
                        如果binlogSeqNum变量小于等于1       // 按binglog命名规范，此时已经没有binlog可以在找了，返回null
                            设置shouldBreak等于true
                        否则
                            设置startSearchBinlogFile变量等于startSearchBinlogFile.substring(0, startSearchBinlogFile.indexOf(".") + 1) + String.format("%06d", binlogSeqNum - 1);    // 找当前文件的前一个文件
            } catch (Exception e) {
                设置binlogSeqNum变量等于startSearchBinlogFile.substring(startSearchBinlogFile.indexOf(".") + 1)
                如果binlogSeqNum变量小于等于1               // 按binglog命名规范，此时已经没有binlog可以在找了，返回null
                    设置shouldBreak等于true
                否则
                    设置startSearchBinlogFile变量等于startSearchBinlogFile.substring(0, startSearchBinlogFile.indexOf(".") + 1) + String.format("%06d", binlogSeqNum - 1);            // 找当前文件的前一个文件
            }
        }
        返回null
    private EntryPosition findAsPerTimestampInSpecificLogFile(MysqlConnection mysqlConnection, Long startTimestamp, EntryPosition endPosition, String searchBinlogFile)     // 找searchBinlogFile文件中，最靠近startTimestamp之前的事物开始事件位置或事物结束位置的下一个事件位置，前提是不能超过之前拿到的最新文件（endPosition）
        设置logPosition变量等于new LogPosition()实例
        try {
            执行mysqlConnection.reconnect()
            创建new SinkFunction<LogEvent>()实例，设置给sinkFunction变量
                public boolean sink(LogEvent event) {
                    try {
                        执行binlogParser.parse(event)，设置给entry变量，如果entry变量等于null，返回true
                        如果entry.entryType等于TRANSACTIONBEGIN或TRANSACTIONEND
                            如果entry.header.executeTime大于等于startTimestamp，返回false                                    // 时间已经超了，再找没意义了
                        如果endPosition.journalName等于entry.header.logfileName并且endPosition.position小于等于entry.header.logfileOffset + event.eventLen
                            返回false                                                                                      // 已经是之前拿到最新的文件了，而且位置超了，在找也没意义了
                        如果entry.entryType等于TRANSACTIONEND                       // 如果是事物结束，设置成下一次事件的位置
                            执行logPosition.setPostion(new EntryPosition(entry.header.logfileName, entry.header.logfileOffset + event.eventLen, entry.header.executeTime))
                        否则如果entry.entryType等于TRANSACTIONBEGIN                 // 如果是事物开始，设置成开始的位置
                            执行logPosition.setPostion(new EntryPosition(entry.header.logfileName, entry.header.logfileOffset, entry.header.executeTime))
                    } catch (Exception e) { }
                    返回running属性
                }
            执行mysqlConnection.seek(searchBinlogFile, 4L, sinkFunction)              // 从searchBinlogFile头部开始找
        } catch (IOException e) { }
        如果logPosition.position不等于null
            返回logPosition.position
        返回null
    private EntryPosition findEndPosition(MysqlConnection mysqlConnection)                                              // 最新（返回当前正在使用的二进制文件以及正在执行二进制文件位置）
        try {
            执行mysqlConnection.query("show master status")返回ResultSetPacket类型的实例，设置给packet变量
            如果packet.getFieldValues()等于null或者元素个数等于0
                抛异常CanalParseException(command : 'show master status' has an error...)
            返回new EntryPosition(packet.getFieldValues().get(0), Long.valueOf(packet.getFieldValues().get(1)))实例
        } catch (IOException e) {
            抛异常CanalParseException(command : 'show master status' has an...);
        }
    private EntryPosition findStartPosition(MysqlConnection mysqlConnection)
        try {
            执行mysqlConnection.query("show binlog events limit 1")返回ResultSetPacket类型的实例，设置给packet变量           // 最旧（返回已存在的二进制文件列表中，最小的二进制文件，以及二进制文件大小）
            如果packet.getFieldValues()等于null或者元素个数等于0
                抛异常CanalParseException(command : 'show binlog events limit 1' has an error...)
            返回new EntryPosition(packet.getFieldValues().get(0), Long.valueOf(packet.getFieldValues().get(1)))实例
        } catch (IOException e) {
            抛异常CanalParseException(command : 'show binlog events limit 1' has an...);
        }

### com.alibaba.otter.canal.parse.inbound.mysql.dbsync.LogEventConvert
    public void setNameFilter(AviaterRegexFilter nameFilter)                            // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.eventFilter
    public void setNameBlackFilter(AviaterRegexFilter nameBlackFilter)                  // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.eventBlackFilter
    public void setCharset(Charset charset)                                             // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.connectionCharset
    public void setFilterQueryDcl(boolean filterQueryDcl)                               // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.filterQueryDcl
    public void setFilterQueryDml(boolean filterQueryDml)                               // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.filterQueryDml
    public void setFilterQueryDdl(boolean filterQueryDdl)                               // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.filterQueryDdl
    public void setFilterRows(boolean filterRows)                                       // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.filterRows
    public void setFilterTableError(boolean filterTableError)                           // com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.filterTableError
    public void setTableMetaCache(TableMetaCache tableMetaCache)

    public void start()
    public void stop()

    public Entry parse(LogEvent logEvent) throws CanalParseException
        // 这里还是看源码吧
    public void reset()                                                                                                 // 清空已加载过的表结构信息
        设置binlogFileName属性等于"mysql-bin.000001"
        如果tableMetaCache属性不等于null，执行tableMetaCache.clearTableMeta()

### com.alibaba.otter.canal.parse.inbound.mysql.dbsync.TableMetaCache

    public TableMetaCache(MysqlConnection con)
        设置connection属性等于con参数
        初始化tableMetaCache属性         // Map<String, ServerRunningMonitor>，类似LoadingCache，key不存在，自动根据key创建实例
            public TableMeta apply(String name)
                try {
                    return getTableMeta0(name);
                } catch (IOException e) {
                    try {
                        connection.reconnect();
                        return getTableMeta0(name);
                    } catch (IOException e1) {
                        throw new CanalParseException("fetch failed by table meta:" + name, e1);
                    }
                }
    private TableMeta getTableMeta0(String fullname) throws IOException                                                 // 返回'desc tableName'表结构
        执行connection.query("desc " + fullname)返回ResultSetPacket实例，设置给packet变量
        **********************ResultSetPacket******************
            FieldDescriptors: field,type,null,key,default,extra
            FieldValues: id,int(10) unsigned,no,pri,null,auto_increment,school_name,varchar(300),yes,,null,
        *******************************************************
        用FieldValues的个数除以FieldDescriptors的个数，计算行数，并遍历指定行数
            创建实例FieldMeta实例，并填充类似id/int(10) unsigned/no/pri/null,auto_increment一行信息到其属性中，并添加fieldMeta实例到fieldMetas变量中
        返回new TableMeta(fullname, fieldMetas)实例
    public void clearTableMeta()
        执行tableMetaCache.clear()

