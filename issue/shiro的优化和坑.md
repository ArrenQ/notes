# 关于shiro的几个优化
### shiro 和 druid 矛盾。 
    需要说明的是，系统中使用 druid 和 shiro。由于某些特定的路径下的请求在系统配置中不进行user验证，直接放行。
    当一次请求结束后，shiro会自动判断用户是否已经退出登录，如果是则删除session。这带来一个问题，不进行user验证的所有请求，在执行完成后，都被shiro当做退出登录了。
    因此session会被自动删除。而 druid 配置启动数据库请求监听时 filter 需要获取session。当获取session失败就会报错。这对业务似乎没有影响。
    但似乎springmvc会因为报错而将response的返回码变成 500，而不 200。 而有些客户接入我们的接口时可能会比较严格的判断返回码必须为200。
    这个锅应该shiro背。解决这个问题的方式是通过第二个优化方案(shiro 和 redis矛盾)顺带解决的。
### shiro 和 redis 矛盾
    shiro 默认每次获取session都会通过cacheManager中的sessionDao去获取。我们系统中cacheManager设置的是redis实现。那么每一次请求中一大堆的filter都会去getSession()
    这导致一次请求shiro就丧心病狂的访问了redis 无数次。解决这个问题的方式参考MyShiroSessionManager实现的说明。正好这种解决方法对shiro和druid问题也有神奇疗效。
    不过这锅大概shiro不背。如果shiro一次访问中只从redis中获取一次session，那么实际上其他服务器共享session时修改的数据，就无法及时获取（不过这场景很少）。

### shiro的坑
    1. shiro 创建session时，会同时调用update session，会连续两次访问redis，如果session有大量的更新，也会多次访问redis
    2. shiro 删除session时也有类似问题
        protected void onExpiration(Session s, ExpiredSessionException ese, SessionKey key) {
                log.trace("Session with id [{}] has expired.", s.getId());
                try {
                    onExpiration(s); // 检查走change并删除session
                    notifyExpiration(s);
                } finally {
                    afterExpired(s); // 删除session
                }
            }

### shiro session定时清理问题。
    开启session validation的方法，父类实际上已经实现，但是开启session validation定时任务是延迟开启的，而且没有做同步处理。
    可能会导致系统启动时N个request同时请求，导致定时任务被多次开启。
    本系统使用了 swagger，而swagger bean的加载都是在http端口启动后才进行的。swagger加载前，所以的request都会被阻塞。
    如果swagger加载太慢会导致大量request堆积，并同时开始调用 enableSessionValidationIfNecessary ，
    而enableSessionValidationIfNecessary 内部调用 enableSessionValidation 最终同时并发创建了大量相同的定时任务。
    该方法如果创建定时任务成功， enableSessionValidationIfNecessary 方法不会再次对它进行调用。所以内部加锁并不影响后续的操作。

### 以上问题的解决
    public class MyShiroSessionManager extends DefaultWebSessionManager {
        private final Logger logger = LoggerFactory.getLogger(MyShiroSessionManager.class);
        private final Lock lock = new ReentrantLock();
        /**
         * 获取session
         * 优化单次请求需要多次访问redis的问题.
         * 问题描述，shiro这老兄每次获取session都会通过retrieveSession（cacheManager）去获取。但每一次请求发生时，一堆拦截器都会去getSession()
         * 这导致每一次请求，shiro都丧心病狂的疯狂访问redis。因此需要对这个做优化。
         * 暂时不打算考虑在{@link ShiroRedisCache} 中加入ThreadLocal来缓存，
         * 因为session有自己的生命周期，在某些情况下session被容器回收。但线程对象只有在线程注销时才被清理（如果容器是使用线程池则是被下一次请求覆盖）。
         * 这样做可能导致session本身被容器回收后，仍然有可能获取到。虽然 ShiroRedisCache 和方法是和shiro管理session的周期是一致的，我们也可以在里面进行处理
         * 但不如修改这里来得简单。
         * 另外这种做法对 shiro + druid 产生的问题有神奇的疗效。
         */
        @Override
        protected Session retrieveSession(SessionKey sessionKey) throws UnknownSessionException {
    
            Serializable sessionId = getSessionId(sessionKey);
            if (sessionId == null) {
                logger.debug("Unable to resolve session ID from SessionKey [{}].  Returning null to indicate a " +
                        "session could not be found.", sessionKey);
                return null;
            }
    
            //=== start 在父类实现基础上，添加这段代码，默认先获取request中的session。 request获取不到才通过retrieveSession到 cacheManager(redis) 中获取。
            ServletRequest request = null;
            if (sessionKey instanceof WebSessionKey) {
                request = ((WebSessionKey) sessionKey).getServletRequest();
            }
            if (request != null) {
                Object sessionObj = request.getAttribute(sessionId.toString());
                if (sessionObj != null) {
                    return (Session) sessionObj;
                }
            }
            //=== end
    
            Session session = super.retrieveSession(sessionKey);
            if (request != null) {
                request.setAttribute(sessionId.toString(), session);
            }
            return session;
        }
    
        /**
         * 开启session validation的方法，父类实际上已经实现，但是开启session validation定时任务是延迟开启的，而且没有做同步处理。
         * 可能会导致系统启动时N个request同时请求，导致定时任务被多次开启。
         * 本系统使用了 swagger，而swagger bean的加载都是在http端口启动后才进行的。swagger加载前，所以的request都会被阻塞。
         * 如果swagger加载太慢会导致大量request堆积，并同时开始调用 enableSessionValidationIfNecessary ，
         * 而enableSessionValidationIfNecessary 内部调用 enableSessionValidation 最终同时并发创建了大量相同的定时任务。
         *
         * 该方法如果创建定时任务成功， enableSessionValidationIfNecessary 方法不会再次对它进行调用。所以内部加锁并不影响后续的操作。
         */
        @Override
        protected void enableSessionValidation() {
            logger.info("试图启动定时任务.......");
            if(lock.tryLock()) {
                try {
                    SessionValidationScheduler scheduler = getSessionValidationScheduler();
                    if (isSessionValidationSchedulerEnabled() && (scheduler == null || !scheduler.isEnabled())) {
                        super.enableSessionValidation();
                    }
                } finally {
                    lock.unlock();
                }
            }
        }
    }