### Greendao 条件查询数据报错 the bind value at index 2 is null

导致报错的方法： 
* `xxxDao.queryBuilder().where(xxxDao.Properties.XXX.eq(value).unique()`
* `xxxDao.queryBuilder().where(xxxDao.Properties.XXX.notEq(value).unique()`
* ...

先说下解决方法：**就是对以上的方法的调用传入的 value值做空判断。**
比如：  `xxxDao.queryBuilder().where(xxxDao.Properties.XXX.eq(value == null ? "" : value).unique()`

或者写一个统一处理的类，类似下面这样，
```
import org.greenrobot.greendao.Property;
import org.greenrobot.greendao.query.WhereCondition;

public class CheckNullProperty {

    public static WhereCondition eq(Property property, Object value) {
        if (value == null) {
            value = "";
        }
        return property.eq(value);
    }
    ...
}

```

然后这样调用 `xxxDao.queryBuilder().where(CheckNullProperty.eq(xxxDao.Properties.XXX, value))`

---

报错的原因是 **org.greenrobot.greendao.Property** 类方法传入的 **value**  为 **null**  导致的。

查看 以下`.eq(value)`的源代码可以知道，当 **value** 为 **null** 时，返回的 `PropertyCondition` 的 value 也是 **null**。
```
/** Creates an "equal ('=')" condition  for this property. */
public WhereCondition eq(Object value) {
  return new PropertyCondition(this, "=?", value);
}
```
```
public PropertyCondition(Property property, String op, Object value) {
    super(checkValueForType(property, value));
    ...
}
```
```
private static Object checkValueForType(Property property, Object value) {
    if (value != null && value.getClass().isArray()) {
        throw new DaoException("Illegal value: found array, but simple object required");
    }
    Class<?> type = property.type;
    if (type == Date.class) {
        ...
    } else if (property.type == boolean.class || property.type == Boolean.class) {
        if (value instanceof Boolean) {
            return ((Boolean) value) ? 1 : 0;
        } else if (value instanceof Number) {
           ...
        } else if (value instanceof String) {
           ...
        }
    }
    return value;
}
```
然后通过 断点 跟踪，发现最后报错的源头 `android.database.sqlite.SQLiteProgram`
```
public void bindString(int index, String value) {
    if (value == null) {
        throw new IllegalArgumentException("the bind value at index " + index + " is null");
    }
    bind(index, value);
}
```


