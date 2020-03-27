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

# shiro的坑
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
