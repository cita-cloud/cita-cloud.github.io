---
title: 技术沙龙第九期
date: 2022-02-14
categories: 分布式系统
---

绚磊：


做一个简单的开发过程中遇到的使用Mybatis plus更新统计数据，存在线程安全问题，也是分布式事务问题的分享吧。[呲牙]

绚磊：


前提：
在我们开发中常常会遇到统计需求。例如，针对一个文件的查看次数的统计。统计的逻辑，挺简单就是数据库数据自增。
在我们的系统中，现在大部分使用的ORM框架是Mybatis plus。Mybatis plus的特点是，将对数据库操作的sql封装成java
程序员熟悉的class对象方法。
通过Mybatis plus来对 数据库自增往往会有如下的操作。

绚磊：


假设数据存在t_file表，创建File对象如下
```
@Data
@TableName("t_file")
@AllArgsConstructor
@NoArgsConstructor
public class File {

    private Integer id;

    private String filename;

    private Integer times;
}
```

绚磊：


生成Mybatis plus的 mapper 接口类
```
@Mapper
public interface FileMapper extends BaseMapper<File> {
}
```

绚磊：


生成service层对象类
```
@Service
public class FileServiceImpl extends ServiceImpl<FileMapper, File> implements FileService {


    @Override
    public Integer increase(){
        //获取 文件 在数据库存储的信息
        File file = this.baseMapper.selectById(1);
        Integer times = file.getTimes();
        //对文件的获取次数进行 增加 1 次
        file.setTimes(file.getTimes() 1);
        //更新数据库数据
        this.baseMapper.updateById(file);
        //返回增加后的获取次数
        return file.getTimes();
    }
}
```
绚磊：


咋一看可能没发现问题，通过简单的单个请求测试也会发现功能正常。但实际上，大量请求并发执行时，会造成统计数据小于实际请求数据量。

绚磊：


下面进行下问题复现吧，我这边新建了一个测试项目。

绚磊：

![](1.jpeg)

![](9.jpeg)


启动项目，打上断点 。断点挂在 Thread线程上

绚磊：

![](3.jpeg)

表里 初始数据

绚磊：

![](4.jpeg)

从浏览器发送 两次请求

绚磊：

![](5.jpeg)

可以看到 两个请求进来了。

绚磊：

![](6.jpeg)

![](7.jpeg)

后续的结果很明显了，http-nio-7777-exec-1和http-nio-7777-exec-2获取到的 id=1 的times 都是0

![](8.jpeg)

绚磊：

后续进行了 times= times 1，然后更新表后，会发现虽然进行了两次请求，但实际数据库表里的数据为1次。

绚磊：


整理下逻辑：

绚磊：


1.当A请求，请求到服务器，开启线程1执行 File file = this.baseMapper.selectById(1);语句完成后，进入阻塞，此时file.getTimes() 的值为 0。
2.此时B请求，请求到服务器，开启线程2，也执行到File file = this.baseMapper.selectById(1);语句完成后，进入阻塞，此时file.getTimes() 的值也为 0。
3.此时可以明显发现问题出现了
4.后续A请求，B请求无论哪个先执行完，更新到数据库中的file.times 的 值 只会是1 【但请求数是2】


绚磊：


接下详细分析出现问题的原因：

绚磊：


这是一个很明显的线程安全问题，多个线程操作同一个资源。此外，由于这是对数据库记录的操作，这也涉及到数据库的事务，事务隔离级别，我们发现上面的代码，没有使用到数据库事务，
那么是否有可能加上事务就会解决问题了呢，这里先把结论说出来吧，实际上加上事务也是会存在问题的。总结下，上面线程安全问题的原因是，多线程通过了一系列非原子性的操作，修改了
相同的资源。接下来我们分下下数据库层面，默认情况下，MySQL。默认情况下，MySQL 的 innodb数据引擎的事务隔离级别是 RR，也就是Repeatable Read，
rr的原理是 MVCC多版本并发控制 悲观锁 乐观锁。从数据库事务角度分析上面问题，其实是事务A通过select * from file 获取到的对象 file id =1; name="file",times=1【这是一个快照snapshot】。
事务B 获取到的也是一个快照，获取到的也是 file.times = 1。所以也最终导致了最后 统计数据 小于 实际请求数量。

绚磊：


说了这么多，还没说解决方法。接下去说说几种解决方法，根据问题分析得出造成问题的原因有两点：1.多线程 2.非原子性操作。
那么可以从两个维度来解决问题：1.对多线程操作的方法加锁 2.将操作修改成 原子性操作。
先说说第一个维度的几个方案，加锁：分别从数据库，java单个应用，分布式多个应用做出方案。
首先由于这个场景是针对数据库的数据做统计自增操作，上面也分析了，也有数据库事务隔离级别的影响，那么是否能考虑将事务隔离级别调整下，原本默认是REPEATABLE_READ，调整成SERIALIZABLE。然后测试下看。

绚磊：


修改 FileServiceImpl类

![](10.jpeg)

绚磊：


测试结果，两个请求获取到的仍然是0

![](11.jpeg)

绚磊：


第一个请求，update数据库成功，第二个请求，报死锁错误。

![](12.jpeg)

绚磊：


这里可能涉及到 mysql 锁机制，后续我再测试分析下。

绚磊：


总结此方法：能实现数据的安全。但是只能有成功一个请求，实际编码中不推荐此方法。

绚磊：


第二个方案：加synchronized 或者 ReentrantLock 来解决，多线程问题。

绚磊：


修改Service类

![](13.jpeg)

绚磊：


测试打上断点，两个请求并发执行，会发现第一个请求进入了断点，第二个请求没有进入断点，但是浏览器上第二个请求阻塞了。

![](14.jpeg)

绚磊：


最后的执行结果：正确

![](15.jpeg)

![](16.jpeg)

绚磊：


但是 此方法：仅适用于单节点应用，现在基本上都是前后端分离的分布式系统。可以想象下，启动多个节点服务，此时两个请求分别打在两个服务上。实际上并不存在，线程竞争。此时问题变成了，分布式事务问题。

绚磊：


所以 就有个第三个方案：方案加分布式锁，例如利用redis加分布式锁。这里就不演示了，此方法肯定是可行的，但是增加了逻辑上的复杂度，带来了分布式锁的加锁和解锁问题，还涉及到等待请求的公平与非公平。

绚磊：


最后 思考下，是否还有其他方案呢，其实从原子性操作角度分析，由于我们在进行自增操作的时候是先获取了数据库的值，然后再进行java逻辑层的自增，再更新到mysql数据库中。这样获取值和更新不是一个原子操作。通过自定义 sql 。 不使用mybatis plus提供的方法，来做操作 。这样就能保证自增的原子性了。

绚磊：

![](17.jpeg)

修改方法为：

![](18.jpeg)

绚磊：


测试结果：

![](19.jpeg)

![](20.jpeg)

绚磊：


分析：有同学可能会问，两个事务下线程A和线程B同时阻塞在，update操作；然后线程A先commit，接着线程B再commit，是否会有问题。

由于update 在本身就是X锁排他锁，线程A，线程B并发执行，只有一个线程能获取到锁。假设线程B获取到锁，那么线程A就进入等待。

线程B执行完 update 语句后，但是未提交，生成一条redo log；

线程A执行完 update 语句后，但是未提交，生成一条redo log；

通过sql可以查询到获取锁的sql

```
SELECT * FROM performance_schema.events_statements_history WHERE thread_id IN(
SELECT b.`THREAD_ID` FROM sys.`innodb_lock_waits` AS a , performance_schema.threads AS b
WHERE a.`blocking_pid` = b.`PROCESSLIST_ID`)
```

![](21.jpeg)

绚磊：


- 此时执行select times from t_vc where id=5 for update;会阻塞。因为在采用INNODB的MySQL中，更新操作默认会加行级锁.所以窗口二这里会卡住。

- 然后线程A commit，释放锁；线程Bcommit，释放锁。

绚磊：


从执行日志看：

![](22.jpeg)

绚磊：


先后执行了更新操作。

绚磊：


最后总结下： 通过自定义sql update table set a=a 1 where id=1 来解决 统计数据的自增问题，最为合适