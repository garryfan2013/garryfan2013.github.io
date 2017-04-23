---
layout:     post
title:      "JS Promise 异步任务控制"
subtitle:   "JS Promise"
date:       2017-04-23
author:     "Garry fan"
header-img: "img/post-bg-2015.jpg"
tags:
    - Javascript
    - Promise
---

## Promise多异步任务的顺序控制及并发管理

最近闲暇时在研究nodejs，试着写一个与数据库进行交互的模块，数据库的查询执行是异步任务，在处理一些稍微复杂且有依赖的业务逻辑时，不免牵扯到多个异步任务间的执行依赖，以及并发执行等问题，Promise作为一种标准的javascript异步任务处理规范，有必要值得好好研究学习。

下面的内容，就使用过程中涉及到的典型多异步函数执行场景，来举例说明如何使用promise进行异步任务控制。

### 多异步任务串行执行

利用回调的嵌套，也可以实现有依赖关系的异步函数间的执行处理，但是带来的弊病也是显而易见的，callback hell这种叫法就可以看出回调嵌套给广大程序猿朋友们带来的伤害...

下面是采用传统的回调方式来实现异步任务串行执行：

```javascript
var conut = 0
# 模拟的异步函数，返回count作为异步处理完成后返回的结果
var asyncJob = function (cb) {
    setTimeout(function () {
        cb(count++);
    }, 1000);
}

# 采用回调嵌套的方式实现任务的串行执行
asyncJob(function (c) {
    console.log('job_1 done: ' + c);
    asyncJob(function (c) {
        console.log('job_2 done: ' + c);
        asyncJob(function (c) {
            console.log('job_3 done: ' + c);
        });
    });
});
```

下面是采用Promise处理的方式

```javascript
# 用promise来包装异步任务函数
var asyncJobWityPromise = function () {
    return new Promise(function (resolve, reject) {
        asyncJob(function (c) {
            resolve(c);
        });
    });
}

asyncJobWithPromise()
.then(function (c) {
    console.log('job_1 done: ' + c);
    return asyncJobWithPromise();
})
.then(function (c) {
    console.log('job_2 done: ' + c);
    return asyncJobWithPromise();
})
.then(function (c) {
    console.log('job_3 done: ' + c);
})
```
>在Promise的then(onFufiiled, onRejected)方法里，如果传递的onFullfiled函数里面执行的是异步任务，那么需要显式的返回该异步任务的promise，否则后一层的then方法会立即执行，不会等待上一层的异步任务完成

### 多异步任务的并行执行

废话不多说，直接上代码

```javascript
var promiseArray = []
var jobCount = 0
while(jobCount++ < 10) {
    promiseArray.push(asyncJobWithPromise().then(function (c) {
        console.log('job' + jobCount + ' done: ' + c);
    }))
}

Promise.all(promiseArray)
.then(function () {
    console.log('All jobs done');
})
```

>所有的并发任务执行完成后，执行最终的回调，"All jobs done"

### 多串行任务流的并行执行

基本原理与多异步任务的并发执行一样，只不过提交给Promise.all方法的promiseArray里的各个promise是串行任务流最终返回的那个promise

```javascript
# 串行任务流1包含1个串行的异步任务
var jobPipeLine1 = asyncJobWithPromise()
    .then(function (c) {
        console.log('jobPipeLine1 job1 done');
        console.log('jobPipeLine1 done');
    });

# 串行任务流2包含2个串行的异步任务
var jobPipeLine2 = asyncJobWithPromise()
    .then(function (c) {
        console.log('jobPipeLine2 job1 done');
        return asyncJobWithPromise();
    })
    .then(function (c) {
        console.log('jobPipeLine2 job2 done');
        console.log('jobPipeLine2 done');
    });

# 串行任务流3包含3个串行的的异步任务
var jobPipeLine3 = asyncJobWithPromise()
.then(function (c) {
    console.log('jobPipeLine3 job1 done');
    return asyncJobWithPromise();
})
.then(function (c) {
    console.log('jobPipeLine3 job2 done');
    return asyncJobWithPromise();
})
.then(function (c) {
    console.log('jobPipeLine3 job3 done');
    console.log('jobPipeLine3 done');
});

Promise.all([jobPipeLine1, jobPipeLine2, jobPipeLine3])
.then(function () {
    console.log('All jobs done');
})
```
