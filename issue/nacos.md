## 坑
    alibaba的库一向坑多。项目中使用了nacos作为配置中心。由于nacos加载使用的是在autoconfig，就出现了许多加载问题。
   
    坑1.messagesource autoconfig在加载前会检查propoerties中是否有国际化配置，没有就不创建。
        由于config的condition检查早于autoconfig，因此nacos还来不及获取properties，导致messages文件如果不在根目录，messagesource根本不会创建。
    坑2.logging 是在通过 LoggingApplicationListener创建的，导致和坑1类似的问题。  logging读取不到nacos配置。
        这个可以使用 spring-cloud-nacos解决。spring-boot-nacos不行。