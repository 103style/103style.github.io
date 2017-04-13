相信很多码友都碰到过这种情况，在一个界面放了一个datepicker.

但是在5.0以上的手机上颜色显示的效果不怎么好。

就像下图这样，颜色处理的不怎么好。


一开始百度找解决办法，搜了一下没什么结果，只能啃官方的api了，然后就找到了。

其实这种效果很好处理。

只要在xml文件中设置一下属性就可以了


android:headerBackground
头部背景，设置这个属性为 #808080 就变下图这样了。是不是感觉好多了。





http://blog.csdn.net/lxk_1993/article/details/51351365

另外还有其他的属性：


android:calendarViewShown="false" 是否显示日历视图
android:firstDayOfWeek="" 设置日历星期第一天是哪一天
android:headerBackground="@color/gray" 头部的背景颜色
android:endYear="2100" 最后一年，例如2100
android:maxDate="12/31/2100" 日历视图的最大日期,格式为mm/dd/yyyy
android:minDate="01/01/1900" 日历视图的最小日期，格式为mm/dd/yyyy
android:spinnersShown="false" 是否显示下拉菜单
android:startYear="1940" 从哪一年开始 例如1940
android:calendarTextColor="@color/white"日历的列表文字颜色（Api 21 以上才能用）
android:datePickerMode="calendar" 定义部件的外观，有spinner和calendar两种选择（Api 21 以上才能用）
android:dayOfWeekBackground="@color/gray" 头部的星期的背景颜色（Api 21 以上才能用）
android:dayOfWeekTextAppearance="@color/gray" 头部的星期的文字外观（Api 21 以上才能用）
android:headerDayOfMonthTextAppearance="@color/white" 头部对应 号数 的文字外观（Api 21 以上才能用）
android:headerMonthTextAppearance="@color/white"头部对应 月份 的文字外观（Api 21 以上才能用）
android:headerYearTextAppearance="@color/white" 头部对应 年份 的文字外观（Api 21 以上才能用）
android:yearListItemTextAppearance="@color/white" 选择年的列表的文字外观（Api 21 以上才能用）
android:yearListSelectorColor="@color/gray" 选择年的列表中选中的颜色（Api 21 以上才能用）


如果你喜欢我的博客，请关注我。