# RTKLIB时间系统函数

```cpp
// 字符串转换为数字
double str2num(const char *s, int i, int n);

// 字符串转换为时间
int str2time(const char *s, int i, int n, gtime_t *t);

// 时间转换为字符串
void time2str(gtime_t t, char *str, int n);

// epoch时间转换为时间
gtime_t epoch2time(const double *ep);

// 时间转换为epoch时间
void time2epoch(gtime_t t, double *ep);

// GPS周和秒转换为时间
gtime_t gpst2time(int week, double sec);

// 时间转换为GPS周和秒
double time2gpst(gtime_t t, int *week);

// GST周和秒转换为时间
gtime_t gst2time(int week, double sec);

// 时间转换为GST周和秒
double time2gst(gtime_t t, int *week);

// 北斗周和秒转换为时间
gtime_t bdt2time(int week, double sec);

// 时间转换为北斗周和秒
double time2bdt(gtime_t t, int *week);

// 将时间转换为指定格式的字符串
char *time_str(gtime_t t, int n);

// 时间加法
gtime_t timeadd(gtime_t t, double sec);

// 时间差
double timediff(gtime_t t1, gtime_t t2);

// GPS时间转换为UTC时间
gtime_t gpst2utc(gtime_t t);

// UTC时间转换为GPS时间
gtime_t utc2gpst(gtime_t t);

// GPS时间转换为北斗时间
gtime_t gpst2bdt(gtime_t t);

// 北斗时间转换为GPS时间
gtime_t bdt2gpst(gtime_t t);

// 获取当前时间
gtime_t timeget(void);

// 设置时间
void timeset(gtime_t t);

// 重置时间
void timereset(void);

// 时间转换为一年中的天数
double time2doy(gtime_t t);

// UTC时间转换为格林尼治平太阳时
double utc2gmst(gtime_t t, double ut1_utc);

// 读取闰秒表
int read_leaps(const char *file);

// 调整GPS周数（适用于大于等于1024周的GPS周数）
int adjgpsweek(int week);

// 获取系统运行时间（以毫秒为单位）
uint32_t tickget(void);

// 睡眠指定毫秒数
void sleepms(int ms);
```

### 1.RTKLIB时间的表示、转换、处理

##### `timeget`：从系统时间获取当前时间。

GNSS数据处理对时间精度非常敏感，如果用double类型的TOW，相当于距离上只有0.004m的精度，。出于精度的考虑，RTKLIB将时间表示为gtime_t结构体，用time_t类型表示1970年以来的整秒数，有double类型表示不到1s的时间，time_t随机器而定一般是是无符号__int64 类型，由于整数长度限制，gtime_t不能表示1970年以前和2038以后的时间。

```cpp
typedef struct {        /* time struct */
    time_t time;        /* time (s) expressed by standard time_t */
    double sec;         /* fraction of second under 1 s */
} gtime_t;
```

##### `time2epoch`：将时间值（以秒为单位）转换为GPS周数和周内秒（GPS时间）

该函数的输入是一个`gtime_t`类型的时间`t`，以及一个指向具有6个元素的double类型数组`ep`的指针。函数的输出是将转换后的epoch时间存储在`ep`数组中。

儒略日（连续数字表示）Julian Day至格里高利转换方法：

```cpp
a= floor(julday+0.5);
b=a+1537;
c = floor((b-122.1)/365.25);
d = floor(365.25*c);
e = floor((b-d)/30.6001);
D = floor(b-d- floor(30.6001*e)+ rem(julday+.5,1))； 	%天
H=(rem(julday+.5,1))*24;  								%时
M=e-1-12*floor(e/14); 									%月
Y=c-4715-floor((7+M)/10); 								%年
```

Rem 函数是取余函数 rem(x,y)=x-y*fix(x/y),其中fix()是向0取整，例如fix(-1.5)=-1, fix(1.5)=1。
注：上面的转换算法仅在1900年3月1日至2100年2月28日期间有效。

```cpp
extern void time2epoch(gtime_t t, double *ep)
{
    const int mday[]={ /* # of days in a month */
        31,28,31,30,31,30,31,31,30,31,30,31,31,28,31,30,31,30,31,31,30,31,30,31,
        31,29,31,30,31,30,31,31,30,31,30,31,31,28,31,30,31,30,31,31,30,31,30,31
    };
    int days,sec,mon,day;
    
    /* leap year if year%4==0 in 1901-2099 */
    days=(int)(t.time/86400);
    sec=(int)(t.time-(time_t)days*86400);
    for (day=days%1461,mon=0;mon<48;mon++) {
        if (day>=mday[mon]) day-=mday[mon]; else break;
    }
    ep[0]=1970+days/1461*4+mon/12; ep[1]=mon%12+1; ep[2]=day+1;
    ep[3]=sec/3600; ep[4]=sec%3600/60; ep[5]=sec%60+t.sec;
}
```

1. `const int mday[]={...};` - 定义了一个数组`mday[]`表示每个月的天数，包括闰年的情况。
2. `int days,sec,mon,day;` - 定义了几个整数类型的变量，用于存储天数、秒数和月份等计算结果。
3. `days=(int)(t.time/86400);` - 将时间`t`转换为天数，每天的秒数为86400。
4. `sec=(int)(t.time-(time_t)days*86400);` - 计算剩余的秒数，即将总秒数减去整天的秒数。
5. `for (day=days%1461,mon=0;mon<48;mon++) {...}` - 使用循环来确定当前月份和日期。
6. `if (day>=mday[mon]) day-=mday[mon];` - 判断是否超过当前月份的天数，并进行相应的天数调整。
7. `ep[0]=1970+days/1461*4+mon/12;` - 计算年份，其中1461天是闰年的周期。
8. `ep[1]=mon%12+1;` - 计算月份，使用取余运算符获取当前月份，加1以将月份从0开始转换为从1开始。
9. `ep[2]=day+1;` - 计算日期，加1以将日期从0开始转换为从1开始。
10. `ep[3]=sec/3600;` - 计算小时，将剩余秒数除以3600获取小时数。
11. `ep[4]=sec%3600/60;` - 计算分钟，通过计算剩余秒数除以3600的余数，再除以60，获取分钟数。
12. `ep[5]=sec%60+t.sec;` - 计算秒数，通过计算剩余秒数除以60的余数，再加上时间`t`中的秒数`t.sec`。

##### `epoch2time`：将GPS周数和周内秒转换为时间值（以秒为单位）。

```cpp
extern gtime_t epoch2time(const double *ep)
{
    const int doy[]={1,32,60,91,121,152,182,213,244,274,305,335};   //每月第一天的doy
    gtime_t time={0};
    int days,sec,year=(int)ep[0],mon=(int)ep[1],day=(int)ep[2];
    
    if (year<1970||2099<year||mon<1||12<mon) return time;
    
    /* leap year if year%4==0 in 1901-2099 */
    days=(year-1970)*365+(year-1969)/4+doy[mon-1]+day-2+(year%4==0&&mon>=3?1:0);
    sec=(int)floor(ep[5]);
    time.time=(time_t)days*86400+(int)ep[3]*3600+(int)ep[4]*60+sec;
    time.sec=ep[5]-sec;
    return time;
}
```

1. `const int doy[]={...};` - 定义了一个数组`doy[]`，表示每个月的第一天对应的年内天数。
2. `gtime_t time={0};` - 定义了一个`gtime_t`类型的时间结构体`time`，并将其初始值设置为0。
3. `int days,sec,year=(int)ep[0],mon=(int)ep[1],day=(int)ep[2];` - 定义了几个整数类型的变量，分别表示总天数、秒数、年份、月份和日期。
4. `if (year<1970||2099<year||mon<1||12<mon) return time;` - 判断年份和月份，如果不满足条件，则直接返回初始的时间结构体。
5. `days=(year-1970)*365+(year-1969)/4+doy[mon-1]+day-2+(year%4==0&&mon>=3?1:0);` - 计算从1970年1月1日到给定日期的总天数，考虑了闰年的影响。
6. `sec=(int)floor(ep[5]);` - 将秒数向下取整，得到整数部分。
7. `time.time=(time_t)days*86400+(int)ep[3]*3600+(int)ep[4]*60+sec;` - 计算总秒数，包括日期的秒数、小时的秒数和分钟的秒数。
8. `time.sec=ep[5]-sec;` - 计算秒数的小数部分。
9. `return time;` - 返回时间结构体`time`。

##### `str2num`：字符串转数字

RTKLIB中的字符转换数字的函数str2num中有一个特殊的地方，对字符多进行了一步判断，判断其是否为字符’d’或’D’,若符合则将其转换为’E’，其余则不变，进行这一步的目的在于对采用科学计数法表示的数字的判读。C语言中用E表示的科学计数法数字可以识别，而D不能，因此在此处进行转换。

```cpp
extern double str2num(const char *s, int i, int n)
{
    double value;
    char str[256],*p=str;
    
    if (i<0||(int)strlen(s)<i||(int)sizeof(str)-1<n) return 0.0;
    for (s+=i;*s&&--n>=0;s++) *p++=*s=='d'||*s=='D'?'E':*s;
    *p='\0';
    return sscanf(str,"%lf",&value)==1?value:0.0;
}
```

1. `double value;` - 存储转换后的数值。
2. `char str[256],*p=str;` - 定义一个长度为256的字符串数组`str`和一个指向字符串数组的指针`p`。
3. `if (i<0||(int)strlen(s)<i||(int)sizeof(str)-1<n) return 0.0;` - 检查输入参数是否合法，包括起始位置`i`必须大于等于0，输入字符串`s`的长度必须大于等于起始位置`i`，字符串数组`str`剩余空间必须大于等于需要复制的字符个数`n`。
4. `for (s+=i;*s&&--n>=0;s++) *p++=*s=='d'||*s=='D'?'E':*s;` - 遍历字符串中指定位置之后的字符，将字符复制到字符串数组`str`中。如果字符是字母'd'或'D'，则将其转换为大写字母'E'，用于表示指数。
5. `*p='\0';` - 在字符串数组`str`的最后一个字符后面加上结束符。
6. `return sscanf(str,"%lf",&value)==1?value:0.0;` - 使用`sscanf`函数将字符串数组`str`中的内容转换为double型数值，并将转换结果存储在变量`value`中。如果转换成功，则返回转换后的数值；否则返回0.0

##### `time2str`：将时间值转换为字符串格式（如"yyyy/mm/dd hh:mm:ss.sss"），以传入的字符串指针s返回。

```cpp
extern void time2str(gtime_t t, char *s, int n)
{
    double ep[6];
    if (n<0) n=0; else if (n>12) n=12;
    if (1.0-t.sec<0.5/pow(10.0,n)) {t.time++; t.sec=0.0;};
    time2epoch(t,ep);	//先把时间转为epoch时间数组
    sprintf(s,"%04.0f/%02.0f/%02.0f %02.0f:%02.0f:%0*.*f",ep[0],ep[1],ep[2],
            ep[3],ep[4],n<=0?2:n+3,n<=0?0:n,ep[5]);
}
```

1. `double ep[6];` - `ep`数组表示时间，是一个包含年、月、日、小时、分钟和秒的`double`类型数组。
2. `if (n<0) n=0; else if (n>12) n=12;` - 对输入的参数`n`进行限制，如果`n`小于0，则将`n`设为0；如果`n`大于12，则将`n`设为12。确保`n`在0到12之间。
3. `if (1.0-t.sec<0.5/pow(10.0,n)) {t.time++; t.sec=0.0;};` - 判断秒数的小数部分是否大于等于0.5/pow(10,n)。如果是，则将时间t向上进位1秒，并将秒数的小数部分设置为0。
4. `time2epoch(t,ep);` - 将时间t转换为epoch时间数组，存储到ep数组中。
5. `sprintf(s,"%04.0f/%02.0f/%02.0f %02.0f:%02.0f:%0*.*f",ep[0],ep[1],ep[2],ep[3],ep[4],n<=0?2:n+3,n<=0?0:n,ep[5]);` - 使用sprintf函数将ep数组中的时间格式化为字符串，并将结果保存在字符串数组s中。其中，格式化字符串"%04.0f/%02.0f/%02.0f %02.0f:%02.0f:%0*.*f"表示按照指定的格式输出年、月、日、小时、分钟和秒。

##### `str2time`：将字符串格式的时间转换为时间值。

```cpp
extern int str2time(const char *s, int i, int n, gtime_t *t)
{
    double ep[6];   //ep数组表示时间：double类型数组，存年月日时分秒
    char str[256],*p=str;
    
    if (i<0||(int)strlen(s)<i||(int)sizeof(str)-1<i) return -1;
    for (s+=i;*s&&--n>=0;) *p++=*s++;
    *p='\0';
    if (sscanf(str,"%lf %lf %lf %lf %lf %lf",ep,ep+1,ep+2,ep+3,ep+4,ep+5)<6)
        return -1;  //sscanf从字符串中读取年月日时分秒，先存到ep数组中
    if (ep[0]<100.0) ep[0]+=ep[0]<80.0?2000.0:1900.0;
    *t=epoch2time(ep);  //再把ep数组转换成gtime_t类型
    return 0;
}
```

1. `double ep[6];` - `ep`数组表示时间，是一个包含年、月、日、小时、分钟和秒的`double`类型数组。
2. `char str[256],*p=str;` - 定义一个长度为256的字符串数组`str`和一个指向字符串数组的指针`p`。
3. `if (i<0||(int)strlen(s)<i||(int)sizeof(str)-1<i) return -1;` - 检查输入参数是否合法。其中，起始位置`i`必须大于等于0，输入字符串`s`的长度必须大于等于起始位置`i`，字符串数组`str`的剩余空间必须大于等于`i`。
4. `for (s+=i;*s&&--n>=0;) *p++=*s++;` - 遍历字符串中指定位置开始的字符，将字符复制到字符串数组`str`中。
5. `*p='\0';` - 在字符串数组`str`的最后一个字符后面加上结束符。
6. `if (sscanf(str,"%lf %lf %lf %lf %lf %lf",ep,ep+1,ep+2,ep+3,ep+4,ep+5)<6) return -1;` - 使用`sscanf`函数从字符串数组`str`中读取年、月、日、小时、分钟和秒，并将读取的数据存储到`ep`数组中。如果成功读取的数据不满足6个，则表示转换失败，返回-1。
7. `if (ep[0]<100.0) ep[0]+=ep[0]<80.0?2000.0:1900.0;` - 判断年份是否小于100，如果小于100，则根据年份的范围进行修正，加上2000或1900，以确定年份。
8. `*t=epoch2time(ep);` - 将`ep`数组转换为`gtime_t`类型的时间，并将结果存储在`t`指针指向的变量中。
9. `return 0;` - 返回0表示转换成功。

### 2.**gtime_t**、**GPST**、**UCT**、**BDT**、**GST**间的转换函数

##### `gpst2time ()`：GPST转gtime_t

​	**input**：int week：GPS周数		double sec ：GPS秒数

​	**output**：t ：gtime_t格式秒

```cpp
extern gtime_t gpst2time(int week, double sec)
{
    gtime_t t=epoch2time(gpst0);	
    //获取GPS时间的起始时间，存储在t中“static const double gpst0[]={1980,1, 6,0,0,0}; /* gps time reference */”   

    if (sec<-1E9||1E9<sec) sec=0.0;	//秒数小于-10^9或大于10^9，将秒数设为0，处理异常情况
    t.time+=(time_t)86400*7*week+(int)sec;	//总秒数=周数×7天/周×86400秒/天+输入秒数的整数部分
    t.sec=sec-(int)sec;	//秒数的小数部分
    return t;

}
```

##### **time2gpst ()**：gtime_t转GPST。

​	**input**：  gtime_t格式秒

​	**output**：int week：GPS周数		double sec ：GPS秒数

​	**int  *week**：定义了一个指向整数的指针week，存储GPS时间的周数。

​				既可用作输入，也可用作输出。

​				可以将一个周数的值传递给函数（作为输入），使用它进行计算，若传递了一个非空指针，函数会将计算后的周数值				存储在指向的位置（作为输出），以便在函数外部访问这个值。

​				如果不关心函数计算的周数，可将它设置为 `NULL`，不需要在函数外部获取周数的值。

```cpp
extern double time2gpst(gtime_t t, int *week)
{
    gtime_t t0=epoch2time(gpst0);
    //获取GPS时间的起始时间，存储在t中“static const double gpst0[]={1980,1, 6,0,0,0}; /* gps time reference */”
    time_t sec=t.time-t0.time;	//计算总秒数
    int w=(int)(sec/(86400*7));	//整除，计算GPS周数
    
    if (week) *week=w;
    //检查是否提供了一个int类型的指针week。如果有，将GPS周数w存储在week指针指向的位置，以便返回给调用函数
    return (double)(sec-(double)w*86400*7)+t.sec;	//GPS秒=总秒数-整数周对应的秒数+输入秒数的小数部分

}
```



**timeadd()**：给传入的gtime_t增加秒数

```cpp
/* add time --------------------------------------------------------------------
* add time to gtime_t struct                    //更新时间，增加秒数
* args   : gtime_t t        I   gtime_t struct
*          double sec       I   time to add (s)
* return : gtime_t struct (t+sec)
*-----------------------------------------------------------------------------*/
extern gtime_t timeadd(gtime_t t, double sec)
{
    double tt;
    
    t.sec+=sec; //t.sec=sec+t.sec, 给定时间秒的小数部分+要增加的秒数
    tt=floor(t.sec); //取整（小于t.sec）
    t.time+=(int)tt;//t.time=t.time+(int)tt，增大秒数的整数部分
    t.sec-=tt;//t.sec=t.sec-tt，更新秒的小数部分
    return t;
}
```

**timediff ()**：求时间差 **t1 - t2**

```cpp
/* time difference -------------------------------------------------------------
* difference between gtime_t structs
* args   : gtime_t t1,t2    I   gtime_t structs
* return : time difference (t1-t2) (s)
*-----------------------------------------------------------------------------*/
extern double timediff(gtime_t t1, gtime_t t2)
{
    return difftime(t1.time,t2.time)+t1.sec-t2.sec;
    //使用C标准库函数 difftime 来计算两个 gtime_t 结构中的整数时间字段 time 之间的差异
    //再加上小数部分
}
```

**rtkcmn.c中跳秒的定义**

```cpp
static double leaps[MAXLEAPS+1][7]={ /* (y,m,d,h,m,s,utc-gpst) */
    {2017,1,1,0,0,0,-18},
    {2015,7,1,0,0,0,-17},
    {2012,7,1,0,0,0,-16},
    {2009,1,1,0,0,0,-15},
    {2006,1,1,0,0,0,-14},
    {1999,1,1,0,0,0,-13},
    {1997,7,1,0,0,0,-12},
    {1996,1,1,0,0,0,-11},
    {1994,7,1,0,0,0,-10},
    {1993,7,1,0,0,0, -9},
    {1992,7,1,0,0,0, -8},
    {1991,1,1,0,0,0, -7},
    {1990,1,1,0,0,0, -6},
    {1988,1,1,0,0,0, -5},
    {1985,7,1,0,0,0, -4},
    {1983,7,1,0,0,0, -3},
    {1982,7,1,0,0,0, -2},
    {1981,7,1,0,0,0, -1},
    {0}
};
```

##### `utc2gpst`：考虑闰秒，将UTC（协调世界时）时间转换为GPS时间。

```cpp
/* utc to gpstime --------------------------------------------------------------
* convert utc to gpstime considering leap seconds
* args   : gtime_t t        I   time expressed in utc
* return : time expressed in gpstime
* notes  : ignore slight time offset under 100 ns
*-----------------------------------------------------------------------------*/
extern gtime_t utc2gpst(gtime_t t)
{
    int i;
    
    for (i=0;leaps[i][0]>0;i++) //遍历闰秒表（数组 leaps），判断条件leaps[i][0]>0用于判断是否遍历到了闰秒表的末尾
    {
        if (timediff(t,epoch2time(leaps[i]))>=0.0) //输入的UTC时间晚于或等于闰秒的发生时间，则找到了最近的适用闰秒
            return timeadd(t,-leaps[i][6]);//将输入的UTC时间与该闰秒的时间偏差进行减法运算UTC-(UTC-GPST),并返回
    }
    return t;//直接返回输入的gpstime，表示无需进行转换。
}
```

##### `gpst2utc`：考虑闰秒，将GPS时间转换为UTC时间。

```cpp
/* gpstime to utc --------------------------------------------------------------
* convert gpstime to utc considering leap seconds
* args   : gtime_t t        I   time expressed in gpstime
* return : time expressed in utc
* notes  : ignore slight time offset under 100 ns
*-----------------------------------------------------------------------------*/
extern gtime_t gpst2utc(gtime_t t)
{
    gtime_t tu;//存储转换后的UTC时间
    int i;
    
    for (i=0;leaps[i][0]>0;i++) //遍历闰秒表（数组 leaps）,leaps[i][0]>0用于判断是否遍历到了闰秒表的末尾
    {
        tu=timeadd(t,leaps[i][6]);//将输入的gpstime与闰秒的时间偏差进行加法运算GPST+(UTC-GPST),赋值给tu
        if (timediff(tu,epoch2time(leaps[i]))>=0.0) //转换后的UTC时间晚于或等于闰秒的发生时间
            return tu;
    }
    return t;//直接返回输入的gpstime，表示无需进行转换
}
```

**`gpst2bdt`：将GPS时间转换为BD时间**

```cpp
/* gpstime to bdt --------------------------------------------------------------
* convert gpstime to bdt (beidou navigation satellite system time)
* args   : gtime_t t        I   time expressed in gpstime
* return : time expressed in bdt
* notes  : ref [8] 3.3, 2006/1/1 00:00 BDT = 2006/1/1 00:00 UTC
*          no leap seconds in BDT
*          ignore slight time offset under 100 ns
*-----------------------------------------------------------------------------*/
extern gtime_t gpst2bdt(gtime_t t)
{
    return timeadd(t,-14.0);
}
```

##### **`bdt2gpst`：将BD时间转换为GPS时间**

```cpp
/* bdt to gpstime --------------------------------------------------------------
* convert bdt (beidou navigation satellite system time) to gpstime
* args   : gtime_t t        I   time expressed in bdt
* return : time expressed in gpstime
* notes  : see gpst2bdt()
*-----------------------------------------------------------------------------*/
extern gtime_t bdt2gpst(gtime_t t)
{
    return timeadd(t,14.0);
}
```

**time2doy ()：gtime_t转年积日DOY**

```cpp
extern double time2doy(gtime_t t)
{
    double ep[6];//定义数组存储时间t的各个分量
    
    time2epoch(t,ep);//将时间t转换为ep数组表示的时间格式
    ep[1]=ep[2]=1.0; //ep数组的月和日设置为1
    ep[3]=ep[4]=ep[5]=0.0;//时、分、秒设置为0，得当月1日零点
    return timediff(t,epoch2time(ep))/86400.0+1.0;
    //计算时间t与调整后的时间之间的差值，除以86400.0（一天的秒数），再加上1.0，即可得到年积日
}
```

国际原子时（Temps Atomique International, TAI）指以世界时 1958 年 1 月 1 日 0 时为起点，以铯原子基态两级间跃迁辐射的 9192631770 周所需的时间作为 1 秒秒长定义的一 种连续均匀的时间系统。TT 与 TAI 的转换关系
**TT=TAI+32.184**

协调世界时（UTC）也属于 TAI，但是 UTC 通过跳秒保持与 UT1 在时刻上相近（差异小于 0.9s），从而有了实际的物理意义。跳秒的发生时间通常在一年的 6 月 30 日或 12 月 31 日最后一分钟变为 61s 或 59s。截止 2020 年 1 月，UTC 和 TAI 之间总的跳秒数为 37s， 近期的一次跳秒在 2016 年 12 月 31 日。UTC 与 TAI 的转换公式为
![image-20231015162619305](C:\Users\FBM\AppData\Roaming\Typora\typora-user-images\image-20231015162619305.png)

leap_second为跳秒，跳秒（leap second）是国际协调时间（UTC）与国际原子时（TAI）之间的时间调整。由于地球自转速度的微小变化，使得地球平均日长（UT1）与国际原子时之间存在不稳定的差异。为了维持UTC和地球自转的大致同步，国际地球自转与参考系统服务（IERS）会根据地球自转的观测数据，不定期地在UTC中插入或删除一秒。

当跳秒发生时，时间会调整到下一秒（或前一秒），在调整时刻，持续时间为59秒（或61秒）。这个调整通常在UTC的最后一天（或第一个月）的最后一秒进行。跳秒的目的是确保整个世界使用的时间系统与日地运动保持同步。

需要注意的是，跳秒只适用于UTC，而不适用于其他时间表示方式，如国际原子时（TAI）、GPS时间（GPST）等。因此，在进行时间计算和转换时，需要考虑跳秒对时间的影响，特别是涉及到UTC的时候。![image-20231015162639880](C:\Users\FBM\AppData\Roaming\Typora\typora-user-images\image-20231015162639880.png)

##### 

##### 



##### 

