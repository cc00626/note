### 说一说promise

#### Promise

Promise 对象是异步编程的一种解决方案。Promise 是一个构造函数，接收一个函数作为参数，返回一个 Promise 实例。一个 Promise 实例有三种状态，分别是pending、*fulfilled*  和 rejected。实例的状态只能由 pending 转变 *fulfilled*  或者 rejected 状态，并且状态一经改变，无法再被改变了。

状态的改变是通过传入的 resolve() 和 reject() 函数来实现的，当我们调用resolve回调函数时，会执行Promise对象的then方法传入的第一个回调函数，当我们调用reject回调函数时，会执行Promise对象的then方法传入的第二个回调函数，或者catch方法传入的回调函数。

Promise的实例有**两个过程**：

- pending -> fulfilled : Resolved（已完成）
- pending -> rejected：Rejected（已拒绝）

一旦从进行状态变成为其他状态就永远不能更改状态了。

 在通过new创建Promise对象时，我们需要传入一个回调函数，我们称之为executor 

✓ 这个回调函数会被立即执行，并且给传入另外两个回调函数resolve、reject； 

✓ 当我们调用resolve回调函数时，会执行Promise对象的then方法传入的回调函数； 

✓ 当我们调用reject回调函数时，会执行Promise对象的catch方法传入的回调函数；  

情况一：如果resolve传入一个普通的值或者对象，那么这个值会作为then回调的参数；

情况二：如果resolve中传入的是另外一个Promise，那么这个新Promise会决定原Promise的状态： 

情况三：如果resolve中传入的是一个对象，并且这个对象有实现then方法，那么会执行该then方法，并且根据then方法的结 果来决定Promise的状态：  

**then方法接受两个参数**： 

fulfilled的回调函数：当状态变成fulfilled时会回调的函数； 

reject的回调函数：当状态变成reject时会回调的函数；  

**Promise有三种状态，那么这个Promise处于什么状态呢？** 

当then方法中的回调函数本身在执行的时候，那么它处于pending状态； 

当then方法中的回调函数返回一个结果时，那么它处于fulfilled状态，并且会将结果作为resolve的参数；

 ✓ 情况一：返回一个普通的值； 

 ✓ 情况二：返回一个Promise；

 ✓ 情况三：返回一个thenable值； 

当then方法抛出一个异常时，那么它处于reject状态  

Promise有五个常用的方法：then()、catch()、all()、race()、finally。

**Promise.allSettled()** 方法以 promise 组成的可迭代对象作为输入，并且返回一个 [Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise) 实例。当输入的所有 promise 都已敲定时（包括传递空的可迭代类型），返回的 promise 将兑现，并带有描述每个 promsie 结果的对象数组。



### await，async
