#### return 和 confirm
return 监听是当无法找到合适的路由时触发的消息，需要和 mandatory 一起使用
confirm 监听是当

#### 消息可靠性投递方案一
1. 需要两张表（或DB） Biz, Msg
2. 消息发送前，修改Biz状态为待确认，创建 Msg 记录。
3. 发送消息，并等待 confirm listener 触发，触发后修改 Biz和 Msg的状态为已处理
4. 开启定时任务，定期检查所有状态为待确认的 Msg记录（表示未确认），再次发送消息（retry）。可以设置最高重试次数

该方案会修改两次  Biz，一次 Msg， 并新增一次msg。DB压力较大

#### 消息可靠性投递方案二
1. 准备好 `业务队列`, `检查队列`,`Callback队列`，并创建 一个 CallbackService 服务 用于消费 `Callback队列`
2. 消息发送前，修改 Biz 表记录的状态，
3. 发送消息到 `业务队列`, 并且·不·等待 confirm listener
4. 延迟（如5分钟）发送一条消息到 `检查队列`
5. 当消息处理完后，有消费者发送一条 `确认消息` 到 `Callback队列`
6. callback service收到`确认消息`后，将结果存入 Msg DB
7. callback service收到 `延迟检查消息`后，检查 Msg DB 有没有成功记录。如果有，什么都不做。如果没有，则通过 RPC 通知生产者服务进行重试

该方案只会修改一次 Biz 业务记录（甚至可能不用修改，因为默认就是发送后没有被callback service RPC 通知重试就算成功）
和一次 Msg DB 的新增。


