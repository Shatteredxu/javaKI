##### Join使用

```java
public class Main {
    public static void main(String[] args) {
        ThreadBoy boy = new ThreadBoy();
        boy.start();
    }
    static class ThreadBoy extends Thread{
        @Override
        public void run() {
            System.out.println("男孩和女孩准备出去逛街");
            ThreadGirl girl = new ThreadGirl();
            girl.start();
            try {
                girl.join();//等着girl线程执行完毕
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("男孩和女孩开始去逛街了");
        }
    }
    static class ThreadGirl extends Thread{
        @Override
        public void run() {
            int time = 5000;
            System.out.println("女孩开始化妆,男孩在等待。。。");
            try {
                Thread.sleep(time);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("女孩化妆完成！，耗时" + time);
        }
    }
}
```

### Join源码

```java
//当传入的millis等于0的时候，使用wait(0)来等待
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
        if (millis == 0) {
            while (isAlive()) {//线程还在运行中的话，则继续等待，如果指定时间到了，或者线程运行完成了
                wait(0);//wait是一个native方法，等待notifyAll()的调用去唤醒它。
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

在this线程结束后，会调用exit本地方法

```java
//线程退出函数：
void JavaThread::exit(bool destroy_vm, ExitType exit_type) ｛
...
//这里会处理join相关的销毁逻辑
ensure_join(this);
...
｝
 //处理join相关的销毁逻辑
 static void ensure_join(JavaThread* thread) {
      Handle threadObj(thread, thread->threadObj());

      ObjectLocker lock(threadObj, thread);

      thread->clear_pending_exception();

      java_lang_Thread::set_thread_status(threadObj(), java_lang_Thread::TERMINATED);

      java_lang_Thread::set_thread(threadObj(), NULL);

      //这里就调用notifyAll方法，唤醒等待的线程
      lock.notify_all(thread);

      thread->clear_pending_exception();
    } 
```



