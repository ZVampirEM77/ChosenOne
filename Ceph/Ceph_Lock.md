#Ceph Lock

## cls/lock/
### 目录结构
cls/lock/  
- cls\_lock.cc  
- cls\_lock\_client.h  
- cls\_lock\_client.cc  
- cls\_lock\_ops.h  
- cls\_lock\_ops.cc  
- cls\_lock\_types.h  
- cls\_lock\_types.cc

Ceph中的提交记录说明为

> cls_lock: objclass for advisory locking

    Providing an objclass to create and manipulate advisory
    locking. Also providing a client api to control it. A lock
    may either be exclusively locked or shared among multiple
    lockers. A locker is identified by the rados client name, and
    by a cookie-string.
    A lock may be assigned with a tag that every operation on that
    lock should use. A lock can be unlocked by the client that locked
    it, or may be broken by other clients.
    When a non-zero lock duration is assigned to a lock by a locker,
    that locker expires after that time duration.
    A lock may have a description.
    Locks on a specific object can be listed. Lockers of a specific
    lock can be enumerated (by get_info).

也就是说cls/lock/目录下的文件是用于实现Ceph advisory lock的。

cls\_lock\_types.h  -- Ceph advisory lock处理过程中所需要的数据类型定义  
cls\_lock\_types.cc  --  上述数据类型中所定义的处理操作实现  
cls\_lock\_ops.h  --  Ceph advisory lock所有操作所对应的op类的定义  
cls\_lock\_ops.cc  --  上述op类中所定义的处理操作的实现  
cls\_lock\_client.h  --  1、定义了rados::cls::lock命名空间下的操作；2、定义了Lock类；
cls\_lock\_client.cc  --  1、上述定义的rados::cls::lock命名空间下的操作的具体实现；2、Lock类成员函数的具体实现；  
cls\_lock.cc  --  Ceph advisory lock对外提供的各种接口的具体实现  

### 如何使用Ceph advisory lock
要了解这个问题，首先要知道为什么将其定义在cls/目录下。  
需要明确的一点是，cls目录对于Ceph的意义：  
cls是Ceph的一个模块扩展，它允许用户自定义对象的操作接口和实现方法，为用户提供了一种比较方便的接口扩展方式。

了解了cls目录的意义之后，就知道了，将advisory lock的实现放在cls目录下是为了能够为用户提供操作lock的接口，同时便于用户对其功能进行扩展。

第二个问题，advisory lock为用户提供了哪些操作接口？如何操作lock？
> rados --help  
> ...  
> ADVISORY LOCKS  
> &ensp;&ensp;lock list <obj-name>  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;List all advisory locks on an object  
> &ensp;&ensp;lock get <obj-name> <lock-name>  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Try to acquire a lock  
> &ensp;&ensp;lock break <obj-name> <lock-name> <locker-name>  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Try to break a lock acquired by another client  
> &ensp;&ensp;lock info <obj-name> <lock-name>  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Show lock information  
> &ensp;&ensp;options:  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;--lock-tag&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Lock tag, all locks operation should use the same tag  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;--lock-cookie &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Locker cookie  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;--lock-description &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Description of lock  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;--lock-duration&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Lock duration (in seconds)  
> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;--lock-type&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;Lock type (shared, exclusive)

在Ceph中，也可以直接实例化Lock类，设置name、cookie以及duration等参数，来使用
Ceph advisory lock。

### Lock类的参数含义
name -- 即所实例化创建的Lock对象的名称  
cookie -- 与name字段一同，用于唯一标识一个Lock实例  
tag -- all locks operation should use the same tag，在RGW使用中通常设置为zone id  
description -- Lock实例对应的描述信息  
duration -- 对应Lock实例lock操作的持续时间，对于一个非零的duration来说，过了指定时间间隔后，会自动释放lock

### 锁的类型
当前Ceph advisory lock支持 *互斥锁 (LOCK_EXCLUSIVE)* 和 *共享锁 (LOCK_SHARED)*两种类型的锁。

### example
rados::cls::lock::Lock l("test_lock");  
utime_t time(max\_lock\_secs, 0);  
l.set_duration(time);
std::string cookie = XXX(a random str);
l.set_cookie(cookie)
std::string oid;  
/* get oid*/  
...  
librados::IoCtx *ctx = store->get\_bl\_pool_ctx();  
op\_ret = l.lock\_exclusive(ctx, oid); &ensp;&ensp;//互斥锁  
if (op\_ret == -EBUSY) {  
&ensp;&ensp;// 锁已经被抢占了,等待若干秒后重新尝试抢占锁;   
}

if (op\_ret < 0) {
&ensp;&ensp;// get lock时出错  
} else {  
&ensp;&ensp;// 成功获取到锁，执行后面的互斥操作  
}




## 扩展问题
### common/RWLock.h


### 为什么要使用advisory lock？什么情况下需要使用advisory lock？