---
layout: post
title: '基于ViewPager+Recyclerview实现的CalendarView视图(二)'
date: 2018-03-08
author: sung
cover: ''
tags: Android开发 CalendarView Recyclerview ContentProvider
---
懒癌时隔多年终于来更新第二篇了。。。该篇主要讲之前vp+rc实现的calendar的日期数据的保存，之前讲了VIewPager的Calendar的实现方式，针对日期相关的数据一直都是即时生成的方式，遗留下来了日期相关属性无法保存，每次都是新数据的问题。接下来讲讲如何结合实际需求使用ContentProvider保存date日期



## 开篇



##### 存什么表？

首先想到的有三种表存储方式，一个是按照每月的数据存一个表、二是按照ViewPager每页的数据存一个表、三简单粗暴整个一个表。考虑到每页的数据都掺差有上月和下月的日期，如果按月存表每一页都要访问三个数据库按月方式就作罢了，再一个对比按页存表和一整个表对比，按页存表就得存多个表，访问的时候肯定需要先前判断哪个表，又考虑到ViewPager的页数动态及各类型的判断，所以决定采用第三种方式。一整个表，访问简单也便于管理



##### 需要存什么？

既然是日历，那肯定是存日期了。再次明确与日期相关的属性什么必要存什么不必要存，目前date object内有如下的数据：

```java
public int _id = -1;//数据库中存的id
public int year;
public int month;
public int day;
public String YYMMDD;//年月日字符串
public String YYMM;//年月
public int position;//在grid中的位置
public int pagerIndex;//在viewpager中的页数位置
public boolean currentMonth = false;//当月标志
public boolean sellectStatus = false;//选中标志
```

年月日是肯定要存的，两个中文串可用可不用看需求存不存，因为查询的时候设置数据需要日期在当前页的位置数据所以把该日期所在的页数和在该页数所在的position也一并存了。剩下的currentMonth和sellectStatus，当月的bool值可与实际月份判断故不存，选中标志便于在恢复数据的时候能够正确记录之前的数据操作就存上了



## 正文



### 数据存储

使用ContentProvider存储日期数据



##### ContentProvider配置

第一步：新建一个基类 BaseProvider 继承自 ContentProvider，静态代码块定义uri和存储数据

```java
static {
	mUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
	//和CONTENT_URI保持一致
	mUriMatcher.addURI(Provider.AUTHORITY, "dates", DATES_DIR);
	mUriMatcher.addURI(Provider.AUTHORITY, "dates/#", DATES_ID);

	//定义存入库的数据
	mDatesProjectionMap = new HashMap<>();
	mDatesProjectionMap.put(Provider.DatesColumns._ID,Provider.DatesColumns._ID);
	mDatesProjectionMap.put(Provider.DatesColumns.POSITION,Provider.DatesColumns.POSITION);
	mDatesProjectionMap.put(Provider.DatesColumns.PAGER_INDEX,Provider.DatesColumns.PAGER_INDEX);
	mDatesProjectionMap.put(Provider.DatesColumns.YEAR,Provider.DatesColumns.YEAR);
	mDatesProjectionMap.put(Provider.DatesColumns.MONTH,Provider.DatesColumns.MONTH);
	mDatesProjectionMap.put(Provider.DatesColumns.DAY,Provider.DatesColumns.DAY);
    mDatesProjectionMap.put(Provider.DatesColumns.CURRENT_MONTH,Provider.DatesColumns.CURRENT_MONTH);
    mDatesProjectionMap.put(Provider.DatesColumns.SELLECT_STATUS,Provider.DatesColumns.SELLECT_STATUS);
    mDatesProjectionMap.put(Provider.DatesColumns.YYMMDD,Provider.DatesColumns.YYMMDD);
	mDatesProjectionMap.put(Provider.DatesColumns.YYMM,Provider.DatesColumns.YYMM);
}
```

重写增删改查四个方法

```java
	//查
    @Nullable
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        SQLiteQueryBuilder qb = new SQLiteQueryBuilder();
        qb.setTables(Provider.DatesColumns.TABLE_NAME);

        switch (mUriMatcher.match(uri)) {
            case DATES_DIR:
                qb.setProjectionMap(mDatesProjectionMap);
                break;

            case DATES_ID:
                qb.setProjectionMap(mDatesProjectionMap);
                qb.appendWhere(Provider.DatesColumns._ID + "=" + uri.getPathSegments().get(1));
                break;

            default:
                throw new IllegalArgumentException("Unknown URI " + uri);
        }

        // If no sort order is specified use the default
        String orderBy;
        if (TextUtils.isEmpty(sortOrder)) {
            orderBy = Provider.DatesColumns.DEFAULT_SORT_ORDER;
        } else {
            orderBy = sortOrder;
        }

        // Get the database and run the query
        SQLiteDatabase db = mDatabaseHelper.getReadableDatabase();
        Cursor c = qb.query(db, projection, selection, selectionArgs, null, null, orderBy);

        // Tell the cursor what uri to watch, so it knows when its source data changes
        c.setNotificationUri(getContext().getContentResolver(), uri);
        return c;
    }

    //增
    @Nullable
    @Override
    public Uri insert(Uri uri, ContentValues contentValues) {
        if (mUriMatcher.match(uri) != DATES_DIR){
            throw new IllegalArgumentException("Unknown URI"+uri);
        }

        ContentValues values;
        if (contentValues != null){
            values = new ContentValues(contentValues);
        }else {
            values = new ContentValues();
        }

        if (values.containsKey(Provider.DatesColumns.POSITION) == false)
            values.put(Provider.DatesColumns.POSITION,0);

        if (values.containsKey(Provider.DatesColumns.PAGER_INDEX) == false)
            values.put(Provider.DatesColumns.PAGER_INDEX,0);

        if (values.containsKey(Provider.DatesColumns.YEAR) == false)
            values.put(Provider.DatesColumns.YEAR,1993);

        if (values.containsKey(Provider.DatesColumns.MONTH) == false)
            values.put(Provider.DatesColumns.MONTH,1);

        if (values.containsKey(Provider.DatesColumns.DAY) == false)
            values.put(Provider.DatesColumns.DAY,1);

        //0 = false ; 1 = true
        if (values.containsKey(Provider.DatesColumns.CURRENT_MONTH) == false)
            values.put(Provider.DatesColumns.CURRENT_MONTH,0);
        if (values.containsKey(Provider.DatesColumns.SELLECT_STATUS) == false)
            values.put(Provider.DatesColumns.SELLECT_STATUS,0);

        if (values.containsKey(Provider.DatesColumns.YYMMDD) == false)
            values.put(Provider.DatesColumns.YYMMDD,"");
        if (values.containsKey(Provider.DatesColumns.YYMM) == false)
            values.put(Provider.DatesColumns.YYMM,"");

        SQLiteDatabase db = mDatabaseHelper.getWritableDatabase();
        long insert = db.insert(Provider.DatesColumns.TABLE_NAME, Provider.DatesColumns.POSITION, values);
        if (insert > 0 ){
            Uri uri1 = ContentUris.withAppendedId(Provider.DatesColumns.CONTENT_URI, insert);
            getContext().getContentResolver().notifyChange(uri1,null);
            return uri1;
        }
//        return null;
        throw new SQLException("Failed to insert row into " + uri);
    }

    //删
    @Override
    public int delete(Uri uri, String where, String[] whereArgs) {
        SQLiteDatabase db = mDatabaseHelper.getWritableDatabase();
        int count;
        switch (mUriMatcher.match(uri)){
            case DATES_DIR:
                count = db.delete(Provider.DatesColumns.TABLE_NAME,where,whereArgs);
                break;
            case DATES_ID:
                String noteId = uri.getPathSegments().get(1);
                count = db.delete(Provider.DatesColumns.TABLE_NAME, Provider.DatesColumns._ID + "=" + noteId
                        + (!TextUtils.isEmpty(where) ? " AND (" + where + ')' : ""), whereArgs);
                break;
            default:
                throw new IllegalArgumentException("Unknown URI " + uri);
        }
        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }

    //改
    @Override
    public int update(Uri uri, ContentValues values, String where, String[] whereArgs) {
        SQLiteDatabase db = mDatabaseHelper.getWritableDatabase();
        int count;
        switch (mUriMatcher.match(uri)) {
            case DATES_DIR:
                count = db.update(Provider.DatesColumns.TABLE_NAME, values, where, whereArgs);
                break;

            case DATES_ID:
                String noteId = uri.getPathSegments().get(1);
                count = db.update(Provider.DatesColumns.TABLE_NAME, values, Provider.DatesColumns._ID + "=" + noteId
                        + (!TextUtils.isEmpty(where) ? " AND (" + where + ')' : ""), whereArgs);
                break;

            default:
                throw new IllegalArgumentException("Unknown URI " + uri);
        }

        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }
```

注意两个常量，区分uri类型的，全部数据还是单个数据

```java
//all data
private static final int DATES_DIR = 1;
//single data
private static final int DATES_ID = 2;
```

```java
@Nullable
@Override
public String getType(Uri uri) {
	switch (mUriMatcher.match(uri)){
        case DATES_DIR:
            return Provider.CONTENT_TYPE;
        case DATES_ID:
            return Provider.CONTENT_ITEM_TYPE;
        default:
            throw new IllegalArgumentException("Unknown URI"+uri);
	}
}
```

第二步：清单文件增加标签（authorities和Provider常量类中AUTHORITY保持一致）

```xml
 <provider
 	android:authorities="com.sung.provider.calendarview"
 	android:name=".provider.BaseProvider"/>
```

第三步：创建一个Provider工具类，提供于增删改查入口

代码太长捡关键点说一下，首先增的方法

```java
public static int insert(Context context, DateObject date) {
	ContentValues values = new ContentValues();
    values.put(Provider.DatesColumns.POSITION, date.position);
    values.put(Provider.DatesColumns.PAGER_INDEX, date.pagerIndex);
    values.put(Provider.DatesColumns.YEAR, date.year);
    values.put(Provider.DatesColumns.MONTH, date.month);
    values.put(Provider.DatesColumns.DAY, date.day);
    values.put(Provider.DatesColumns.CURRENT_MONTH, date.currentMonth ? 1 : 0);
    values.put(Provider.DatesColumns.SELLECT_STATUS, date.sellectStatus ? 1 : 0);
    values.put(Provider.DatesColumns.YYMMDD, date.YYMMDD);
    values.put(Provider.DatesColumns.YYMM, date.YYMM);
    Uri uri = context.getContentResolver().insert(Provider.DatesColumns.CONTENT_URI, values);
    //Log.d("insert uri=" + uri);
    String lastPath = uri.getLastPathSegment();
    int _id = -1;
    if (TextUtils.isEmpty(lastPath)) {
    Log.d("insert failure!");
    } else {
    Log.d("insert success! the id is " + lastPath);
    }
    return _id;
}
```

通过上下文context.getContentResolver()获取contentresolver对象调用insert方法，此时会走BaseProvider重写的insert方法，将DateObject对象的各个数据以键值对的形式保存并返回一个_id，此id为数据库的唯一位置标示，绝不会重复

查的方法分为查单个或者查列表，查单个除开以_id查询只需查单个外其他查询条件必须要多个组合条件查单个，且组合条件内需要有唯一标识（eg：PAGER_INDEX + POSITION 构成唯一结果条件查询，其他需要处理成列表结果）。查列表的关键是sql语句的编写，示例代码如下

```java
/**
     * 根据pager index查整页
     * */
public static List<DateObject> query(Context context, int pagerIndex, boolean queryAll) {
	List<DateObject> dates = new ArrayList<>();
    Cursor c = context.getContentResolver().query(Provider.DatesColumns.CONTENT_URI,
    	new String[]{Provider.DatesColumns._ID,
    	Provider.DatesColumns.POSITION,
    	Provider.DatesColumns.PAGER_INDEX,
    	Provider.DatesColumns.YEAR,
        Provider.DatesColumns.MONTH,
        Provider.DatesColumns.DAY,
        Provider.DatesColumns.CURRENT_MONTH,
        Provider.DatesColumns.SELLECT_STATUS,
        Provider.DatesColumns.YYMMDD,
        Provider.DatesColumns.YYMM},
        Provider.DatesColumns.PAGER_INDEX + " = " + pagerIndex + ")" + " group by ( " + Provider.DatesColumns.POSITION,
        null, Provider.DatesColumns.POSITION + " asc");
    while (c != null && c.moveToNext()) {
        DateObject date = new DateObject();
        date._id = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns._ID));
        date.position = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns.POSITION));
        date.pagerIndex = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns.PAGER_INDEX));
        date.year = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns.YEAR));
        date.month = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns.MONTH));
        date.day = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns.DAY));
        date.currentMonth = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns.CURRENT_MONTH)) == 0 ? false : true;
        date.sellectStatus = c.getInt(c.getColumnIndexOrThrow(Provider.DatesColumns.SELLECT_STATUS)) == 0 ? false : true;
        date.YYMMDD = c.getString(c.getColumnIndexOrThrow(Provider.DatesColumns.YYMMDD));
        date.YYMM = c.getString(c.getColumnIndexOrThrow(Provider.DatesColumns.YYMM));

    	dates.add(date);
	}
	return dates;
}
```

关键在于query方法，第一个参数表的uri（Provider定义的常量URI），第二个参数为你查表需要返回的数据，即你需要返回数据库所保存的哪些字段（String数组形式）这里需要查表生成dateobject所以就全字段了，第三个是查询条件，相当于sql语句里的where，因为需要两个条件过滤所以用了一个group by的语句形式，第四个参数是配合第三个使用，如果第三个内有？那么第四个参数就会替换？，第五个参数是排序方式，这里传的是Provider.DatesColumns.POSITION + " asc"（以日期所在当前页的位置升序排列）

```java
//源码
public final @Nullable Cursor query(@RequiresPermission.Read @NonNull Uri uri,
            @Nullable String[] projection, @Nullable String selection,
            @Nullable String[] selectionArgs, @Nullable String sortOrder) {
	return query(uri, projection, selection, selectionArgs, sortOrder, null);
}
```

查出来的结构因为是列表所以用了一个while

```java
while (c != null && c.moveToNext()) {}
```

当游标不为空且有下一条数据的时候继续

配置步骤粗略就到这



##### ContentProvider相关

基础的一些配置讲了，再讲一下在配置过程中同步或者需要用到的。

SQLiteOpenHelper

必不可少的一个类，该类定义了数据库的版本和名字以及表的创建和更新

```java
/* 建表语句 */
sqLiteDatabase.execSQL("CREATE TABLE "+Provider.DatesColumns.TABLE_NAME+" ("
	+Provider.DatesColumns._ID+" INTEGER PRIMARY KEY,"
    +Provider.DatesColumns.POSITION + " INTEGER,"
    +Provider.DatesColumns.PAGER_INDEX + " INTEGER,"
    +Provider.DatesColumns.YEAR + " INTEGER,"
    +Provider.DatesColumns.MONTH + " INTEGER,"
    +Provider.DatesColumns.DAY + " INTEGER,"
    +Provider.DatesColumns.CURRENT_MONTH + " INTEGER,"
    +Provider.DatesColumns.SELLECT_STATUS + " INTEGER,"
    +Provider.DatesColumns.YYMMDD + " VARCHAR(11),"
    +Provider.DatesColumns.YYMM + " VARCHAR(11)"
    +");");
```

每次DataObject对象有变动和插入数据库时键值对的新增之类的都必须要提数据库的版本version。数据库名字定义一次最好不要变动。（小点解释一下：VARCHAR(11) == 11个字符之内的字符串）

表更新就是先drop再create。

Provider

该类是一个静态常量类，为避免存数据库的时候混乱，在该类中事先定义好存入的key

```java
// 这个是每个Provider的标识，在Manifest中使用
public static final String AUTHORITY = "com.sung.provider.calendarview";

public static final String CONTENT_TYPE = "vnd.android.cursor.dir/vnd.com.sung.provider.calendarview";

public static final String CONTENT_ITEM_TYPE = "vnd.android.cursor.item/vnd.com.sung.provider.calendarview";

public static final class DatesColumns implements BaseColumns{

	//通用uri
    public static final Uri CONTENT_URI = Uri.parse("content://"+AUTHORITY+"/dates");

    public static final String TABLE_NAME = "dates";
    public static final String DEFAULT_SORT_ORDER = "position desc";

    public static final String POSITION = "position";
    public static final String PAGER_INDEX = "pager_index";
    public static final String YEAR = "year";
    public static final String MONTH = "month";
    public static final String DAY = "day";
    public static final String CURRENT_MONTH = "currentmonth";
    public static final String SELLECT_STATUS = "sellectstatus";
    public static final String YYMMDD = "yy_mm_dd";
    public static final String YYMM = "yy_mm";

}
```

ContentProvider相关的可能表述的有些不够好，需要学习了解的可以专门搜一搜写这个的博文，有很多。



### 数据使用

再回到CalendarView上。文前说了使用ContentProvider存的是DateObject的数据，存储那一块已经ok接下来就是数据库数据何时用以及怎么用



场景一：要先用必须要先存。比较合适的时候就是日期数据初始化的时候把当页的数据存表，因为不能说每次初始化都能一遍，所以存之前先查，查到有当页数据就不存，反之存。DateAdapter中setDates设置数据方法是唯一数据设置入口，判断逻辑就加这

```java
public void setDates(List dates, int pagerIndex, boolean reset){
	if (dates == null)
		return;

	if (reset){
		this.dates.clear();
	}
    this.dates.addAll(dates);
    this.notifyDataSetChanged();

	//查询当前pagerindex是否存入数据库
    List<DateObject> result = ProviderMannager.query(mContext, pagerIndex, true);
    if (result.size() == 0) {
		Log.d("date query not exits !!");
        for (DateObject date : (List<DateObject>)dates) {
			ProviderMannager.insert(mContext, date);
		}
	}else {
		Log.d("date query exits !!");
	}
}
```

场景二：数据是以页的形式展示，所以每回使用只需要查询当页数据，ProviderMannager里的query查页的方法，因为取数据生成list的方法封装在了utils内的唯一方法所以只需要在DateList方法内重新生成日期数据的之前取一遍数据库

```java
/**
 * 获取显示在grid上的天数数组
 *
 * @param month 月
 * @param year  年
 * @param pagerIndex 日期所属页标（pager中）
 */
public static List<DateObject> DateList(Context context, int year, int month, int pagerIndex) {
	List<DateObject> dates = new ArrayList<>();

	//先查询判断数据库有没有存当前数据
	List<DateObject> result = ProviderMannager.query(context, pagerIndex, true);
    if (result.size() != 0){
		dates = result;
        return dates;
	}

	//未存重新初始化数据
    int firstDayPosition = FirstDayPosition(month);
    int lastMonth = month - 1;
    //当前1月 上月年份-1
    if (month == 0) {
		year = year - 1;
        lastMonth = 11;
	}

    //上月分总天数
    int lastTotalNum = TotalDays(lastMonth);
    //当前月份总天数
    int currentTotalNum = TotalDays(month);

    for (int i = 0; i < 35; i++) {
    	int day;
        boolean currentMonth = false;
        if (i < firstDayPosition) {
        	//上月位置
            day = lastTotalNum - (firstDayPosition - 1 - i);
		} else if (i >= firstDayPosition && i <= (firstDayPosition + currentTotalNum)) {
    		//当月位置
            day = i + 1 - firstDayPosition;
            currentMonth = true;
		} else {
			//下月位置
	        day = i - currentTotalNum - firstDayPosition;
    	}
        DateObject date = new DateObject(year, month, day);
        date.position = i;
        date.currentMonth = currentMonth;
        date.pagerIndex = pagerIndex;//传入月份在pager中的页数位置

		dates.add(date);
	}
    return dates;
}
```

场景三：日历所提供的选择的数据也存入了数据库，所以在adapter内的用户选择操作监听也需要更改数据库相对应的值

```java
/**
 * 状态变更
 * */
@Override
public void onClick(View view) {
	if (view instanceof RelativeLayout){
		DateViewHolder holder = (DateViewHolder) view.getTag();
        DateObject date = dates.get(holder.position);
        if (!date.currentMonth)
        	return;

		date.sellectStatus = !date.sellectStatus;

        this.notifyDataSetChanged();
        onCalendarDayClickListner.onClick(holder.position);

        // 更新数据库当前日期选中状态
        // date._id == -1即代表当前点击的date对象
        // query的未查询到有效的数据表id此时弃置不做更新处理
        if (date._id != -1) {
        	ProviderMannager.update(mContext, date._id, date.sellectStatus);
		}
	}
}
```



### 尾语

CalendarView初使的功能开发到这告一段落。后期研究一下如何直接compile。不是大牛，代码/描述有啥不对各位看官请指出，虚心接受，共同进步～～觉得还ok的话大佬们给个赞哈👍

上一篇：[基于ViewPager+Recyclerview实现的CalendarView视图(一)](http://zhouqipa.com/2017/07/21/%E5%9F%BA%E4%BA%8EViewPager+Recyclerview%E5%AE%9E%E7%8E%B0%E7%9A%84CalendarView%E8%A7%86%E5%9B%BE(%E4%B8%80).html)



🌹blog：[zhouqipa.com](http://www.zhouqipa.com)

[源代码](https://github.com/ssccbb/CalendarView)