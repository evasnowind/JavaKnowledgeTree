spring 生命周期，自己画图



Java的伪泛型与集合的关系， 泛型可以写 class Student<A extends B>，如果A需要限定两个父接口B和C怎么办？ enum AType到底是什么？







## Java 8 特性与常用工具

### lombok

### Guava







AOP注解相关

参见aopDemo     spring_aop_log   这两个项目







一般情况，比较数据库、序列化框架的性能意义不大



mybaits

hibernate





为何大公司都用mybatis：

对DBA友好，sql可控，便于sql优化，hibernate出来的sql基本没法看



mybatis 使用mybatis generator 自动生成的xml，继承这个xml再去做自定义，不容易错



事务的隔离级别

写偏序

写service时尽量不要出现相互嵌套，否则容易出现循环依赖





数据源

动态数据源的实现





建议

2、基础应用建议少讲一些，尤其是中间件怎么这种，我觉得报这个训练营的，如果最佳实践多点时间，并且中间件原理、最佳实践适当扩充一下内容？尤其今天讲的最佳实践，个别例子讲的很长，但ppt上就一行、也没加上提示语、关键词，后续复习会比较麻烦，还需要重新翻视频





nicefish 项目里有TypeHandler 的使用

SQL查询尽量不要用函数，因为函数往往无法使用索引



s

我很希望使用技术手段来解决这一问题，通过服务质量监控来在研发提测后，自动报告相关数

据，例如；研发代码涉及流程链路展示、每个链路测试次数、通过次数、失败次数、当时的出入参信息

以及对应的代码块在当前提测分支修改记录等各项信息。最终测试在执行验证时候，分配验证渠道扫描

到所有分支节点，可以清晰的看到全链路的影响。那么，这样的测试才是可以保证系统的整体质量的。

