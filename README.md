binlog2sql
========================

从MySQL binlog解析出你要的SQL。根据不同选项，你可以得到原始SQL、回滚SQL、去除主键的INSERT SQL等。

用途
===========

* 数据回滚
* 主从切换后数据不一致的修复
* 从binlog生成标准SQL，带来的衍生功能


项目状态
===
正常维护

* 已测试环境
    * Python 2.6, 2.7
    * MySQL 5.6


安装
==============

```
git clone https://github.com/danfengcao/binlog2sql.git
pip install -r requirements.txt
```

使用
=========

### MySQL server必须设置以下参数:

    [mysqld]
    server-id = 1
    log_bin = /var/log/mysql/mysql-bin.log
    max_binlog_size = 1000M
    binlog-format = row

###基本用法

**解析出标准SQL**

```bash
$ python binlog2sql.py -h127.0.0.1 -P3306 -uadmin -p'admin' -dtest -t test3 test4 --start-file='mysql-bin.000002'

输出：
INSERT INTO `test`.`test4`(`addtime`, `data`, `id`) VALUES ('2016-12-10 13:03:10', 'test', 1); #start 185 end 351
INSERT INTO `test`.`test3`(`addtime`, `data`, `id`) VALUES ('2016-12-10 13:03:22', '中文', 3); #start 378 end 543
INSERT INTO `test`.`test3`(`addtime`, `data`, `id`) VALUES ('2016-12-10 13:03:38', 'english', 4); #start 570 end 736
UPDATE `test`.`test3` SET `addtime`='2016-12-10 12:00:00', `data`='中文', `id`=3 WHERE `addtime`='2016-12-10 13:03:22' AND `data`='中文' AND `id`=3 LIMIT 1; #start 763 end 954
DELETE FROM `test`.`test3` WHERE `addtime`='2016-12-10 13:03:38' AND `data`='english' AND `id`=4 LIMIT 1; #start 981 end 1147
```

**解析出回滚SQL**

```bash
python binlog2sql.py --flashback -h127.0.0.1 -P3306 -uadmin -p'admin' -dtest -ttest3 --start-file='mysql-bin.000002' --start-pos=763 --end-pos=1147

输出：
INSERT INTO `test`.`test3`(`addtime`, `data`, `id`) VALUES ('2016-12-10 13:03:38', 'english', 4); #start 981 end 1147
UPDATE `test`.`test3` SET `addtime`='2016-12-10 13:03:22', `data`='中文', `id`=3 WHERE `addtime`='2016-12-10 12:00:00' AND `data`='中文' AND `id`=3 LIMIT 1; #start 763 end 954
```
###选项
**mysql连接配置**

-h host; -P port; -u user; -p password

**解析模式**

--stop-never 持续同步binlog。可选。不加则同步至执行命令时最新的binlog位置。

--popPk 对INSERT语句去除主键。可选。

-B, --flashback 生成回滚语句。可选。与stop-never或popPk不能同时添加。

**解析范围控制**

--start-file 起始解析文件。必须。

--start-pos start-file的起始解析位置。可选。默认为start-file的起始位置；

--end-file 末尾解析文件。可选。默认为start-file同一个文件。若解析模式为stop-never，此选项失效。

--end-pos end-file的末尾解析位置。可选。默认为end-file的最末位置；若解析模式为stop-never，此选项失效。

**对象过滤**

-d, --databases 只输出目标db的sql。可选。默认为空。

-t, --tables 只输出目标tables的sql。可选。默认为空。

###应用案例

#### **误删整张表数据，需要紧急回滚**

详细描述可参见[example/mysql-rollback-your-data.md](./example/mysql-rollback-your-data.md)

```bash
test库tbl表原有数据
mysql> select * from tbl;
+----+--------+---------------------+
| id | name   | addtime             |
+----+--------+---------------------+
|  1 | 小赵   | 2016-12-10 00:04:33 |
|  2 | 小钱   | 2016-12-10 00:04:48 |
|  3 | 小孙   | 2016-12-10 00:04:51 |
|  4 | 小李   | 2016-12-10 00:04:56 |
+----+--------+---------------------+
4 rows in set (0.00 sec)

mysql> delete from tbl;
Query OK, 4 rows affected (0.00 sec)

tbl表被清空
mysql> select * from tbl;
Empty set (0.00 sec)
```

**恢复数据步骤**：

1. 登录mysql，查看目前的binlog文件

	```bash
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000046 |  12262268 |
| mysql-bin.000047 |      3583 |
+------------------+-----------+
```

2. 最新的binlog文件是mysql-bin.000047，我们再定位误操作SQL的binlog位置

	```bash
$ python binlog2sql/binlog2sql.py -h127.0.0.1 -P3306 -uadmin -p'admin' -dtest -ttbl --start-file='mysql-bin.000047'
输出：
DELETE FROM `test`.`tbl` WHERE `addtime`='2016-12-10 00:04:33' AND `id`=1 AND `name`='小赵' LIMIT 1; #start 3346 end 3556
DELETE FROM `test`.`tbl` WHERE `addtime`='2016-12-10 00:04:48' AND `id`=2 AND `name`='小钱' LIMIT 1; #start 3346 end 3556
DELETE FROM `test`.`tbl` WHERE `addtime`='2016-12-10 00:04:51' AND `id`=3 AND `name`='小孙' LIMIT 1; #start 3346 end 3556
DELETE FROM `test`.`tbl` WHERE `addtime`='2016-12-10 00:04:56' AND `id`=4 AND `name`='小李' LIMIT 1; #start 3346 end 3556
```
        
3. 生成回滚sql，并检查回滚sql是否正确

```bash
$ python binlog2sql/binlog2sql.py -h127.0.0.1 -P3306 -uadmin -p'admin' -dtest -ttbl --start-file='mysql-bin.000047' --start-pos=3346 --end-pos=3556 -B
输出：
INSERT INTO `test`.`tbl`(`addtime`, `id`, `name`) VALUES ('2016-12-10 00:04:56', 4, '小李'); #start 3346 end 3556
INSERT INTO `test`.`tbl`(`addtime`, `id`, `name`) VALUES ('2016-12-10 00:04:51', 3, '小孙'); #start 3346 end 3556
INSERT INTO `test`.`tbl`(`addtime`, `id`, `name`) VALUES ('2016-12-10 00:04:48', 2, '小钱'); #start 3346 end 3556
INSERT INTO `test`.`tbl`(`addtime`, `id`, `name`) VALUES ('2016-12-10 00:04:33', 1, '小赵'); #start 3346 end 3556
```
        
3. 确认回滚sql正确，执行回滚语句。登录mysql确认，数据回滚成功。

```bash
$ python binlog2sql.py -h127.0.0.1 -P3306 -uadmin -p'admin' -dtest -ttbl --start-file='mysql-bin.000047' --start-pos=3346 --end-pos=3556 -B | mysql -h127.0.0.1 -P3306 -uadmin -p'admin'

mysql> select * from tbl;
+----+--------+---------------------+
| id | name   | addtime             |
+----+--------+---------------------+
|  1 | 小赵   | 2016-12-10 00:04:33 |
|  2 | 小钱   | 2016-12-10 00:04:48 |
|  3 | 小孙   | 2016-12-10 00:04:51 |
|  4 | 小李   | 2016-12-10 00:04:56 |
+----+--------+---------------------+
```

###限制
* mysql server必须开启，离线模式下不能解析
* flashback模式，生成的回滚语句不能超过内存大小(有待优化)


###优点（对比mysqlbinlog）

* 纯Python开发，安装与使用都很简单
* 自带flashback、popPk解析模式，无需再装补丁
* 解析为标准SQL，方便理解、调试
* 代码容易改造，可以支持更多个性化解析



###联系我
有任何问题，请与我联系 [danfengcao.info@gmail.com](danfengcao.info@gmail.com)



参考资料
==============
[1] 彭立勋, [MySQL下实现闪回的设计思路](http://www.penglixun.com/tech/database/mysql_flashback_feature.html)

[2] \_\_七把刀__, [MySQL binlog格式解析](http://www.jianshu.com/p/c16686b35807?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

[3] noplay, [Pure Python Implementation of MySQL replication protocol build on top of PyMYSQL](https://github.com/noplay/python-mysql-replication)

