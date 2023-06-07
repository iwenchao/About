
使用，实现原理和细节，彼此区别，适用场景

# Runnable
## 声明

```
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}

```
## 使用

如果要实现多继承就得要用implements，Java 提供了接口 java.lang.Runnable 来解决上边的问题。

Runnable是可以共享数据的，多个Thread可以同时加载一个Runnable，当各自Thread获得CPU时间片的时候开始运行Runnable，同一个Runnable实例里面的资源是被共享的，所以使用Runnable更加的灵

```
public class RunnableTest {
	public static void main(String[] args) {
		MyRunnable runnable = new MyRunnable();
		Thread thread = new Thread(runnable);
		System.out.println("开始执行任务时间:  "+getNowTime());
		thread.start();
		System.out.println("启动任务之后时间:  "+getNowTime());
	}
	
	public static String getNowTime()
	{
		SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		return format.format(new Date());
	}
}
class MyRunnable implements Runnable
{
	@Override
	public void run() {
		try {
			Thread.sleep(3000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

```

# Thread

## 使用
继承Thread类，需要覆盖方法 run()方法，在创建Thread类的子类时需要重写 run(),加入线程所要执行的代即可。直接继承Thread类有一个很大的缺点，因为“java类的继承是单一的，extends后面只能指定一个父类”，所有如果当前类继承Thread类之后就不可以继承其他类

# Callable

## 声明
```
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}

```

## 使用
Runnable是执行工作的独立任务，但是它不返回任何值。如果你希望任务在完成的能返回一个值，那么可以实现Callable接口而不是Runnable接口。在Java SE5中引入的Callable是一种具有类型参数的泛型，它的参数类型表示的是从方法call()(不是run())中返回的值。

```

public class CallableTest {
	public static void main(String[] args) {
		//创建实现了Callable接口的对象
		MyCallable callable = new MyCallable();
		//将实现Callable接口的对象作为参数创建一个FutureTask对象
		FutureTask<String> task = new FutureTask<>(callable);
		//创建线程处理当前callable任务
		Thread thread = new Thread(task);
		//开启线程
		System.out.println("开始执行任务的时间: "+getNowTime());
		thread.start();
		//获取到call方法的返回值
		try {
			String result = task.get();
			System.out.println("得到返回值: "+result);
			System.out.println("结束执行get的时间: "+getNowTime());
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
	
	public static String getNowTime()
	{
		SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
		return format.format(new Date());
	}
}
class MyCallable implements Callable<String>
{
	@Override
	public String call() throws Exception {
		Thread.sleep(3000);
		return "call method result";
	}
}
```

## Future

Executor就是Runnable和Callable的调度容器，Future就是对于具体的Runnable或者Callable任务的执行结果进行

取消、查询是否完成、获取结果、设置结果操作。get方法会阻塞，直到任务返回结果(Future简介)。

```
/**
* @see FutureTask
 * @see Executor
 * @since 1.5
 * @author Doug Lea
 * @param <V> The result type returned by this Future's <tt>get</tt> method
 */
public interface Future<V> {
    //用来取消异步任务的执行。如果异步任务已经完成或者已经被取消，或者由于某些原因不能取消，则会返回false。
    //如果任务还没有被执行，则会返回true并且异步任务不会被执行。
    boolean cancel(boolean mayInterruptIfRunning);
    //判断任务是否被取消，如果任务在结束(正常执行结束或者执行异常结束)前被取消则返回true，否则返回false。
    boolean isCancelled();
    //判断任务是否已经完成，如果完成则返回true，否则返回false。需要注意的是：任务执行过程中发生异常、任务被取消也属于任务已完成，也会返回true。
    isDone():
    //获取任务执行结果，如果任务还没完成则会阻塞等待直到任务执行完成。如果任务被取消则会抛出CancellationException异常，
    //如果任务执行过程发生异常则会抛出ExecutionException异常，如果阻塞等待过程中被中断则会抛出InterruptedException异常。
    V  get():
    //带超时时间的get()版本，如果阻塞等待过程中超时则会抛出TimeoutException异常。
    V get(long timeout,Timeunit unit):

}

```


## FutureTask
FutureTask则是一个RunnableFuture<V>，而RunnableFuture实现了Runnbale又实现了Futrue<V>这两个接口，

```
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
 
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }


Runnable注入会被Executors.callable()函数转换为Callable类型，即FutureTask最终都是执行Callable类型的任务。该适配函数的实现如下 ：
        public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }

下面是RunnableAdapter
    /**
     * A callable that runs given task and returns given result
     */
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }

```


# 实现Runnable接口相比继承Thread类有如下优势：

可以避免由于Java的单继承特性而带来的局限；
增强程序的健壮性，代码能够被多个线程共享，代码与数据是独立的；
适合多个相同程序代码的线程区处理同一资源的情况。
 

# 实现Runnable接口和实现Callable接口的区别:

1. Runnable是自从java1.1就有了，而Callable是1.5之后才加上去的
2. Callable规定的方法是call(),Runnable规定的方法是run()
3. Callcble是可以有返回值的，具体的返回值就是在Callable的接口方法call返回的，并且这个返回值具体是通过实现Future接口的对象的get方法获取的，这个方法是会造成线程阻塞的；而Runnable是没有返回值的，因为Runnable接口中的run方法是没有返回值的；
当将一个Callable的对象传递给ExecutorService的submit方法，则该call方法自动在一个线程上执行，并且会返回执行结果Future对象。
同样，将Runnable的对象传递给ExecutorService的submit方法，则该run方法自动在一个线程上执行，并且会返回执行结果Future对象，但是在该Future对象上调用get方法，将返回null。
4. Callable里面的call方法是可以抛出异常的，我们可以捕获异常进行处理；但是Runnable里面的run方法是不可以抛出异常的，异常要在run方法内部必须得到处理，不能向外界抛出；
5. 运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。
6. 加入线程池运行，Runnable使用ExecutorService的execute方法，Callable使用submit方法。 

