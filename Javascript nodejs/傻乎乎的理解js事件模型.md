# 傻乎乎的理解事件模型

基本上一看博客，技术书籍，上面讲到 **JavaScript**是以单线程的方式运行的， 而从cpu的角度， 线程是cpu的执行代码的最小单元；按照我们这种c++ coder来说，那么如果我一个定义了一个函数或者说任务执行花了很多很多时间， 你后边的任务哪怕再着急，也不会被执行到的；

看下面这个代码：

```javascript
function sleep(ms) {
    ms += new Date().getTime();
    while(new Date() < ms){
        ;
    }
}
console.log("position 1");
setTimeout("alert('3 seconds!')",3000);
sleep(10000);//10s stop here
console.log("position 2");
```
这段代码就是设定了一个定时器“3秒后弹出一个alert框”；之后就在**当前任务**中, 那这段代码真的会三秒后会弹出alert对话框吗？事实上你应该知道了， 他是不会弹出来的，这也就是很多人说的js执行时单线程的，因为有个任务（函数）sleep一直在执行，直到10s之后sleep函数才退出。所以设置的定时器不会被触发，这里很多人开始解释会说js的消息循环、事件循环等等在很多做底层网络库或者其他相关的底层开发人员来说会很熟悉，但是对于大部分js的学习者来说这是一个难以理解和想象的模型，特别是对计算机系统了解不是很深入的人来说，就是一个恶梦；

那么我们该怎么理解事件模型呢，一个简单的例子就是银行排队办理业务的排队号， 所谓的js的事件循环，task队列无非就是这样一种情况，我们编写的任何js函数的执行都是这些队列里面的一个环节，也就是说一个任务会由很多函数和代码的执行组成，就像银行窗口办理一个人的任务时会有询问，交流，填表，输密码，签字确认等等环节一样，这些环节组成了这个task、或者称之为调用栈；利用这个对比，我们来分析上面的代码； 你设置了一个settimeout任务，就相当于你在银行的窗口取了一个号码，之后你就在排队的队伍里面等待着排在你前面的所有人办理完所有业务；而现在你取了号之后，有一个客户（即代码中当前执行的代码）在办理业务时睡着了（sleep），那么即使当你设置这个定时器的时候你告诉银行工作人员，三秒之后你来教我办理业务，但是因为不能再前一个业务没有办理完成之前离开工作岗位，所以你设置的timeout任务就被耽搁下来了.

