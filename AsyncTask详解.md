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

###3、源码解析
注：本篇源码分析基于Andorid-21。

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

这里可以看到execute()最终调用了executeOnExecutor()，上面代码都比较简单，有疑惑的地方就是**mWorker**和**mFuture**，找到这两个变量的
初始化实在AsyncTast的构造方法中，代码如下：

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

WorkerRunnable是一个包含mParams变量，实现Callable接口的一个抽象类，mParams用来保存execute()方法传递的参数，代码如下：

    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }