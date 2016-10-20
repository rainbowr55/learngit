##ThreadLocal 学习笔记
####ThreadLocal的作用
在看Looper源码的时候接触到了ThreadLocal这个类，于是学习了下Android ThreadLocal的源码
先看看ThreadLocal用什么作用：

- 为不同线程提供不同的数据副本，可以在不同线程中互不干扰地存储并提供数据，
隔离了线程和线程,避免了线程访问实例变量发生冲突的问题.
- 执行逻辑与执行数据进行有效解耦，通过线程安全的共享对象来进行数据共享,可以有效避免在编程层次之间形成数据依赖,这也成为了XWork事件处理体系设计的核心
 
####ThreadLocal的Demo
```java

/**
 * Created by rainbow on 2016/10/19.
 */
public class AccountAdder {

    private static ThreadLocal<Person> accountThreadLocal = new ThreadLocal<Person>() {
        @Override
        protected Person initialValue() {
            Person person = new Person();
            person.setName("mainName");
            person.setAccountBalance(0);
            person.setAge(20);
            return person;
        }
    };

    public static Person getPerson() {
        return accountThreadLocal.get();
    }

    public static void addCount(String name,int year) {
        Person person = accountThreadLocal.get();
        person.setName(name);
        person.setAccountBalance(person.getAccountBalance() + 10000);
        person.setAge(person.getAge() + year);
    }
}


/**
 * Created by rainbow on 2016/10/19.
 */
public class PersonAccountThread extends Thread {
    public static final String TAG = "PersonAccountThread";

    private int year = 1;

    public void setYear(int year) {
        this.year = year;
    }

    @Override
    public void run() {
        for (int i = 0; i < 3; i++) {
            AccountAdder.addCount(this.getName(), this.year);
            Person person = AccountAdder.getPerson();
            Log.e(TAG, "thread [" + this.getName() + "]" + "- " + person.toString());
        }
    }
}


```
假设要给三个孩子提供分三次提供助学金，助学金发放时间分为 1年一次、两年一次、三年一次
列出三个孩子分别在几岁时的账户余额。

```java


    private void showResult() {
        PersonAccountThread threadOne = new PersonAccountThread();
        threadOne.setYear(1);
        PersonAccountThread threadTwo = new PersonAccountThread();
        threadTwo.setYear(2);
        PersonAccountThread threadThree = new PersonAccountThread();
        threadThree.setYear(3);
        threadOne.start();
        threadTwo.start();
        threadThree.start();
    }
```
结果如下：
```log
[Thread-4823]- Person{name='Thread-4823', age='21', accountBalance=10000}
[Thread-4823]- Person{name='Thread-4823', age='22', accountBalance=20000}
[Thread-4823]- Person{name='Thread-4823', age='23', accountBalance=30000}
[Thread-4824]- Person{name='Thread-4824', age='22', accountBalance=10000}
[Thread-4824]- Person{name='Thread-4824', age='24', accountBalance=20000}
[Thread-4824]- Person{name='Thread-4824', age='26', accountBalance=30000}
[Thread-4825]- Person{name='Thread-4825', age='23', accountBalance=10000}
[Thread-4825]- Person{name='Thread-4825', age='26', accountBalance=20000}
[Thread-4825]- Person{name='Thread-4825', age='29', accountBalance=30000}
```

####ThreadLocal的具体实现
```java	
    public class ThreadLocal<T> {
    
	    public ThreadLocal() {}
	 
	    public T get() {
	  
	        Thread currentThread = Thread.currentThread();
	        Values values = values(currentThread);
	        if (values != null) {
	            Object[] table = values.table;
	            int index = hash & values.mask;
	            if (this.reference == table[index]) {
	                return (T) table[index + 1];
	            }
	        } else {
	            values = initializeValues(currentThread);
	        }
	
	        return (T) values.getAfterMiss(this);
	    }
	
	    protected T initialValue() {
	        return null;
	    }
	
	    public void set(T value) {
	        Thread currentThread = Thread.currentThread();
	        Values values = values(currentThread);
	        if (values == null) {
	            values = initializeValues(currentThread);
	        }
	        values.put(this, value);
	    }
    
    }
```
   首先看ThreadLocal的set方法 在上面的set方法中，首先获取到当前的线程，紧接着获取线程中的localValues Values对象，如果为空则去初始化：
我们来看看如何初始化的

```java
  /**
     * Creates Values instance for this thread and variable type.
     */
    Values initializeValues(Thread current) {
        return current.localValues = new Values();
    }

```
创建了Value对象并赋值给线程中的localValues，然后初始化value对象里的值 这就为不同线程提供不同的数据副本，可以在不同线程中互不干扰地存储并提供数据。因为每个线程从同一个ThreadLocal对象获取到的Vaule都是各自线程new出来的，并且为了保证每个线程正确的从同一个ThreadLocal对象取自己的Value所以在进行存储Value的时候要使用对应起来， 下面看下ThreadLocal的值到底是怎么localValues中进行存储的。在localValues内部有一个数组：private Object[] table，ThreadLocal的值就是存在在这个table数组中，下面看下localValues是如何使用put方法将ThreadLocal的值存储到table数组中的，如下所示：
```java	

       /**
         * Sets entry for given ThreadLocal to given value, creating an
         * entry if necessary.
         */
        void put(ThreadLocal<?> key, Object value) {
            cleanUp();

            // Keep track of first tombstone. That's where we want to go back
            // and add an entry if necessary.
            int firstTombstone = -1;

            for (int index = key.hash & mask;; index = next(index)) {
                Object k = table[index];

                if (k == key.reference) {
                    // Replace existing entry.
                    table[index + 1] = value;
                    return;
                }

                if (k == null) {
                    if (firstTombstone == -1) {
                        // Fill in null slot.
                        table[index] = key.reference;
                        table[index + 1] = value;
                        size++;
                        return;
                    }

                    // Go back and replace first tombstone.
                    table[firstTombstone] = key.reference;
                    table[firstTombstone + 1] = value;
                    tombstones--;
                    size++;
                    return;
                }

                // Remember first tombstone.
                if (firstTombstone == -1 && k == TOMBSTONE) {
                    firstTombstone = index;
                }
            }
        }
 
```
上面的代码实现数据的存储过程，这里不去分析它的具体算法，但是我们可以得出一个存储规则，那就是ThreadLocal的值在table数组中的存储位置总是为ThreadLocal的reference字段所标识的对象的下一个位置，比如ThreadLocal的reference对象在table数组的索引为index，那么ThreadLocal的值在table数组中的索引就是index+1。最终ThreadLocal的值将会被存储在table数组中：table[index + 1] = value。上面分析了ThreadLocal的set方法，这里分析下它的get方法，如下所示：
```java	
	    public T get() {
	  
	        Thread currentThread = Thread.currentThread();
	        Values values = values(currentThread);
	        if (values != null) {
	            Object[] table = values.table;
	            int index = hash & values.mask;
	            if (this.reference == table[index]) {
	                return (T) table[index + 1];
	            }
	        } else {
	            values = initializeValues(currentThread);
	        }
	
	        return (T) values.getAfterMiss(this);
	    }

```
可以发现，ThreadLocal的get方法的逻辑也比较清晰，它同样是取出当前线程的localValues对象，如果这个对象为null那么就返回初始值，初始值由ThreadLocal的initialValue方法来描述，默认情况下为null，当然也可以重写这个方法，它的默认实现如下所示：

```java	
	    protected T initialValue() {
	        return null;
	    }

```
如果localValues对象不为null，那就取出它的table数组并找出ThreadLocal的reference对象在table数组中的位置，然后table数组中的下一个位置所存储的数据就是ThreadLocal的值。从ThreadLocal的set和get方法可以看出，它们所操作的对象都是当前线程的localValues对象的table数组，因此在不同线程中访问同一个ThreadLocal的set和get方法，它们对ThreadLocal所做的读写操作仅限于各自线程的内部，这就是为什么ThreadLocal可以在多个线程中互不干扰地存储和修改数据，理解ThreadLocal的实现方式有助于理解Looper的工作原理。

 