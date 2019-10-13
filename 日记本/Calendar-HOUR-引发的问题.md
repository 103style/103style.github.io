```
    /**
     * 获取今天的开始时间
     *
     * @return 今天的开始时间
     */
    public static long getTodayStartTime() {
        Calendar calendar = Calendar.getInstance();
        calendar.set(Calendar.HOUR, 0); // 此处错误 应该用  Calendar.HOUR_OF_DAY
        calendar.set(Calendar.MINUTE, 0);
        calendar.set(Calendar.SECOND, 0);
        return calendar.getTimeInMillis() / 1000;
    }
```
以上代码为获取 **今天的起始秒数**。
如果你在 **上午** 时间测试的话，这个返回值时没问题的；
但是如果你在 **下午** 测试，你就会发现这个值 **多了  12 * 60 * 60 （12个小时）**。

---

以下是 `Calendar.HOUR` 和 `Calendar.HOUR_OF_DAY` 的源代码注释。
```
    /**
     * Field number for <code>get</code> and <code>set</code> indicating the
     * hour of the morning or afternoon. <code>HOUR</code> is used for the
     * 12-hour clock (0 - 11). Noon and midnight are represented by 0, not by 12.
     * E.g., at 10:04:15.250 PM the <code>HOUR</code> is 10.
     *
     * @see #AM_PM
     * @see #HOUR_OF_DAY
     */
    public final static int HOUR = 10;

    /**
     * Field number for <code>get</code> and <code>set</code> indicating the
     * hour of the day. <code>HOUR_OF_DAY</code> is used for the 24-hour clock.
     * E.g., at 10:04:15.250 PM the <code>HOUR_OF_DAY</code> is 22.
     *
     * @see #HOUR
     */
    public final static int HOUR_OF_DAY = 11;
```
看源码注释，我们可以了解到:
* `Calendar.HOUR`：设置的为 `12小时制的值`，设置为0， **上午** 是 `0点`，**下午** 则是 `12点`。
* `Calendar.HOUR_OF_DAY`：设置的为 `24小时制的值`，设置为 `0` 即为 `0点`。

所以，开头的代码需要修改为：
```
    /**
     * 获取今天的开始时间
     *
     * @return 今天的开始时间
     */
    public static long getTodayStartTime() {
        Calendar calendar = Calendar.getInstance();
        calendar.set(Calendar.HOUR_OF_DAY, 0);
        calendar.set(Calendar.MINUTE, 0);
        calendar.set(Calendar.SECOND, 0);
        return calendar.getTimeInMillis() / 1000;
    }
```
