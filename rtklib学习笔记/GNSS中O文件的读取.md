# GNSS中O文件的读取

## 文件命名

`SSSSMRCCC_T_YYYYDDDHHMM_DDU_DDU_DD.FFF`

`SMAR00CHN_R_20232990825.23o`

- 测站标识 SSSSMRCCC = 四个字符的测站名 + 标识编号 + 接收机编号 + ISO-3166-1全球国家名称代码（可参看 Ref.3）
- 数据流种类 T 包括：R 接收机观测数据；S RTCM 实时数据流；U 未知数据
- 数据开始时间 YYYYDDDHHMM = 四个字符的年-三个字符的年积日-两个字符的时-两个字符的分
- 数据持续时间 DDU = 两个字符的数字 + 一个字符的单位
- 数据采样间隔 DDU = 两个字符的数字 + 一个字符的单位
- 数据类型 DD 一般用到的是 MO 表示多系统数据
- 数据存储格式 FFF 一般是 rnx 表示 RINEX 格式的数据

## 文件头

### **头文件部分1：**

只需要看每一行最右侧的注释就可以：

（1）接收机型号：REC # / TYPE / VERS的第二项

（2）天线类型：ANT # / TYPE

（3）近似坐标：APPROX POSITION XYZ

（4）天线高：ANTENNA: DELTA H/E/N的第一项

### 头文件部分2：

`G    8 C1C L1C D1C S1C C5X L5X D5X S5X                      SYS / # / OBS TYPES`
`R    4 C1C L1C D1C S1C                                      SYS / # / OBS TYPES`
`E    8 C1C L1C D1C S1C C5X L5X D5X S5X                      SYS / # / OBS TYPES`
`C    4 C2I L2I D2I S2I                                      SYS / # / OBS TYPES`
`J    8 C1C L1C D1C S1C C5X L5X D5X S5X                      SYS / # / OBS TYPES`

- G：首先是一个字母的系统缩写：

- 8：8种数据类型
- 后面18个三字母的数据类型。数据类型都是由三个字符组成，首先是第一个字符：

`C伪距;L载波;D多普勒;S信号强度`；

第二个字符是数字，代表频数编号；

| 系统 |  数字  |
| :--: | :----: |
| GPS  |  125   |
| GAL  | 15786  |
| GLO  | 14263  |
| BDS  | 215786 |
| SBAS |   15   |
| QZSS |  1256  |

第三个字符表示跟踪模式或通道，比如常用的

`C C/A码;S LxC(D);L LxC(P);X LxC(D+P);P AS off; W AS on;Y Y码;M M码`

![在这里插入图片描述](https://img-blog.csdnimg.cn/94017c3d0c9644949e75297b92d297d9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5rWB5rWq54yq5aS05ouv5pWR5Zyw55CD,size_12,color_FFFFFF,t_70,g_se,x_16)

## 数据块

第一行指示了此历元时间和卫星数目

 `2023 10 26 08 25 20.3680438  0 33`

依次为：年`2023`月`10`日`26`时`08`分`25`秒`20.3680438` + 历元标志`0` + 当前历元所观测到的卫星数`33`。关于历元标志，0表示正常，1表示在前一历元和当前历元之间发生了电源故障，`>1` 表示事件标志。

之后的数据按照头文件`SYS / # / OBS TYPES`的顺序，一共有`33`行

`G03  23389310.817           0.000        -120.201          27.000    23389331.803           0.000         -90.016          37.000`

`G    8 C1C L1C D1C S1C C5X L5X D5X S5X                      SYS / # / OBS TYPES`

第一个值为伪距观测值，类型标识为`C1C`

第二个值为相位观测值，类型标识为`L1C`

第三个值为多普勒观测值，类型标识为`D1C`

第四个值为载噪比，类型标识为`S1C`

后面四个是相同的道理，只不过波段号不同，通道不同

## 观测值

![image-20231029172108664](C:\Users\FBM\AppData\Roaming\Typora\typora-user-images\image-20231029172108664.png)

每一个观测类型的组成包括：观测值 + LLI + 信号强度

以图片中为例

第一个伪距观测值`22585468.500 7`

值为`22585468.500m`, LLI 位为空，信号强度`SSI=7`

### 失锁标识 LLI

其中 LLI（Loss of Lock Indicator）表示失锁标识符，它的范围为0~7，0或空格表示跟踪连续正常或者状态未知；bit 0置1表示在前一历元与当前历元之间发生了失锁,可能有周跳；bit 1置1表示可能发生了半周跳（半周跳是什么东西？），如果软件不能处理半周数据，则应跳过此历元；bit 2置1表示为反欺骗(AS)下的观测值(可能会受到噪声增加的影响)。其中, bit 0和bit 1仅用于相位。

LLI的范围是`0~7`，化成2进制就是`000~111`

### 信号强度 SSI

信号强度（Signal Strength Indicator，SSI）在RINEX格式中，用`1~9`表示信号强度，1表示可能的最小信号强度，5表示良好S/N比的阈值，9表示可能的最大信导强度，0或空表示未知或未给出。

## 参考

https://blog.csdn.net/Gou_Hailong/article/details/120911467