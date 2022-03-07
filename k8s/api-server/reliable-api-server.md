## ha
 1. because service is stateless
 2. use load balancer to guide the flow


## 3a
 1. authentication
 2. authorization
 3. admission
  - mutate
  - validate


## rate limit
 1. APF(分类型flow -> 分等级level)

### rate limit algorithm
 1. 固定窗口算法
  特点:
   - 一段时间内规定请求的阈值数量
   - 请求进来计数器加1
   - 时间段结束计数器重置为0
  缺点:
   - 请求在当前时间段的末段新时间段段开始前进入，请求没处理完又会有更多请求进来，达不到限流效果。
 2. 滑动窗口算法
  特点：
   - 在固定窗口的时间段基础上，把时间段继再细分，计时器落在更加小的时间段里面。
   - 当请求时间大于当前窗口最大时间，则计时窗口平移一个窗口。
   - 第一个窗口的时间窗口丢弃，第二个时间窗口成为当前时间段的第一个，向后加入时间窗口。
   - 每个时间窗口都有请求阈值
   - 请求进来同样需要时间窗口的计数器加1
  缺点:
   - 不能应付突发流量
 3. 漏斗算法
  特点：
   - 当漏斗达到最大容量才丢弃请求
   - 漏斗没满仍然可以接收请求
   - 漏斗以恒定速率将请求流出
  缺点：
   - 不能应付突发流量
 4. 令牌桶算法
  特点：
   - 请求来到时需要在令牌桶拿取令牌才能被处理
   - 以恒定速率生产令牌
   - 令牌桶满时会丢弃令牌
  优点：
   - 可以应付突发流量
   - 限流的方式在于生产令牌的速率


## aggregate
 1. aggregate api server(web hook) 