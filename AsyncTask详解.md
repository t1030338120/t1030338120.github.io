##Android AsyncTask 源码解析

###1、概述
AsyncTask是Android框架提供给开发者的一个辅助类，使用该类我们可以轻松的处理异步线程与主线程的交互.


下面是AsyncTask的简单Demo，来源官方文档
<pre><code>
    private class DownloadFilesTask extends AsyncTask<URL, Integer, Long> {
     protected Long doInBackground(URL... urls) {
         int count = urls.length;
         long totalSize = 0;
         for (int i = 0; i < count; i++) {
             totalSize += Downloader.downloadFile(urls[i]);
             publishProgress((int) ((i / (float) count) * 100));
             // Escape early if cancel() is called
             if (isCancelled()) break;
         }
         return totalSize;
     }

     protected void onProgressUpdate(Integer... progress) {
         setProgressPercent(progress[0]);
     }

     protected void onPostExecute(Long result) {
         showDialog("Downloaded " + result + " bytes");
     }
    }


    new DownloadFilesTask().execute(url1, url2, url3);
</pre></code>


###问题
* onPreExecute()方法什么时候被调用
* doInBackground()怎么被调用执行的
* doInBackground()执行完毕，怎么切换到UI线程并执行onPostExecute()


###3、源码解析
注：本篇源码分析基于Andorid-23

####问题1、onPreExecute()方法什么时候被调用
这里从AsyncTask的启动入口来分析：查看AsyncTask的execute()方法源码:

    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING; //设置当前AsyncTask的状态为RUNNING，上面的switch也可以看出，每个异步任务在完成前只能执行一次。
        onPreExecute(); //执行了onPreExecute()，当前依然在UI线程，所以我们可以在其中做一些准备工作

        mWorker.mParams = params;//将我们传入的参数赋值给mWorker.mParams
        exec.execute(mFuture);//开启线程池，后台执行任务

        return this;
    }


上面代码我们一下发现了onPreExecute()方法，**第一个问题的答案就有了**
<br>
####问题2、doInBackground()怎么被调用执行的
继续寻找第二个问题的答案，向下看发现了**mWorker**和**mFuture**这两个变量，这两个变量是干什么的？找一下它们的初始化代码，在AsyncTast的构造方法中找到了，代码如下：

    public AsyncTask() {
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }
        };

        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }

* mWorker是一个Callable对象，快速的看一下代码，在mWorker的call()方法中，发现了我们doInBackground()方法，   看到这里第二个问题就有点头绪了。

* mFuture本质是一个Runable对象，并且在初始化的时候将mWorker作为参数传入。

再次回到***executeOnExecutor()***往下看，有这么一行代码***exec.execute(mFuture);***，这行代码就是通过线程池exec来开启一个后台线程（线程池后面会有详解讲解），既然是开启线程，run()方法肯定会执行的，这里Runable对象是mFuture,下面就看一下mFuture的<B>run()</B>方法:

    public void run() {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;//callable是构造方法时传入
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

上面代码中我们看到result = c.call(),找一下c的初始化就是callable,
callable是构造方法传入，mFuture的初始方法 mFuture = new FutureTask<Result>(mWorker) {...}，看到这里我们就知道了callback就是mWorker对象了，再次看一下mWorker的call()方法，就会发现我们熟悉的doInBackgroud()方法。OK看到这里doInBackground()的执行过程已经清晰了。<br>
这里再次总结下：*1.executeOnExecutor()方法里面通线程池开启后台线程**（exec.execute(mFuture)）*，后台线程通过mWorker.call()方法回调了doInBackgroud()，这就实现了doInBackgroud()方法的后台运行**。


####问题3、doInBackground()执行完毕，怎么切换到UI线程并执行onPostExecute()
我们在写代码的时候在子线程中更新界面（异步切换线程）,最简单的做法就是通过Handler来实现。AsyncTast也是通过Handler来从实现线程切换的。接着看mWorker的call()方法：
            
         public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                return postResult(result);
            }

在call()方法中，将doInBackground()的执行结果传入了postResult()中，继续看一下postResult()方法源码：
    
    private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }

看到这里基本已经清晰了，上面代码通过getHandler()方法获取Handler对象发出了一条消息，消息中携带了MESSAGE_POST_RESULT常量和一个表示任务执行结果的AsyncTaskResult对象。这里通过Handler发生一个消息，然后肯定会在Handler的handleMessage()方法中处理。

下面是getHandler()代码

    private static Handler getHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler();
            }
            return sHandler;
        }
    }

    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());//确保即使在子线程中调用execute()方法，onPostExecute（）也在UI线程
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }


result.mTask就是一个AsyncTast对象，看一下它的finish()方法。


    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

在这里我们看到调用了onPostExecute()方法。整个流程已经清楚了。