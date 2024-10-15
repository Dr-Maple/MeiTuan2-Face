# MeiTuan2-Face
美团二面：为什么不推荐使用 MyBatis 二级缓存？大部分人都答不上来！

一级缓存应用场景生效的条件与springboot集成时一级缓存不生效原因解决与springboot集成时一级缓存不生效问题二级缓存应用场景开启的方法生效的条件缓存清除条件源码中是如何填充二级缓存的？查询时如何使用二级缓存？为什么mybatis默认不开启二级缓存？


为了增加查询的性能，mybatis 提供了二级缓存架构，分为一级缓存和二级缓存。这两级缓存最大的区别就是：一级缓存是会话级别的，只要出了这个 SqlSession，缓存就没用了。而二级缓存可以跨会话，多个会话可以使用相同的缓存！一级缓存使用简单，默认就开启。二级缓存需要手动开启，相对复杂，而且要注意的事项也多，否则可能有隐患。一级缓存应用场景订单表与会员表是存在一对多的关系，为了尽可能减少join查询，进行了分阶段查询。即先查询出订单表，再根据member_id字段查询出会员表，最后进行数据整合。

而如果订单表中存在重复的member_id，就会出现很多重复查询。针对这种情况，mybatis通过一级缓存来解决：在同一次查询会话(SqlSession)中如果出现相同的语句及参数，就会从缓存中取出，不再走数据库查询。一级缓存只能作用于查询会话中，所以也叫做会话缓存。生效的条件一级缓存要生效，必须满足以下条件条件：必须是相同的会话必须是同一个 mapper，即同一个 namespace必须是相同的 statement，即同一个 mapper 中的同一个方法必须是相同的 sql 和参数查询语句中间没有执行 session.clearCache() 方法查询语句中间没有执行 insert/update/delete 方法（无论变动记录是否与缓存数据有无关系）与springboot集成时一级缓存不生效原因图片因为一级缓存是会话级别的，要生效的话，必须要在同一个 SqlSession 中。但是与 springboot 集成的 mybatis，默认每次执行sql语句时，都会创建一个新的 SqlSession！所以一级缓存才没有生效。当调用 mapper 的方法时，最终会执行到 SqlSessionUtils 的 getSqlSession 方法，在这个方法中会尝试在事务管理器中获取 SqlSession，如果没有开启事务，那么就会 new 一个 DefaultSqlSession。

所以说，即便在同一个方法中，通过同一个 mapper 连续调用两次相同的查询方法，也不会触发一级缓存。解决与springboot集成时一级缓存不生效问题在上面的代码中也看到了，mybatis 在查询时，会先从事务管理器中尝试获取 SqlSession，取不到才会去创建新的 SqlSession。所以可以猜测只要将方法开启事务，那么一级缓存就会生效。加上 @Transactional 注解，看下效果：图片没错，的确生效了。在代码中可以看到，从事务管理器中，获取到了 SqlSession：图片再看看源码中是什么时候将 SqlSession 设置到事务管理器中的。SqlSessionUtils 中，在获取到 SqlSession 后，会调用 registerSessionHolder 方法注册 SessionHolder 到事务管理器：图片具体是在 TransactionSynchronizationManager 的 bindResource 方法中操作的，将 SessionHolder 保存到线程本地变量(ThreadLocal) resources 中，这是每个线程独享的。图片然后在下次查询时，就可以从这里取出此 SqlSession，使用同一个 SqlSession 查询，一级缓存就生效了。所以基本原理就是：如果当前线程存在事物，并且存在相关会话，就从 ThreadLocal 中取出。如果没有事务，就重新创建一个 SqlSession 并存储到 ThreadLocal 当中，共下次查询使用。

至于缓存查询数据的地方，是在 BaseExecutor 中的 queryFromDatabase 方法中。执行 doQuery 从数据库中查询数据后，会立马缓存到 localCache(PerpetualCache类型) 中：图片二级缓存应用场景业务系统中存在很多的静态数据如，字典表、菜单表、权限表等，这些数据的特性是不会轻易修改但又是查询的热点数据。一级缓存针对的是同一个会话当中相同SQL，并不适合这情热点数据的缓存场景。为了解决这个问题引入了二级缓存，它脱离于会话之外，多个会话可以使用相同的缓存。看一个例子：@RestController
@RequestMapping("item")
public class ItemController {
 
    @Autowired
    private ItemMapper itemMapper;
 
    @GetMapping("/{id}")
    public void getById(@PathVariable("id") Long id) {
        System.out.println("==================== begin ====================");
        Item item = itemMapper.selectById(id);
        System.out.println(JSON.toJSONString(item));
    }
 
} 
当发送两次 get 请求时（两个不同的会话），通过日志可以发现第二次查询使用的是缓存图片开启的方法二级缓存需要手动来开启，mybatis 默认没有开启二级缓存。1）在 yaml 中配置 cache-enabled 为 truemybatis-plus:
  configuration:
    cache-enabled: true
2）Mapper 接口上添加 @CacheNamespace 注解图片3）实体类实现 Serializable 接口图片生效的条件当会话提交或关闭之后才会填充二级缓存必须是同一个 mapper，即同一个命名空间必须是相同的 statement，即同一个 mapper 中的同一个方法必须是相同的 SQL 语句和参数如果 readWrite=true(默认就是true)，实体对像必须实现 Serializable 接口缓存清除条件只有修改会话提交之后，才会执行清空操作xml 中配置的 update 不能清空 @CacheNamespace 中的缓存数据任何一种增删改操作都会清空整个 namespace 中的缓存源码中是如何填充二级缓存的？在生效条件中提到了，二级缓存必须要在会话提交或关闭之后，才能生效！在查询到结果后，会调用 SqlSession 的 commit 方法进行提交（如果开启事务的话，提交 SqlSession 走的不是这里了，但最终填充二级缓存的地方是一样的。）：图片在此方法中，最终会调用到 TransactionalCache 的 flushPendingEntries 方法中填充二级缓存：图片springboot 集成 mybatis 的话，如果没有开启事务，每次执行查询，都会创建新的 SqlSession，所以即使是在同一个方法中进行查询操作，那也是跨会话的。查询时如何使用二级缓存？在查询的时候，最终会调用 MybatisCachingExecutor 的 query 方法，里面会从 TransactionalCacheManager 中尝试根据 key 获取二级缓存的内容。可以看到，这个 key 很长，由 mapper、调用的查询方法、SQL 等信息拼接而成，这也是为什么想要二级缓存生效，必须满足前面所说的条件。如果能在二级缓存中查询到，就直接返回了，不需要访问数据库。图片具体的调用层数实在太多，用到了装饰者模式，最终是在 PerpetualCache 中获取缓存的：图片打印日志是在 LoggingCache 中：图片为什么mybatis默认不开启二级缓存？答案就是，不推荐使用二级缓存！二级缓存虽然能带来一定的好处，但是有很大的隐藏危害！它的缓存是以 namespace(mapper) 为单位的，不同 namespace 下的操作互不影响。且 insert/update/delete 操作会清空所在 namespace 下的全部缓存。那么问题就出来了，假设现在有 ItemMapper 以及 XxxMapper，在 XxxMapper 中做了表关联查询，且做了二级缓存。此时在 ItemMapper 中将 item 信息给删了，由于不同 namespace 下的操作互不影响，XxxMapper 的二级缓存不会变，那之后再次通过 XxxMapper 查询的数据就不对了，非常危险。来看一个例子：@Mapper
@Repository
@CacheNamespace
public interface XxxMapper {
 
    @Select("select i.id itemId,i.name itemName,p.amount,p.unit_price unitPrice " +
            "from item i JOIN payment p on i.id = p.item_id where i.id = #{id}")
    List<PaymentVO> getPaymentVO(Long id);
 
}
 
 
@Autowired
private XxxMapper xxxMapper;
 
@Test
void test() {
 System.out.println("==================== 查询PaymentVO ====================");
 List<PaymentVO> voList = xxxMapper.getPaymentVO(1L);
 System.out.println(JSON.toJSONString(voList.get(0)));
 System.out.println("====================  更新item表的name ==================== ");
 Item item = itemMapper.selectById(1);
 item.setName("java并发编程");
 itemMapper.updateById(item);
 System.out.println("====================  重新查询PaymentVO ==================== ");
 List<PaymentVO> voList2 = xxxMapper.getPaymentVO(1L);
 System.out.println(JSON.toJSONString(voList2.get(0)));
}
上面的代码，test()方法中前后两次调用了 xxxMapper.getPaymentVO 方法，因为没有加 @Transactional 注解，所以前后两次查询，是两个不同的会话，第一次查询完后，SqlSession 会自动 commit，所以二级缓存能够生效；然后在中间进行了 Item 表的更新操作，修改了下名称；由于 itemMapper 与 xxxMapper 不是同一个命名空间，所以 itemMapper 执行的更新操作不会影响到 xxxMapper 的二级缓存；再次调用 xxxMapper.getPaymentVO，发现取出的值是走缓存的，itemName 还是老的。但实际上 itemName 在上面已经被改了！执行日志如下：图片所以说，二级缓存的隐藏危害是比较大的，当有表关联时，一个不注意就会出问题，不建议使用。
