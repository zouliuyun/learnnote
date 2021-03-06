### bash并发编程和flock

在shell编程中，需要使用并发编程的场景并不多。我们倒是经常会想要某个脚本不要同时出现多次同时执行，比如放在crond中的某个周期任务，如果执行时间较长以至于下次再调度的时间间隔，那么上一个还没执行完就可能又打开一个，这时我们会希望本次不用执行。本质上讲，无论是只保证任何时候系统中只出现一个进程还是多个进程并发，我们需要对进程进行类似的控制。因为并发的时候也会有可能产生竞争条件，导致程序出问题。

我们先来看如何写一个并发的bash程序。当我们知道如何对命令进行作业控制和如何使用wait命令之后，我们就已经可以写一个简单的bash并发程序了。我们这次的例子稍微复杂一点，我们写一个bash脚本，创建一个计数文件，并将里面的值写为0。然后打开100个子进程，每个进程都去读取这个计数文件的当前值，并加1写回去。如果程序执行正确，最后里面的值应该是100，因为每个子进程都会累加一个1写入文件，我们来试试：

```sh
[zorro@zorrozou-pc0 bash]$ cat racing.sh
#!/bin/bash

countfile=/tmp/count

if ! [ -f $countfile ]
then
    echo 0 > $countfile
fi

do_count () {
    read count < $countfile
    echo $((++count)) > $countfile
}

for i in `seq 1 100`
do
     do_count &
done

wait

cat $countfile

rm $countfile

我们再来看看这个程序的执行结果：

[zorro@zorrozou-pc0 bash]$ ./racing.sh 
26
[zorro@zorrozou-pc0 bash]$ ./racing.sh 
13
[zorro@zorrozou-pc0 bash]$ ./racing.sh 
34
[zorro@zorrozou-pc0 bash]$ ./racing.sh 
25
[zorro@zorrozou-pc0 bash]$ ./racing.sh 
45
[zorro@zorrozou-pc0 bash]$ ./racing.sh 
5
```
多次执行之后，每次得到的结果都不一样，也没有一次是正确的结果。这就是典型的竞争条件引起的问题。当多个进程并发的时候，如果使用的共享的资源，就有可能会造成这样的问题。这里的竞争调教就是：当某一个进程读出文件值为0，并加1，还没写回去的时候，如果有别的进程读了文件，读到的还是0。于是多个进程会写1，以及其它的数字。解决共享文件的竞争问题的办法是使用文件锁。每个子进程在读取文件之前先给文件加锁，写入之后解锁，这样临界区代码就可以互斥执行了：
```sh
[zorro@zorrozou-pc0 bash]$ cat flock.sh
#!/bin/bash

countfile=/tmp/count

if ! [ -f $countfile ]
then
    echo 0 > $countfile
fi

do_count () {
    exec 3< $countfile
    #对三号描述符加互斥锁
    flock -x 3
    read -u 3 count
    echo $((++count)) > $countfile
    #解锁
    flock -u 3
    #关闭描述符也会解锁
    exec 3>&-
}

for i in `seq 1 100`
do
     do_count &
done

wait

cat $countfile

rm $countfile
[zorro@zorrozou-pc0 bash]$ ./flock.sh 
100
对临界区代码进行加锁处理之后，程序执行结果正确了。仔细思考一下程序之后就会发现，这里所谓的临界区代码由加锁前的并行，变成了加锁后的串行。flock的默认行为是，如果文件之前没被加锁，则加锁成功返回，如果已经有人持有锁，则加锁行为会阻塞，直到成功加锁。所以，我们也可以利用互斥锁的这个特征，让bash脚本不会重复执行。

[zorro@zorrozou-pc0 bash]$ cat repeat.sh
#!/bin/bash

exec 3> /tmp/.lock

if ! flock -xn 3
then
    echo "already running!"
    exit 1
fi

echo "running!"
sleep 30
echo "ending"

flock -u 3
exec 3>&-
rm /tmp/.lock

exit 0
```
-n参数可以让flock命令以非阻塞方式探测一个文件是否已经被加锁，所以可以使用互斥锁的特点保证脚本运行的唯一性。脚本退出的时候锁会被释放，所以这里可以不用显式的使用flock解锁。flock除了-u参数指定文件描述符锁文件以外，还可以作为执行命令的前缀使用。这种方式非常适合直接在crond中方式所要执行的脚本重复执行。如：

*/1 * * * * /usr/bin/flock -xn /tmp/script.lock -c '/home/bash/script.sh'
关于flock的其它参数，可以man flock找到说明。

下面描述如何准确控制并发的进程数目。
```sh
 #!/bin/bash
#-----------------------------------------------------------------------------------
# 此例子说明了一种用wait、read命令模拟多线程的一种技巧
# 此技巧往往用于多主机检查，比如ssh登录、ping等等这种单进程比较慢而不耗费cpu的情况
# 还说明了多线程的控制
#-----------------------------------------------------------------------------------

function a_sub { # 此处定义一个函数，作为一个线程(子进程)
sleep 3 # 线程的作用是sleep 3s
}


tmp_fifofile="/tmp/$$.fifo"
mkfifo $tmp_fifofile      # 新建一个fifo类型的文件
exec 6<>$tmp_fifofile      # 将fd6指向fifo类型
rm $tmp_fifofile


thread=15 # 此处定义线程数
for ((i=0;i<$thread;i++));do 
echo
done >&6 # 事实上就是在fd6中放置了$thread个回车符


for ((i=0;i<50;i++));do # 50次循环，可以理解为50个主机，或其他

read -u6 
# 一个read -u6命令执行一次，就从fd6中减去一个回车符，然后向下执行，
# fd6中没有回车符的时候，就停在这了，从而实现了线程数量控制

{ # 此处子进程开始执行，被放到后台
      a_sub && { # 此处可以用来判断子进程的逻辑
       echo "a_sub is finished"
      } || {
       echo "sub error"
      }
      echo >&6 # 当进程结束以后，再向fd6中加上一个回车符，即补上了read -u6减去的那个
} &

done

wait # 等待所有的后台子进程结束
exec 6>&- # 关闭df6
exit 0


sleep 3s，线程数为15，一共循环50次，所以，此脚本一共的执行时间大约为12秒
即：
15x3=45, 所以 3 x 3s = 9s
(50-45=5)<15, 所以 1 x 3s = 3s 
所以 9s + 3s = 12s

$ time ./multithread.sh >/dev/null
real        0m12.025s
user        0m0.020s
sys         0m0.064s


而当不使用多线程技巧的时候，执行时间为：50 x 3s = 150s。

此程序中的命令mkfifo tmpfile和linux中的命令mknod tmpfile p
效果相同。区别是mkfifo为POSIX标准，因此推荐使用它。该命令创建了一个先入先出的管道文件，并为其分配文件标志符6。管道文件是进程之间通信的一种方式，注意这一句很重要
exec 6<>$tmp_fifofile      # 将fd6指向fifo类型
如果没有这句，在向文件$tmp_fifofile或者&6写入数据时，程序会被阻塞，直到有read读出了管道文件中的数据为止。而执行了上面这一句后就可以在程序运行期间不断向fifo类型的文件写入数据而不会阻塞，并且数据会被保存下来以供read程序读出。
```
