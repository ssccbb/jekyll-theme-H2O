---
layout: post
title: '基于ViewPager+Recyclerview实现的CalendarView视图(一)'
date: 2017-07-21
author: sung
cover: ''
tags: Android开发 CalendarView Recyclerview
---

一直想着自己搭一个日常记事和带日历打卡类的app，懒癌！这几天项目不是很忙之后仿ios的habitify先做了一个日历带打卡的小功能。先赌为快

![calendar](https://github.com/ssccbb/ssccbb.github.io/blob/master/assets/cover/calendar.gif?raw=true)



### 正片

#### 一句话实现思路

继承自RelativeLayout的自定义容器，里头包含了一个viewpager，viewpager通过inflate添加Recyclerview。通过calendar初始化rc数据集。接口实现点击日期的变化监听。



#### 代码

主视图calendarview继承RelativeLayout添加了一个viewpager

```java
public class CalendarView extends RelativeLayout implements ViewPager.OnPageChangeListener,DateAdapter.onCalendarDayClick{}
```

```java
private void initView(){  
        View view = LayoutInflater.from(context).inflate(R.layout.layout_calendar_view, null);  
        mCalendarTime = (TextView) view.findViewById(R.id.calendar_time_text);  
        mCalendarPager = (ViewPager) view.findViewById(R.id.calendar_view_pager);  
  
        DataInit();  
        PagerInit();  
        this.addView(view);  
    } 
```

初始化了默认数量的pager页，reSetPagerDate()设置当前页日期数据

```java
/** 
    * 共x页 中间当月 前后各x/2月 
    * */  
    private void PagerInit(){  
        for (int i = 0; i < DEFAULT_MONTH_NUM; i++) {  
            View pager = LayoutInflater.from(context).inflate(R.layout.layout_calendar_pager, null);  
            RecyclerView date = (RecyclerView) pager.findViewById(R.id.calendar_recycle);  
            date.setLayoutManager(new GridLayoutManager(context,7));  
            DateAdapter adapter = new DateAdapter(null);  
            adapter.setOnCalendarDayClickListner(this);  
            date.setAdapter(adapter);  
            date.setHasFixedSize(true);  
            pager.setTag(date);  
            views.add(pager);  
        }  
        PagerAdapter adapter = new PagerAdapter(views);  
        mCalendarPager.setAdapter(adapter);  
        CURRENT_PAGER_SELECT = LAST_PAGER_SELECT = DEFAULT_MONTH_NUM/2;  
        mCalendarPager.setCurrentItem(CURRENT_PAGER_SELECT);  
        mCalendarPager.setOnPageChangeListener(this);  
  
        //notify  
        reSetPagerDate();  
    }  
```

这儿重写viewpager的滑动监听，切换页的时候也重置切换页的数据集

```java
@Override  
    public void onPageSelected(int position) {  
        LAST_PAGER_SELECT = CURRENT_PAGER_SELECT;  
        CURRENT_PAGER_SELECT = position;  
  
        changeTimeLine();  
  
        //notify  
        RecyclerView date = (RecyclerView) views.get(position).getTag();  
        DateAdapter adapter = (DateAdapter) date.getAdapter();  
        adapter.setDates(CalendarUtils.DateList(mDate.year, mDate.month),true);  
  
    }  
```

数据集的操作我另写了一个工具类CalendarUtils，datelist方法是数据集的主要获取方法。通过android中Calendar类获取到当月第一天的星期数（周日=0），通过得到的星期来确定在rc（recyclerview）中角标位置，然后计算当月的上月和下月的总天数，当月1号之前依次从角标位置大到小上月总天数递减，当月最后一天依次角标位置小到大从1开始递增。贴代码，根据注释理解

```java
	//大月  
    private static int[] months_31_days = {0,2,4,6,7,9,11};  
    //小月  
    private static int[] months_30_days = {3,5,8,10};  
  
    /** 
     * 获取该月第一天在grid显示的position 
     * */  
    private static int FirstDayPosition(int month){  
        try {  
            Calendar calendar = Calendar.getInstance(TimeZone.getTimeZone("GMT+08:00"));  
            SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-M-d");  
            Date date = dateFormat.parse(calendar.get(Calendar.YEAR)+"-"+month+"-1");  
            calendar.setTime(date);  
            return calendar.get(Calendar.DAY_OF_WEEK)-1;  
        } catch (ParseException e) {  
            e.printStackTrace();  
        }  
  
        return 0;  
    }  
  
    /** 
     * 获取显示在grid上的天数数组 
     * @param month 月 
     * @param year 年 
     * */  
    public static List<DateObject> DateList(int year, int month){  
        List<DateObject> dates = new ArrayList<>();  
        int firstDayPosition = FirstDayPosition(month);  
  
        int lastMonth = month - 1;  
        //当前1月 上月年份-1  
        if (month == 0) {  
            year = year - 1;  
            lastMonth = 11;  
        }  
  
        int lastTotalNum = TotalDays(lastMonth);  
        int currentTotalNum = TotalDays(month);  
        for (int i = 0; i < 35; i++) {  
            int day;  
            boolean currentMonth = false;  
            if (i<firstDayPosition){  
                day = lastTotalNum - (firstDayPosition - 1 - i);  
            }else if (i >= firstDayPosition && i <= (firstDayPosition + currentTotalNum)){  
                day = i + 1 - firstDayPosition;  
                currentMonth = true;  
            }else {  
                day = i - currentTotalNum - firstDayPosition;  
            }  
            DateObject date = new DateObject(year, month, day);  
            date.position = i;  
            date.currentMonth = currentMonth;  
            dates.add(date);  
        }  
        return dates;  
    }  
  
    /** 
    * 获取该月总天数 
    * @param month 月份 
    * */  
    private static int TotalDays(int month){  
        //小  
        if (isContainMonth(month, months_30_days)){  
            return 30;  
        }  
        //大  
        else if (isContainMonth(month, months_31_days)){  
            return 31;  
        }  
        //特殊  
        else if (month == 1){  
            Calendar calendar = Calendar.getInstance(TimeZone.getTimeZone("GMT+08:00"));  
            int year = calendar.get(Calendar.YEAR);  
            if (year % 4 == 0){//闰  
                return 29;  
            }else { //平  
                return 28;  
            }  
        }  
        //以防万一  
        else {  
            return 30;  
        }  
    }  
  
    /** 
     * 该月属哪个类别 
     * @param arr 大月／小月／特殊月 
     * @param month 要查询的月份 
     * */  
    private static boolean isContainMonth(int month, int[] arr){  
        for (int i = 0; i < arr.length; i++) {  
            if (arr[i] == month)  
                return true;  
        }  
        return false;  
    }  
```

生成的DateObject数组即所得数据集。dateobject为一日的javaben数据

```java
	public int year;  
    public int month;  
    public int day;  
    public String YYMMDD;//年月日字符串  
    public String YYMM;//年月  
    public int position;//在grid中的位置  
    public boolean currentMonth = false;  
    public boolean sellectStatus = false;  
```

保存以上变量，除开年月日position用于记录该日期在grid中的位置，currentmonth为判断该日期是否当月内的日期，sellectstatus保存了该日期的选中状态，控制打卡选中背景的显示方式。具体的打卡逻辑在rc的适配器DateAdapter内。以下是adapter

```java
/** 
 * Created by sung on 2017/7/19. 
 */  
  
public class DateAdapter extends RecyclerView.Adapter<DateAdapter.DateViewHolder> implements View.OnClickListener{  
    public List<DateObject> dates = new ArrayList<>();  
    private onCalendarDayClick onCalendarDayClickListner;  
  
    public DateAdapter(List<DateObject> dates) {  
        if (dates != null)  
            this.dates = dates;  
    }  
  
    @Override  
    public DateViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {  
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.layout_calendar_item, parent, false);  
        DateViewHolder viewHolder = new DateViewHolder(view);  
        return viewHolder;  
    }  
  
    @Override  
    public void onBindViewHolder(DateViewHolder holder, int position) {  
        //此处设置不复用 防止选中状态以及月标状态错乱  
        holder.setIsRecyclable(false);  
  
        holder.position = position;  
        DateObject date = dates.get(position);  
        holder.day.setText(dates.get(position).day+"");  
        holder.root.setOnClickListener(this);  
        holder.root.setTag(holder);  
  
        //设置月初  
        if (date.day == 1) {  
            if (date.currentMonth)  
                holder.month.setText((date.month)+"月");  
            else  
                holder.month.setText((date.month+1)+"月");  
  
            //多重保险  
            if (holder.day.getText().toString().trim().equals("1"))  
                holder.month.setVisibility(View.VISIBLE);  
        }  
  
        //设置当天  
        if (date.year == CalendarView.TODAY.year  
                && date.month == CalendarView.TODAY.month  
                && date.day == CalendarView.TODAY.day){  
            holder.flag.setVisibility(View.VISIBLE);  
        }  
  
        //设置当月高亮  
        if (!date.currentMonth){  
            holder.root.setAlpha(0.3f);  
        }else {  
            holder.day.setTextColor(Color.BLACK);  
        }  
  
        //设置选中状态  
        setSellectStatus(position,holder);  
  
    }  
  
    @Override  
    public int getItemCount() {  
        return dates.size();  
    }  
  
    public void setDates(List dates, boolean reset){  
        if (dates == null)  
            return;  
  
        if (reset){  
            this.dates.clear();  
        }  
        this.dates.addAll(dates);  
        this.notifyDataSetChanged();  
    }  
  
    @Override  
    public void onClick(View view) {  
        if (view instanceof RelativeLayout){  
            DateViewHolder holder = (DateViewHolder) view.getTag();  
            if (!dates.get(holder.position).currentMonth)  
                return;  
  
            dates.get(holder.position).sellectStatus = !dates.get(holder.position).sellectStatus;  
  
            this.notifyDataSetChanged();  
            onCalendarDayClickListner.onClick(holder.position);  
        }  
    }  
  
    /** 
    * 根据前后的选中状态判断图片的状态 
    * @param holder holder操作ui 
    * @param position 游标 
    * */  
    private void setSellectStatus(int position, DateViewHolder holder){  
        DateObject date = dates.get(position);  
        if (!date.currentMonth)  
            return;  
  
        ImageView img = holder.sellect;  
        TextView day = holder.day;  
  
        if (date.sellectStatus){  
            day.setTextColor(Color.WHITE);  
        }else {  
            day.setTextColor(Color.BLACK);  
        }  
  
        if (position == 0){  
            if (date.sellectStatus && dates.get(position + 1).sellectStatus)  
                img.setImageResource(R.drawable.calendar_sellector_left);  
            else if (date.sellectStatus && !dates.get(position + 1).sellectStatus)  
                img.setImageResource(R.drawable.calendar_sellector_single);  
  
            return;  
        }  
  
        if (position == 34){  
            if (date.sellectStatus && dates.get(position - 1).sellectStatus)  
                img.setImageResource(R.drawable.calendar_sellector_right);  
            else if (date.sellectStatus && !dates.get(position - 1).sellectStatus)  
                img.setImageResource(R.drawable.calendar_sellector_single);  
  
            return;  
        }  
  
        DateObject dateLeft = dates.get(position-1);  
        DateObject dateRight = dates.get(position+1);  
  
        if (dateLeft.sellectStatus && date.sellectStatus && dateRight.sellectStatus){  
            img.setImageResource(R.drawable.calendar_sellector_center);  
        }  
  
        if (!dateLeft.sellectStatus && date.sellectStatus && dateRight.sellectStatus){  
            img.setImageResource(R.drawable.calendar_sellector_left);  
        }  
  
        if (dateLeft.sellectStatus && !date.sellectStatus && dateRight.sellectStatus){  
            img.setImageResource(0);  
        }  
  
        if (dateLeft.sellectStatus && date.sellectStatus && !dateRight.sellectStatus){  
            img.setImageResource(R.drawable.calendar_sellector_right);  
        }  
  
        if (!dateLeft.sellectStatus && date.sellectStatus && !dateRight.sellectStatus){  
            img.setImageResource(R.drawable.calendar_sellector_single);  
        }  
    }  
  
    public static class DateViewHolder extends RecyclerView.ViewHolder {  
        public RelativeLayout root;  
        public ImageView sellect;  
        public TextView flag;  
        public TextView day;  
        public TextView month;  
        public int position;  
  
        public DateViewHolder(View view){  
            super(view);  
            position = 0;  
            root = (RelativeLayout) view.findViewById(R.id.calendar_item_root);  
            sellect = (ImageView) view.findViewById(R.id.calendar_item_sellect);  
            flag = (TextView) view.findViewById(R.id.calendar_item_today);  
            day = (TextView) view.findViewById(R.id.calendar_item_text_day);  
            month = (TextView) view.findViewById(R.id.calendar_item_text_month);  
        }  
    }  
  
    public void setOnCalendarDayClickListner(onCalendarDayClick onCalendarDayClickListner){  
        this.onCalendarDayClickListner = onCalendarDayClickListner;  
    }  
  
    public interface onCalendarDayClick{  
        void onClick(int position);  
    }  
}  
```



bindviewholder中主要做初始化ui的操作，控制首月标志/当日标志/月份内区别/选中区别的不同显示。选中的状态操作在方法setsellectstatus中，依次判断 选中日期／上个日期／下个日期 的选中状态，逻辑如下

![20170721153825406](https://github.com/ssccbb/ssccbb.github.io/blob/master/assets/cover/20170721153825406.jpeg?raw=true)

状态保存未实现，只需讲每页的dateobject存入db，用时从db中读而不选择重新生成的方式即可。基本功能逻辑就这些

The End



[项目代码点这](https://github.com/ssccbb/CalendarView)  路过star。感谢