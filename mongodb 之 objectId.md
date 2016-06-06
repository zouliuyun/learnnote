### OBJECTID

因为游戏业务运行时间太长 战报服务器硬盘爆满，需要删除一些历史战报，但战报并没有单独存放时间字段
只能通过object对象来处理。

1.1 结构
```sh
ObjectId("52cbab42231dea1e819b2a37"),
ObjectId("52cbab5b231dea1e819b2a38"),
ObjectId("52cbab70231dea1e819b2a39"),

52cbab70      时间戳
231dea        机器号
1e81        进程ID
9b2a39    自增数

以上数字为 16进制表示

特点：


1.24个16进制数据，使用 12字节的存储空间。

2.最后3个字节为：自动增长。可确保每秒生成的值也不一样，一秒最多允许每个进程拥有2563个不同ObjectId

3.可转移到客户端生成，而减轻服务器负担（需要客户端的驱动程序）
```

下面介绍几个相关的函数
```sh
$mongo
>> x=ObjectId()  
ObjectId("53b3a89bf988c39955a30f9e")  
> x  
ObjectId("53b3a89bf988c39955a30f9e")  
>x.str  
53b3a89bf988c39955a30f9e  
  
> x.toString()  
ObjectId("53b3a89bf988c39955a30f9e")  
> x.getTimestamp()  
ISODate("2014-07-02T06:37:15Z")  
> x.valueOf()  
53b3a89bf988c39955a30f9e  
>  
```

函数说明：
* 1.Str 功能 与valueOf()相同
* 2.toString()  把一个对象转换成了一字串。这时两个objectId可以对比。
* 3.getTimestamp() 抽取 时间

我们再来看看一个有意思的查询：
既然可以把从objectId 获取日期，那我是否可以使用一个日期做为条件，使用_id 字段来进行查询呢。

我的方法是: 使用 函数  getTimestamp()先把字段 进行处理，这在一般的关系型数据库非常常用。
```sh
[javascript] view plain copy print?在CODE上查看代码片派生到我的代码片
<span style="font-size:18px;">> a = db.order.findOne()  
{  
    "_id" : ObjectId("5331128631a4804b226471e4"),  
    "md5" : "cacae8722c325d62e795e5c273d5f49b",  
                   ……  
}  
> a."_id"  
Wed Jul  2 14:50:57.494 SyntaxError: Unexpected string  
> a._id.getTimestamp()  
ISODate("2014-03-25T05:22:14Z")  
> new Date("2014,03,25")  
ISODate("2014-03-24T16:00:00Z")  
> new Date("2014,03,25")  
ISODate("2014-03-24T16:00:00Z")  
> db.order.find({"_id.getTimestamp()":{$gt:new Date("2014,03,25")}}).count()  
0  
> db.order.find({_id.getTimestamp():{$gt:new Date("2014,03,25")}}).count()  
Wed Jul  2 14:54:00.298 SyntaxError: Unexpected token .  
>   
```

可以看到，看来是行不通。mongodb 无法识别 处理后的字段。
那只能自己生成一个objectId 再进行对比了。
我以前一直以为是无法进行两个 objectId 来进行对比的。
但查询了官方资料。并没有看到相关按时间生成新的objectID 的方法/函数。
(http://docs.mongodb.org/manual/reference/method/)

后来找到了开发牛人的一个便方，解决了此问题。代码如下：

```sh
#构建一个指定日期的objectId()  
进入mongo命令行
$mongo
> var timestamp = Math.floor(new Date(2014,03,01).getTime() / 1000);  #getTime() 返回毫秒数  
> var hex       = ('00000000' + timestamp.toString(16)).substr(-8); #前填充0  
> var v_objectId  = new ObjectId(hex + new ObjectId().str.substring(8)); #更换掉前面的时间值  

其它方法也比较简单，就是把一个日期转换成16进制后，替换到一个新生成的objectId 中去。
#用objectId() 来进行对比查询  
> db.order.find({_id:{$gt:v_objectId}})  
{ "_id" : ObjectId("533e1049e4271af009000005"), "md5" : "a99038f392284b7dac895dc0030486c8",  
...  
 }  

可以看到，查询结果出来了。从上面可以看出。两个objectId 是可以进行对比的。
```
从上面示例代码看到
虽然可以从ojbectId 中提取一些信息。但如果要使用 objectid中的信息来做为查询条件，相对还是比较麻烦的。
就象我上面的，如果我要从objectID 的建立日期 来做为查询条件，我自己得先用查询时间构建一个objectID，然后再进行查询。
如果查询条件还有范围什么的。还不如再建立一个字段create_dt 来保存记录的建立时间。以后查询统计用此字段，还更方便。



我们再来看看查询的计划，看到已使用了索引，此计划虽然不能说明什么。但也可以让我知道是用_id 字段的索引。
```sh
>> db.order.find({_id:{$gt:v_objectId}}).explain()  
{  
    "cursor" : "BtreeCursor _id_",  
    "isMultiKey" : false,  
    "n" : 3,  
    "nscannedObjects" : 3,  
    "nscanned" : 3,  
    "nscannedObjectsAllPlans" : 3,  
    "nscannedAllPlans" : 3,  
    "scanAndOrder" : false,  
    "indexOnly" : false,  
    "nYields" : 0,  
    "nChunkSkips" : 0,  
    "millis" : 0,  
    "indexBounds" : {  
        "_id" : [  
            [  
                ObjectId("533991005c1e4132ccbaf755"),  
                ObjectId("ffffffffffffffffffffffff")  
            ]  
        ]  
    },  
    "server" : "localhost.localdomain:27017"  
}  
```   
