>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 


记录一下 **Android子线程切回主线程** 的方法。

*  `view.post(Runnable action)`：
    ```
    textView.post(() -> {
        textView.setText("更新textView");
    });
    ```

* `activity.runOnUiThread(Runnable action)`：
    ```
    MainActivity.this.runOnUiThread(() -> {
        showIllegalClassDialog(illegalList);
    });
    ```

* **Handler机制**：
    ```
    Handler mainHandler = new Handler(Looper.getMainLooper());
    mainHandler.post(() -> {
        doSomething();
    });
    ```

* **AsyncTask**
    ```
    AsyncTask<String, Void, String> asyncTask = new AsyncTask<String, Void, String>() {
        @Override
        protected String doInBackground(String... strings) {
            return "1";
        }
        @Override
        protected void onPostExecute(String s) {
            textview.setText(s);
        }
    };
    asyncTask.execute("1", "2", "3");
    ```

*  **RxJava等线程切换库**：
    ```
    Observable.just("")
            .observeOn(AndroidSchedulers.mainThread())
            .doOnNext(s -> {
                textView.setText(s);
            });
    ```

以上
