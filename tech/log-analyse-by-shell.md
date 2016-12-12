# 日志分析之shell篇

shell是日志分析最基础，也是最强大的工具之一。本文用到的样例数据文件[在这里](/_img/log-sample01.log)，格式如下：

```
[GIN] 2016/11/21 - 16:50:36 |[97;42m 204 [0m|     441.717µs | 121.69.49.144 |[97;46m  [0m POST    /bid/api
[GIN] 2016/11/21 - 16:50:36 |[97;42m 204 [0m|     773.736µs | 121.69.49.138 |[97;46m  [0m POST    /bid/api
[GIN] 2016/11/21 - 16:50:36 |[97;42m 200 [0m|      49.519µs | 121.69.49.144 |[97;44m  [0m GET     /win/api
[GIN] 2016/11/21 - 16:50:36 |[97;42m 204 [0m|     462.392µs | 121.69.49.139 |[97;46m  [0m POST    /bid/api
[GIN] 2016/11/21 - 16:50:36 |[97;42m 204 [0m|     503.733µs | 121.69.49.145 |[97;46m  [0m POST    /bid/api
[GIN] 2016/11/21 - 16:50:36 |[97;42m 204 [0m|     430.477µs | 121.69.49.138 |[97;46m  [0m POST    /bid/api
[GIN] 2016/11/21 - 16:50:37 |[97;42m 204 [0m|    2.462103ms | 121.69.49.138 |[97;46m  [0m POST    /bid/api
```

这是supervisor记录到的接口的访问日志，对于本文需要用到的字段：

- 日期时间
- http响应码：主要是200和204两种
- 响应时间
- 接口地址

通过对日志的分析，需要达到目标：

- 不同响应码的占比，主要关注除200和204之外的响应码，例如400，500，404等，这是异常导致的
- /bid/api接口下的200状态码的平均响应时间，TOP90%的响应时间，TOP99%的响应时间，和响应时间超过50ms的占比
- 不同状态码下的相应时间分布

## 不同响应码的占比

命令如下：

```sh
cat log-sample01.log|cut -d" " -f6|sort|uniq -c|sort
```

结果如：

```
1    404
79   200
9162 204
```

思路：先筛选需要的字段，然后统计排序。

顺序看这几个命令：

- `cut -d " " -f6`: 根据空格分割字符串，并选取下标为6的子字符串（注意下标是从1开始的），这样就得到了状态码列表
- `sort`: 对状态码列表进行排序，这时为下一个命令准备数据
- `uniq -c`: 对数据进行唯一性统计，需要先排序
- `sort`: 最后的排序是让结果更加明显，默认按第一列进行排序，如果想按照第二列排序`sort -k2`

## bid接口下200状态码的时间分布

确定耗时的单位是否一致：

```sh
grep "/bid/api" log-sample01.log|grep " 200 "|wc -l
grep "/bid/api" log-sample01.log|grep " 200 "|grep "ms |"|wc -l
```

如果这两个命令的输出结果一致，则表示每个接口的耗时的单位都是ms。这里的思路也很简单，使用`grep`命令过滤目标日志记录，然后查找相应的特征，例如`ms |`

最后完整的统计命令如下：

```sh
grep "/bid/api" log-sample01.log|grep " 200 "|sed "s/\s\+/ /g"|cut -d" " -f8|cut -d. -f1|sort|uniq -c|sort -k2 -n
```

结果如下：

```
68 2
5  3
1  4
1  5
1  15
```

结果显示，接近90%的响应时间都在2-3毫秒之间，非常之高效。

解释一下，刚才的统计命令：

- `sed "s/\s\+/ /g"`: 这个替换命令是将多个连续的空格，替换为一个空格。这是关键的一个步骤，完成这个步骤，就可以使用`cut`命令进行分割了
- `cut -d. -f1`: 因为我们统计的是时间段，所以小数点后面的值可以省略
- `sort -k2 -n`: 这个排序命令是按第2列进行排序，`-n`是指按数值进行排序

sort如果需要倒序，则加上`-r`参数

--------------

看到这里，应该能初步感受到使用shell做日志分析的强大了，shell的基本命令不多，但是通过管道组合到一起，就能玩出花来。

还有更强大的工具`awk`

## 经常碰到的问题之一

- 在windows处理的文件，经常出现特殊符号“

这是一个不可见字符，需要处理成换行符，而它的输入是“ctrl+v ctrl+m”!

替换命令：

```sh
cat filename.txt | tr "
```
