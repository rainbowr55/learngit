
### 垃圾回收器有三大职责:


- 分配内存;
- 确保任何被引用的对象保留在内存中;
- 回收不能通过引用关系找到的对象的内存. 
![gc process](http://upload-images.jianshu.io/upload_images/851999-e93c3c96a9a1ba2b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### Heap和Stack
Heap内存是指java运行环境用来分配给对象和JRE类的内存. 是应用的内存空间.
Stack内存是相对于线程Thread而言的, 它保存线程中方法中短期存在的变量值和对Heap中对象的引用等.
Stack内存, 顾名思义, 是类Stack方式, 总是后进先出(LIFO)的.
我们通常说的GC的针对Heap内存的. 因为Stack内存相当于是随用随销的.
####JVM使用分代式的内存管理方式, 将Heap分成三代 --- 新生代, 老一代, 持久代
- 小GC执行非常频繁, 而且速度特别快.
大GC一般会比小GC慢十倍以上.
大小GC都会发出"Stop the World"事件, 也就是说中断程序运行, 直至GC完成. 这也是我们在App优化之消除卡顿中为什么说频繁GC会造成用户感知卡顿.

![head&stack](http://upload-images.jianshu.io/upload_images/851999-675c33a31cc6208d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
###GC流程
![](http://upload-images.jianshu.io/upload_images/851999-16c831585c684eb8.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####当Eden区域内存被分配完时, 小GC程序被触发:
![](http://upload-images.jianshu.io/upload_images/851999-ff339a2842dbfc41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####引用可达的对象会移到Survivor(幸存者)区域--S0, 然后清空Eden区域, 此时引用不可达的对象会直接删除, 内存回收
![](http://upload-images.jianshu.io/upload_images/851999-3ccdda6d0fae500a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####Eden再次满时
当Eden区域再次分配完后, 小GC执行, 引用可达的对象会移到Survivor(幸存者)区域, 而引用不可达的对象会跟随Eden的清空而删除回收.

需要注意的是, 这次引用可达的对象移动到的是S1的幸存者区.
而且, S0区域也会执行小GC, 将其中还引用可达的对象移动到S1区, 且年龄+1. 然后清空S0, 回收其中引用不可达的对象.

此时, 所有引用可达的对象都在S1区, 且S1区的对象存在不同的年龄

 
![](http://upload-images.jianshu.io/upload_images/851999-bc737169e99d3d5c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

####当Eden第三次满时, S0和S1的角色互换了 依此循环
![](http://upload-images.jianshu.io/upload_images/851999-aca606170dba22b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####当Survivor区的对象年龄达到"老年线"时
上面1~3循环, Survivor区的对象年龄也会持续增长, 当其中某些对象年龄达到"老年线", 例如8岁时, 它们会"晋升"到老年区.
![](http://upload-images.jianshu.io/upload_images/851999-9fb0b7b053c0a149.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


####如此1~4步重复, 大体流程是这样的
![](http://upload-images.jianshu.io/upload_images/851999-70906ccc1aacef03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)