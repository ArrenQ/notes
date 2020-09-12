### 关于spring 的 Order 误区
    order并不是指定谁先加载，谁后加载，而是AOP级别的指定加载的bean在集合中的排序。换句话说，影响的是谁先使用，谁后使用。
    AutoConfigureOrder 则是指定autoconfig的工厂文件中指定的所有配置对象，加载顺序，本系统类不存在这个，所有自己系统的配置顺序不受影响
### spring @PropertySource 
    @PropertySource 默认实现只能加载properties文件，使用PropertySource的yml文件会有问题。