# promise-in-ES6

1.
Javascript 最基本的异步编码形式是回调。在某些场景下，程序会发起一些异步操作，比如 XHR 和 timer，当初始化这些对象时，它会传递一个回调函数给这个程序或者说当前这个调用环境，它相信当这些异步操作结束的时候（比如xhr请求返回了，timer时间到了），这个回调函数会被放到调用栈里，当调用栈里在它之前的操作都已经被执行的时候，它就会被执行。
这样操作异步有几个问题：
1.当异步操作完成的时候，只有调用者才能被通知到，其他可能对这个异步操作也感兴趣的部分
   无法被通知到，除非回调函数对它进行了通知。
   
2.错误处理非常困难。同时处理几个异步操作的话管理起来也很困难。完全依赖于回调函数自己
去处理这些事。

3.回调函数需要负责很多事情。它必须要处理异步操作的结果，必须要通知其他基于异步操作结果而执行操作。
这种多任务也是回调函数的一个痛点。

来看一个案例：

```
function getCompanyFromOrderId(orderId){
  getOrder(orderId,function(order){
    getUser(order.userId,function(user){
       getCompany(user.companyId,function(company){
          //do something with company
       });
    });
  });
}
```
解决：Promise
Promise是一个对象，它会去监听异步操作的结果并进行处理。换句话说，promise 能够承诺在异步
操作结束的时候提醒你。Promise的一大优势是它是可组合的。你可以把代表两个不同的异步操作的
promise 链接到一起，这样其中一个就会在另一个之后发生，或者你可以等两个promise都执行完之后再
去执行其他的操作。promise的另一个优势是可以减少回调函数的职责。既然有了提醒机制，就不需要依靠
回调函数来履行通知的职责了。

promise一般是有两个部分组成的：control 和 Promise。 
control 指的是对promise对象状态的控制，比如成功还是失败。
第二部分是promise对象本身，这个对象可以被当做参数传递，它可以让对它感兴趣的其他操作来
注册监听，使得他们可以在异步操作结束的时候执行一些动作。这样的话，流程控制就不再是一个回调
函数的责任了。更加强大的是，如果异步操作有错误，promise也会发起通知,这样，错误处理也就不必
在回调函数中处理。

对上一段代码进行Promise方式的修改：

```
function getCompanyFromOrderId(orderId){
  getOrder(orderId).then(function(order){
    return getUser(order.userId);
  }).then(function(user){
    return getCompany(user.companyId);
  }).then(function(company){
      //do something with company
  }).then(undefined,function(error){
    //handle error
  });
}
```
有一个需要注意的地方是，即使我们立即执行了resolve或者reject某个promise,给promise的回调其实是异步执行的。比如
```
var test=1;
console.log(test);

var promise=new Promise(function(resolve,reject){
  resolve();
});

promise.then(function(){
  test=2;
  console.log(test);
});

test=3;
console.log(test);

执行结果：
1
3
2
```
说明：虽然promise里立即resolve了，但then里的回调函数还是被推入了调用栈中，进行异步执行。

实现一下前面的三个基本的方法：

```
function getOrder(orderId){
  return Promise.resolve({userId:35});
}

function getUser(userId){
  return Promise.resovle({companyId:18});
}

function getCompany(companyId){
  return Promise.resovle({name:'IBM'});
}
```
因为每个方法都返回promise,所以我们的代码可以写成：
```
  getOrder(3).then(function(order){
    return getUser(order.userId);
  }).then(function(user){
    return getCompany(user.companyId);
  }).then(function(company){
      //do something with company
  }).catch(function(error){
      //handle error
  });

```
如果希望等待所有的异步方法执行完之后再执行回调，可以使用Promise.all方法。
```
var courseIds=[1,2,3];
var promises=[];
for(var i=0;i<courseIds.length;i++){
  promises.push(getOrder(courseIds[i]));
}
Promise.all(promises).then(function(values){
  console.log(`the values is `);
  console.log(values);
});

执行结果：

"the values is "
[[object Object] {
  userId: 35
}, [object Object] {
  userId: 35
}, [object Object] {
  userId: 35
}]
```
但由于这是异步操作，所以不能期待values里的值和promise的调用顺序是对应的。

###  Asynchronous Generators

假设有一个功能，每隔0.5秒输出一段文字，如果代码可以像下面这样写，是不是非常爽？
```
console.log('start');
pause(500);
console.log('middle');
pause(500);
console.log('end');

pause方法的实现：
function pause(delay){
  setTimeout(function(){
    console.log(`paused for ${delay} ms`);
  },delay);
}
但是执行结果会是这样：

"start"
"middle"
"end"
"paused for 500 ms"
"paused for 500 ms"

```

显然，这和我们的预期不符，因为setTimeout是异步执行的。如果我们想要得到正确的结果，我们的代码要改成这样：
```
console.log('start');
pause(500,function(){
  console.log('middle');
  pause(500,function(){
    console.log('end');
  });
});

function pause(delay,cb){
  setTimeout(function(){
    console.log(`paused for ${delay} ms`);
    cb();
  },delay);
}

执行结果:
"start"
"paused for 500 ms"
"middle"
"paused for 500 ms"
"end"

```
但显然，这个代码的可读性很差。幸好，ES2015的generator给我们提供了解决方案。

#### generator

一个generator是一个会产生iterator ( 迭代器 ) 的函数。generator需要一个星号和一个yield的关键字。
在generator里，不是用return 去返回一个值，而是用yield返回多个值。
比如：
```
let numbers=function *(){
  yield 1;
  yield 2;
  yield 3;
  yield 4;
  yield 5;
}

let sum=0;
let iterator=numbers();
let next=iterator.next();
while(!next.done){
  sum+=next.value;
  next=iterator.next();
}
console.log(sum);
执行结果：
15
```
yield关键字非常重要，函数每次在使用yield的时候都会产生一个值，generator会返回一个迭代器对象，通过调用迭代器对象的next方法可以让我们访问到generator yield的每一个项。



```
function* main(){
  console.log('start');
  yield pause(500);
  console.log('middle');
  yield pause(500);
  console.log('end');
}

(function(){
  var sequence;
  
  var run=function(generator){
    sequence=generator();
    var next=sequence.next();
  }
  var resume=function(){
    sequence.next();
  }
  window.async={
    run:run,
    resume:resume
  }
})();

function pause(delay){
  setTimeout(function(){
     console.log(`paused for ${delay} ms`);
     async.resume();
  },delay);
}

async.run(main);

执行结果：
"start"
"paused for 500 ms"
"middle"
"paused for 500 ms"
"end"
```
不过一般情况下，我们都会从异步函数里返回数据。假设下列场景：
```
/*getStockPrice和executeTrade都是异步方法，后者根据前者返回的数据决定是否执行*/
var price =getStockPrice();
if(price > 45){
  executeTrade();
}else{
  console.log('trade not made');
}
```
我们用generator的方式来写这些代码。
```
function* main(){
  var price =yield getStockPrice();  //generator在yielding的时候会返回值
  if(price > 45){
    yield executeTrade();
  }else{
     console.log('trade not made');
  }
}

```
然后需要修改前面写的resume方法，修改如下:(其实就是加了一个value参数)
```
 var resume=function(value){
    sequence.next(value);
  }
```
这样写的话，任何调用resume的代码，如果它传递了一个value进去，这个value就将是
yield语句的返回值。getStockPrice和executeTrade方法实现如下：
```
function getStockPrice(){
  //$.get('/price',function(prices){});
  //模拟代码
  setTimeout(function(){
    async.resume(50);
  },300);
}
function executeTrade(){
  setTimeout(function(){
    console.log('trade completed');
    async.resume(); //不返回数据
  },300);
}
```
### generator中的错误处理
我们可能会希望能够这样捕捉错误：
```
function* main(){
 try {
       var price =yield getStockPrice();
       if(price > 45){
         yield executeTrade();
       } else {
         console.log('trade not made');
       }
     } catch(ex){
         console.log(ex.message);
     }
}
然后我们的getStockPrice中抛出错误：
function getStockPrice(){
  setTimeout(function(){
    throw Error（"there was a problem");
    async.resume(50);
  },300);
}
```
然而上述的错误处理并没有按照我们预期的那样工作，这个错误会被浏览器抛出，并没有打印在控制台上。
我们可以在async对象上添加一个方法：
```
(function(){
   ...
   //调用fail方法就好像yield语句自己抛出一个异常，这个和调用next方法会返回一个值很相似
   //只不过他返回的是一个异常
   var fail=function(reason){
    sequence.throw(reason); 
   }
   
   window.async={
     ...
     fail:fail
   }
   ...

})();

```
getStockPrice方法可以这么写：
```
function getStockPrice(){
  setTimeout(function(){
    try{
      throw Error("there was a problem");
      async.resume(50);
    } catch(ex){
      async.fail(ex);
    }
  },300);
}
```
那么上面这些异步函数的实现还有什么问题呢？
很明显，像getStockPrice里，需要有一个 async.resume()的存在，这样的话，这个方法离开了generator就没有了意义。
而且之前也提到过，回调函数不应该去做控制流程的工作，它只应该关注自己的业务。
和promise对象合作可以解决这个问题。
```
(function(){
  var run=function(generator){
    var sequence;
    var process=function(result){
      result.value.then(function(value){
        if(!result.done){
          process(sequence.next(value));
        }
      },function(error){
         if(!result.done){
           process(sequence.throw(error));
         }
      });
    };
    sequence=generator();
    var next=sequence.next();
    process(next);
  };
  
  window.asyncP={
    run:run
  };
})();

function getStockPriceP(){
  return new Promise(function(resolve,reject){
    setTimeout(function(){
     resolve(50);
    },300);
  });
}
function executeTradeP(){
  return new Promise(function(resolve,reject){
    setTimeout(function(){
      console.log('executeTrade');
      //resolve();
      reject(Error('failure'));
    },300);
  });
}
 function* main(){
  try{
     var price=yield getStockPriceP();
     if(price>45){
       yield executeTradeP();
     }else{
       console.log('trade not made');
     }
    } catch(ex){
       console.log('error'+ex.message);
   }
 }
asyncP.run(main);

```
注意，这里sequence.next(value)中的value是 then的函数参数，也就是then对应的
promise里resolve的数据。这样，任何一个用promise来呈现的异步方法都可以是
generator的一部分。就不会像之前的需要写特定为generator调用的函数。























