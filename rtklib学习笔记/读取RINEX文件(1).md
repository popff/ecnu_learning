```cpp
#define NAVEXP      "D"                 /* exponent letter in RINEX NAV */
#define NUMSYS      7                   /* number of systems */	//七种卫星系统
#define MAXRNXLEN   (16*MAXOBSTYPE+4)   /* max RINEX record length */	//最大记录长度
#define MAXPOSHEAD  1024                /* max head line position */	//最大头文件长度
#define MINFREQ_GLO -7                  /* min frequency number GLONASS */
#define MAXFREQ_GLO 13                  /* max frequency number GLONASS */
#define NINCOBS     262144              /* incremental number of obs data */	//2的18次方
```

```cpp
static const int navsys[]={             /* satellite systems */
    SYS_GPS,SYS_GLO,SYS_GAL,SYS_QZS,SYS_SBS,SYS_CMP,SYS_IRN,0
};

//G：GPS、R：GLONASS、E：GALILEO、J：QZSS、S：SBAS、C：BDS、I：IRNSS
static const char syscodes[]="GREJSCI"; /* satellite system codes */

//C：伪距（米）、D：多普勒（Hz）、L：载波相位（整周）、S：载噪比
static const char obscodes[]="CLDS";    /* observation type codes */
```

##### sigind_t：表示每种卫星系统的载波类型和观测值类型 

每种类型的系统对应一个sigind_t结构体，只需要建立七个结构体

```cpp
/* type definition -----------------------------------------------------------*/
typedef struct {                        /* signal index type */
    int n;                              /* number of index */
    									//n表示该卫星系统总的观测值类型数，对应卫星系统标识符后面的数字
    int idx[MAXOBSTYPE];                /* signal freq-index */
    int pos[MAXOBSTYPE];                /* signal index in obs data (-1:no) */
    uint8_t pri [MAXOBSTYPE];           /* signal priority (15-0) */
    uint8_t type[MAXOBSTYPE];           /* type (0:C,1:L,2:D,3:S) */
    uint8_t code[MAXOBSTYPE];           /* obs-code (CODE_L??) */
    double shift[MAXOBSTYPE];           /* phase shift (cycle) */
} sigind_t;
```

##### **obsd_t**：用来存储某个历元中的某个卫星的观测值

```cpp
typedef struct {        /* observation data record */
    gtime_t time;       /* receiver sampling time (GPST) */
    uint8_t sat,rcv;    /* satellite/receiver number */
    uint16_t SNR[NFREQ+NEXOBS]; /* signal strength (0.001 dBHz) */	//信噪比
    uint8_t  LLI[NFREQ+NEXOBS]; /* loss of lock indicator */	//周跳（由于卫星信号失锁而导致整周计数的跳变或中断）
    uint8_t code[NFREQ+NEXOBS]; /* code indicator (CODE_???) */
    double L[NFREQ+NEXOBS]; /* observation data carrier-phase (cycle) */	//载波相位
    double P[NFREQ+NEXOBS]; /* observation data pseudorange (m) */	//伪距
    float  D[NFREQ+NEXOBS]; /* observation data doppler frequency (Hz) */	//多普勒频移
} obsd_t;
```

##### **obs_t** ：存一系列的obsd_t

```cpp
typedef struct {        /* observation data */
    int n,nmax;         /* number of obervation data/allocated */
    					//n：存放obsd_t的数目；nmax：目前data内存空间最大能存的obsd_t数目
    obsd_t *data;       /* observation data records */
} obs_t;
```

#### readrnxh()：读取文件头

```cpp
/* read RINEX file header ----------------------------------------------------*/
static int readrnxh(FILE *fp, double *ver, char *type, int *sys, int *tsys,
                    char tobs[][MAXOBSTYPE][4], nav_t *nav, sta_t *sta)
{
    char buff[MAXRNXLEN],*label=buff+60;
    int i=0;
    
    trace(3,"readrnxh:\n");
    
    *ver=2.10; *type=' '; *sys=SYS_GPS;
    
    while (fgets(buff,MAXRNXLEN,fp)) {
        
        if (strlen(buff)<=60) {		//判定观测文件头文件部分所有字符总长度是否正常
            continue;				//小于等于60则继续
        }
        else if (strstr(label,"RINEX VERSION / TYPE")) {	//如果包含字符串"RINEX VERSION / TYPE"
            *ver=str2num(buff,0,9);		//记录版本号（提取buff 0到9之间的字符转换为数字）
            *type=*(buff+20);			//记录观测文件类型（从buff第20个字符开始提取）
            
            /* satellite system */	//判断卫星系统类型
            switch (*(buff+40)) {	//从buff第20个字符开始提取
                case ' ':
                case 'G': *sys=SYS_GPS;  *tsys=TSYS_GPS; break;
                case 'R': *sys=SYS_GLO;  *tsys=TSYS_UTC; break;
                case 'E': *sys=SYS_GAL;  *tsys=TSYS_GAL; break; /* v.2.12 */
                case 'S': *sys=SYS_SBS;  *tsys=TSYS_GPS; break;
                case 'J': *sys=SYS_QZS;  *tsys=TSYS_QZS; break; /* v.3.02 */
                case 'C': *sys=SYS_CMP;  *tsys=TSYS_CMP; break; /* v.2.12 */
                case 'I': *sys=SYS_IRN;  *tsys=TSYS_IRN; break; /* v.3.03 */
                case 'M': *sys=SYS_NONE; *tsys=TSYS_GPS; break; /* mixed */
                default :
                    trace(2,"not supported satellite system: %c\n",*(buff+40));
                    break;
            }
            continue;
        }
        else if (strstr(label,"PGM / RUN BY / DATE")) {	//如果包含字符串"PGM / RUN BY / DATE"
            continue;
        }
        else if (strstr(label,"COMMENT")) {	//如果包含字符串"COMMENT"
            continue;
        }
        switch (*type) { /* file type */	//判断文件类型分配不同函数读取文件头
            case 'O': decode_obsh(fp,buff,*ver,tsys,tobs,nav,sta); break;
            case 'N': decode_navh (buff,nav); break;
            case 'G': decode_gnavh(buff,nav); break;
            case 'H': decode_hnavh(buff,nav); break;
            case 'J': decode_navh (buff,nav); break; /* extension */
            case 'L': decode_navh (buff,nav); break; /* extension */
        }
        if (strstr(label,"END OF HEADER")) return 1;	//检索到"END OF HEADER"，返回1，头部信息解析完成
        
        if (++i>=MAXPOSHEAD&&*type==' ') break; /* no RINEX file */	
        										//i值达到最大上限，且文件类型为空，跳出循环，返回0，说明无有效RINEX文件
    }
    return 0;
}
```

#### readrnxfp()：根据文件类型调用对应的读取函数

```cpp
/* read RINEX file -----------------------------------------------------------*/
static int readrnxfp(FILE *fp, gtime_t ts, gtime_t te, double tint,
                     const char *opt, int flag, int index, char *type,
                     obs_t *obs, nav_t *nav, sta_t *sta)
{
    double ver;
    int sys,tsys=TSYS_GPS;
    char tobs[NUMSYS][MAXOBSTYPE][4]={{""}};
    
    trace(3,"readrnxfp: flag=%d index=%d\n",flag,index);
    
    /* read RINEX file header */
    //调用readrnxh函数读取文件头，获取观测文件类型type
    if (!readrnxh(fp,&ver,type,&sys,&tsys,tobs,nav,sta)) return 0;	//若读取失败，返回0
    
    /* flag=0:except for clock,1:clock */
    if ((!flag&&*type=='C')||(flag&&*type!='C')) return 0;	//若flag=0且type=C 或 flag=1且type≠C，矛盾，返回0
    
    /* read RINEX file body */	//读取观测文件体
    switch (*type) {	//判断type分别调用不同函数读取
        case 'O': return readrnxobs(fp,ts,te,tint,opt,index,ver,&tsys,tobs,obs,
                                    sta);
        case 'N': return readrnxnav(fp,opt,ver,sys    ,nav);
        case 'G': return readrnxnav(fp,opt,ver,SYS_GLO,nav);
        case 'H': return readrnxnav(fp,opt,ver,SYS_SBS,nav);
        case 'J': return readrnxnav(fp,opt,ver,SYS_QZS,nav); /* extension */
        case 'L': return readrnxnav(fp,opt,ver,SYS_GAL,nav); /* extension */
        case 'C': return readrnxclk(fp,opt,index,nav);
    }
    trace(2,"unsupported rinex type ver=%.2f type=%c\n",ver,*type);
    return 0;
}
```

### 读取OBS观测文件

#### **decode_obsh()：**解析观测数据文件头

```cpp
/* decode RINEX observation data file header ---------------------------------*/
static void decode_obsh(FILE *fp, char *buff, double ver, int *tsys,
                        char tobs[][MAXOBSTYPE][4], nav_t *nav, sta_t *sta)
{
    /* default codes for unknown code */
    const char frqcodes[]="1256789";    //定义频率数组
    const char *defcodes[]={
        "CWX    ",  /* GPS: L125____ */
        "CCXX X ",  /* GLO: L1234_6_ */
        "C XXXX ",  /* GAL: L1_5678_ */
        "CXXX   ",  /* QZS: L1256___ */
        "C X    ",  /* SBS: L1_5____ */
        "XIXIIX ",  /* BDS: L125678_ */
        "  A   A"   /* IRN: L__5___9 */
    };
    double del[3];
    int i,j,k,n,nt,prn,fcn;
    const char *p;
    char *label=buff+60,str[4]; //读文件头标签，从60开始
    
    trace(4,"decode_obsh: ver=%.2f\n",ver);
    
    if      (strstr(label,"MARKER NAME"         )) {	//如果包含字符串"MARKER NAME"
        if (sta) setstr(sta->name,buff,60); 	//若sta存在，将buff的信息记录为测站名，最多60个字符
    }
    else if (strstr(label,"MARKER NUMBER"       )) { /* opt */	//可选）如果包含字符串"MARKER NUMBER"
        if (sta) setstr(sta->marker,buff,20);	//记录测站编号，最多20个字符长度
    }
    else if (strstr(label,"MARKER TYPE"         )) ; /* ver.3 */ //版本3的信息，包含字符串"MARKER TYPE"，空处理
    else if (strstr(label,"OBSERVER / AGENCY"   )) ;	//包含字符串"OBSERVER / AGENCY"，空处理
    else if (strstr(label,"REC # / TYPE / VERS" )) {	//若包含字符串"REC # / TYPE / VERS"
        if (sta) {  //在文件中各个信息占20个字符，因此每20个字符赋值一次，最大长度为20
            setstr(sta->recsno, buff,   20);	//记录编号
            setstr(sta->rectype,buff+20,20);	//记录类型
            setstr(sta->recver, buff+40,20);	//记录版本
        }
    }
    else if (strstr(label,"ANT # / TYPE"        )) {	//若包含字符串"REC # / TYPE / VERS"
        if (sta) {	//每20个字符赋值一次，最大长度为20
            setstr(sta->antsno,buff   ,20);	//天线编号
            setstr(sta->antdes,buff+20,20);	//天线类型
        }
    }
    else if (strstr(label,"APPROX POSITION XYZ" )) {	//若包含字符串"APPROX POSITION XYZ"
        if (sta) {  //循环语句，使得每14字符赋值一次
            		//将测站的近似位置（i：XYZ坐标）从buff中提取并存储到sta_t的pos数组中
            for (i=0,j=0;i<3;i++,j+=14) sta->pos[i]=str2num(buff,j,14);
        }
    }
    else if (strstr(label,"ANTENNA: DELTA H/E/N")) {    //若包含字符串"ANTENNA: DELTA H/E/N"
        if (sta) {	//每14字符赋值一次，记录天线各方向延迟
            for (i=0,j=0;i<3;i++,j+=14) del[i]=str2num(buff,j,14);
            sta->del[2]=del[0]; /* h */
            sta->del[0]=del[1]; /* e */
            sta->del[1]=del[2]; /* n */
        }
    }
    else if (strstr(label,"ANTENNA: DELTA X/Y/Z")) ; /* opt ver.3 */
    else if (strstr(label,"ANTENNA: PHASECENTER")) ; /* opt ver.3 */
    else if (strstr(label,"ANTENNA: B.SIGHT XYZ")) ; /* opt ver.3 */
    else if (strstr(label,"ANTENNA: ZERODIR AZI")) ; /* opt ver.3 */
    else if (strstr(label,"ANTENNA: ZERODIR XYZ")) ; /* opt ver.3 */
    else if (strstr(label,"CENTER OF MASS: XYZ" )) ; /* opt ver.3 */
    
    //观测数据类型读取
    else if (strstr(label,"SYS / # / OBS TYPES" )) { /* ver.3 */
        if (!(p=strchr(syscodes,buff[0]))) {	//如果buff[0]卫星系统不是"GREJSCI"之一，输出错误提示
            trace(2,"invalid system code: sys=%c\n",buff[0]);
            return;
        }
        i=(int)(p-syscodes);	//获取该系统在syscodes[]="GREJSCI"的下标
        n=(int)str2num(buff,3,3);//该系统下观测值类型数量
        for (j=nt=0,k=7;j<n;j++,k+=4) {//读取具体观测值类型，每行第7位开始，每次读4位；j从0到n-1，k从4开始
            if (k>58) {	//若k大于58，说明读完一行
                if (!fgets(buff,MAXRNXLEN,fp)) break;	//一行文件读取结束，重新循环，再读一行
                k=7;
            }
            //nt用于跟踪已经存储的观测类型的数量
            //观测类型数量还未达到最大值
            //将从buff中位置k开始提取的3个字符的观测类型存储到tobs数组的特定位置
            //递增nt以跟踪已存储的观测类型数量
            if (nt<MAXOBSTYPE-1) setstr(tobs[i][nt++],buff+k,3);
        }
        *tobs[i][nt]='\0';	//观测类型字符串末尾设置为空
        
        /* change BDS B1 code: 3.02 */	//3.02版本中修改北斗B1频段的代码
        if (i==5&&fabs(ver-3.02)<1e-3) {	//i=5：BDS；；ver：3.02
            for (j=0;j<nt;j++) if (tobs[i][j][1]=='1') tobs[i][j][1]='2';	//观测类型中第二个字符若为1则修改为2
        }
        /* if unknown code in ver.3, set default code */
        //如果没有该观测类型，则提示设置默认代码
        for (j=0;j<nt;j++) {
            if (tobs[i][j][2]) continue;	//若观测类型中第三个字符已设置，则继续
            //若第二个字符找不到频段代码frqcodes[]="1256789"对应的，则继续下一个观测类型
            //若找到则赋值给p
            if (!(p=strchr(frqcodes,tobs[i][j][1]))) continue;
            tobs[i][j][2]=defcodes[i][(int)(p-frqcodes)];//若第二个字符在frqcodes字符串中找到了，则赋值为第三个字符
            trace(2,"set default for unknown code: sys=%c code=%s\n",buff[0],
                  tobs[i][j]);//提示信息
        }
    }
    //2版本观测类型存储
    else if (strstr(label,"WAVELENGTH FACT L1/2")) ; /* opt ver.2 */
    else if (strstr(label,"# / TYPES OF OBSERV" )) { /* ver.2 */
        n=(int)str2num(buff,0,6);
        for (i=nt=0,j=10;i<n;i++,j+=6) {
            if (j>58) {
                if (!fgets(buff,MAXRNXLEN,fp)) break;
                j=10;
            }
            if (nt>=MAXOBSTYPE-1) continue;
            if (ver<=2.99) {
                setstr(str,buff+j,2);
                convcode(ver,SYS_GPS,str,tobs[0][nt]);
                convcode(ver,SYS_GLO,str,tobs[1][nt]);
                convcode(ver,SYS_GAL,str,tobs[2][nt]);
                convcode(ver,SYS_QZS,str,tobs[3][nt]);
                convcode(ver,SYS_SBS,str,tobs[4][nt]);
                convcode(ver,SYS_CMP,str,tobs[5][nt]);
            }
            nt++;
        }
        *tobs[0][nt]='\0';
    }
    else if (strstr(label,"SIGNAL STRENGTH UNIT")) ; /* opt ver.3 */
    else if (strstr(label,"INTERVAL"            )) ; /* opt */
    else if (strstr(label,"TIME OF FIRST OBS"   )) {
        if      (!strncmp(buff+48,"GPS",3)) *tsys=TSYS_GPS;
        else if (!strncmp(buff+48,"GLO",3)) *tsys=TSYS_UTC;
        else if (!strncmp(buff+48,"GAL",3)) *tsys=TSYS_GAL;
        else if (!strncmp(buff+48,"QZS",3)) *tsys=TSYS_QZS; /* ver.3.02 */
        else if (!strncmp(buff+48,"BDT",3)) *tsys=TSYS_CMP; /* ver.3.02 */
        else if (!strncmp(buff+48,"IRN",3)) *tsys=TSYS_IRN; /* ver.3.03 */
    }
    else if (strstr(label,"TIME OF LAST OBS"    )) ; /* opt */
    else if (strstr(label,"RCV CLOCK OFFS APPL" )) ; /* opt */
    else if (strstr(label,"SYS / DCBS APPLIED"  )) ; /* opt ver.3 */
    else if (strstr(label,"SYS / PCVS APPLIED"  )) ; /* opt ver.3 */
    else if (strstr(label,"SYS / SCALE FACTOR"  )) ; /* opt ver.3 */
    else if (strstr(label,"SYS / PHASE SHIFTS"  )) ; /* ver.3.01 */
    else if (strstr(label,"GLONASS SLOT / FRQ #")) { /* ver.3.02 */
        for (i=0;i<8;i++) {
            if (buff[4+i*7]!='R') continue;
            prn=(int)str2num(buff,5+i*7,2);
            fcn=(int)str2num(buff,8+i*7,2);
            if (prn<1||prn>MAXPRNGLO||fcn<-7||fcn>6) continue;
            if (nav) nav->glo_fcn[prn-1]=fcn+8;
        }
    }
    else if (strstr(label,"GLONASS COD/PHS/BIS" )) { /* ver.3.02 */
        if (sta) {
            sta->glo_cp_bias[0]=str2num(buff, 5,8);
            sta->glo_cp_bias[1]=str2num(buff,18,8);
            sta->glo_cp_bias[2]=str2num(buff,31,8);
            sta->glo_cp_bias[3]=str2num(buff,44,8);
        }
    }
    else if (strstr(label,"LEAP SECONDS"        )) { /* opt */
        if (nav) {
            nav->utc_gps[4]=str2num(buff, 0,6);
            nav->utc_gps[7]=str2num(buff, 6,6);
            nav->utc_gps[5]=str2num(buff,12,6);
            nav->utc_gps[6]=str2num(buff,18,6);
        }
    }
    else if (strstr(label,"# OF SALTELLITES"    )) { /* opt */
        /* skip */ ;
    }
    else if (strstr(label,"PRN / # OF OBS"      )) { /* opt */
        /* skip */ ;
    }
}

```

#### decode_obsepoch()：解码历元首行数据

```cpp
/* decode observation epoch --------------------------------------------------*/
static int decode_obsepoch(FILE *fp, char *buff, double ver, gtime_t *time,
                           int *flag, int *sats)
{
    int i,j,n;
    char satid[8]="";
    
    trace(4,"decode_obsepoch: ver=%.2f\n",ver);
    
    //2.x版本
    if (ver<=2.99) { /* ver.2 */
        if ((n=(int)str2num(buff,29,3))<=0) return 0;
        
        /* epoch flag: 3:new site,4:header info,5:external event */
        *flag=(int)str2num(buff,28,1);
        
        if (3<=*flag&&*flag<=5) return n;
        
        if (str2time(buff,0,26,time)) {
            trace(2,"rinex obs invalid epoch: epoch=%26.26s\n",buff);
            return 0;
        }
        for (i=0,j=32;i<n;i++,j+=3) {
            if (j>=68) {
                if (!fgets(buff,MAXRNXLEN,fp)) break;
                j=32;
            }
            if (i<MAXOBS) {
                strncpy(satid,buff+j,3);
                sats[i]=satid2no(satid);
            }
        }
    }
    //3.x版本
    /* epoch flag: 3:new site,4:header info,5:external event */
    else { /* ver.3 */
        if ((n=(int)str2num(buff,32,3))<=0) return 0;//读取卫星数量n，buff从第32到34字符；若n≤0，说明无效，返回0
        
        *flag=(int)str2num(buff,31,1);	//buff第31字符位，提取flag
        
        if (3<=*flag&&*flag<=5) return n;//若flag在3-5之间，说明有效，返回卫星数量n
        
        //若buff开头不是'>'
        //或buff第1到28字符位提取时间存储到time中，str2time函数输出-1，表示转换失败；输出0，表示成功
        //则提示数据记录无效，输出跟踪信息，返回0
        if (buff[0]!='>'||str2time(buff,1,28,time)) {
            trace(2,"rinex obs invalid epoch: epoch=%29.29s\n",buff);
            return 0;
        }
    }
    //输出跟踪信息，显示解析的时间戳和标志位
    trace(4,"decode_obsepoch: time=%s flag=%d\n",time_str(*time,3),*flag);
    return n;
}
```

#### decode_obsdata()：读取一个历元内一颗卫星的观测值

```cpp
/* decode observation data ---------------------------------------------------*/
static int decode_obsdata(FILE *fp, char *buff, double ver, int mask,
                          sigind_t *index, obsd_t *obs)
{
    sigind_t *ind;
    double val[MAXOBSTYPE]={0};
    uint8_t lli[MAXOBSTYPE]={0};
    char satid[8]="";
    int i,j,n,m,stat=1,p[MAXOBSTYPE],k[16],l[16];
    
    trace(4,"decode_obsdata: ver=%.2f\n",ver);
    
    if (ver>2.99) { /* ver.3 */
        sprintf(satid,"%.3s",buff);
        obs->sat=(uint8_t)satid2no(satid);
    }
    if (!obs->sat) {
        trace(4,"decode_obsdata: unsupported sat sat=%s\n",satid);
        stat=0;
    }
    else if (!(satsys(obs->sat,NULL)&mask)) {
        stat=0;
    }
    /* read observation data fields */
    switch (satsys(obs->sat,NULL)) {
        case SYS_GLO: ind=index+1; break;
        case SYS_GAL: ind=index+2; break;
        case SYS_QZS: ind=index+3; break;
        case SYS_SBS: ind=index+4; break;
        case SYS_CMP: ind=index+5; break;
        case SYS_IRN: ind=index+6; break;
        default:      ind=index  ; break;
    }
    for (i=0,j=ver<=2.99?0:3;i<ind->n;i++,j+=16) {
        
        if (ver<=2.99&&j>=80) { /* ver.2 */
            if (!fgets(buff,MAXRNXLEN,fp)) break;
            j=0;
        }
        if (stat) {
            val[i]=str2num(buff,j,14)+ind->shift[i];
            lli[i]=(uint8_t)str2num(buff,j+14,1)&3;
        }
    }
    if (!stat) return 0;
    
    for (i=0;i<NFREQ+NEXOBS;i++) {
        obs->P[i]=obs->L[i]=0.0; obs->D[i]=0.0f;
        obs->SNR[i]=obs->LLI[i]=obs->code[i]=0;
    }
    /* assign position in observation data */
    for (i=n=m=0;i<ind->n;i++) {
        
        p[i]=(ver<=2.11)?ind->idx[i]:ind->pos[i];
        
        if (ind->type[i]==0&&p[i]==0) k[n++]=i; /* C1? index */
        if (ind->type[i]==0&&p[i]==1) l[m++]=i; /* C2? index */
    }
    if (ver<=2.11) {
        
        /* if multiple codes (C1/P1,C2/P2), select higher priority */
        if (n>=2) {
            if (val[k[0]]==0.0&&val[k[1]]==0.0) {
                p[k[0]]=-1; p[k[1]]=-1;
            }
            else if (val[k[0]]!=0.0&&val[k[1]]==0.0) {
                p[k[0]]=0; p[k[1]]=-1;
            }
            else if (val[k[0]]==0.0&&val[k[1]]!=0.0) {
                p[k[0]]=-1; p[k[1]]=0;
            }
            else if (ind->pri[k[1]]>ind->pri[k[0]]) {
                p[k[1]]=0; p[k[0]]=NEXOBS<1?-1:NFREQ;
            }
            else {
                p[k[0]]=0; p[k[1]]=NEXOBS<1?-1:NFREQ;
            }
        }
        if (m>=2) {
            if (val[l[0]]==0.0&&val[l[1]]==0.0) {
                p[l[0]]=-1; p[l[1]]=-1;
            }
            else if (val[l[0]]!=0.0&&val[l[1]]==0.0) {
                p[l[0]]=1; p[l[1]]=-1;
            }
            else if (val[l[0]]==0.0&&val[l[1]]!=0.0) {
                p[l[0]]=-1; p[l[1]]=1; 
            }
            else if (ind->pri[l[1]]>ind->pri[l[0]]) {
                p[l[1]]=1; p[l[0]]=NEXOBS<2?-1:NFREQ+1;
            }
            else {
                p[l[0]]=1; p[l[1]]=NEXOBS<2?-1:NFREQ+1;
            }
        }
    }
    /* save observation data */
    for (i=0;i<ind->n;i++) {
        if (p[i]<0||val[i]==0.0) continue;
        switch (ind->type[i]) {
            case 0: obs->P[p[i]]=val[i]; obs->code[p[i]]=ind->code[i]; break;
            case 1: obs->L[p[i]]=val[i]; obs->LLI [p[i]]=lli[i];    break;
            case 2: obs->D[p[i]]=(float)val[i];                     break;
            case 3: obs->SNR[p[i]]=(uint16_t)(val[i]/SNR_UNIT+0.5); break;
        }
    }
    trace(4,"decode_obsdata: time=%s sat=%2d\n",time_str(obs->time,0),obs->sat);
    return 1;
}
```

#### **readrnxobsb（）：**读取一个观测历元的观测数据

| FILE   *fp                   | I    | 传入的Rinex文件指针 |
| ---------------------------- | ---- | ------------------- |
| const   char   *opt          | I    | 选项                |
| double   ver                 | I    | Rinex文件版本       |
| int   *tsys                  | I    | 时间系统            |
| char   tobs[][MAXOBSTYPE][4] | I    | 观测值类型数组      |
| int   *flag                  | I    | 历元信息状态        |

| obsd_t   *data | O    | obsd_t类型的观测值数组 |
| -------------- | ---- | ---------------------- |
| sta_t   *sta   | O    | 卫星数组               |

```cpp
/* read RINEX observation data body ------------------------------------------*/
static int readrnxobsb(FILE *fp, const char *opt, double ver, int *tsys,
                       char tobs[][MAXOBSTYPE][4], int *flag, obsd_t *data,
                       sta_t *sta)
{
    gtime_t time={0};	//定义一个时间变量，初始化为0
    sigind_t index[NUMSYS]={{0}};	//定义一个信号索引数组，初始化为0
    char buff[MAXRNXLEN];
    int i=0,n=0,nsat=0,nsys=NUMSYS,sats[MAXOBS]={0},mask;
    
    /* set system mask */	//获取系统掩码
    mask=set_sysmask(opt);	//（SYS_GPS）'G'：GPS    （SYS_CMP）'C'：BeiDou
    
    /* set signal index */	//设置信号指标
    //调用set_index()，将tobs数组中存的观测值类型信息存到sigind_t类型的index[ ]结构体数组中
	//每个传入的tobs都存了一个卫星系统的观测值类型，同理index[]的一个元素就存一个卫星系统的所有观测值类型
    if (nsys>=1) set_index(ver,SYS_GPS,opt,tobs[0],index  );
    if (nsys>=2) set_index(ver,SYS_GLO,opt,tobs[1],index+1);
    if (nsys>=3) set_index(ver,SYS_GAL,opt,tobs[2],index+2);
    if (nsys>=4) set_index(ver,SYS_QZS,opt,tobs[3],index+3);
    if (nsys>=5) set_index(ver,SYS_SBS,opt,tobs[4],index+4);
    if (nsys>=6) set_index(ver,SYS_CMP,opt,tobs[5],index+5);
    if (nsys>=7) set_index(ver,SYS_IRN,opt,tobs[6],index+6);
    
    /* read record */
    while (fgets(buff,MAXRNXLEN,fp)) {	//利用fgets()函数缓存一行数据
        //记录一个观测历元的有效性、时间和卫星数
        /* decode observation epoch */
        //若为第一行，则调用decode_obsepoch()函数解码首行数据（包括历元时刻、卫星数、卫星编号、历元状态等信息），将信息保存
        if (i==0) {
            if ((nsat=decode_obsepoch(fp,buff,ver,&time,flag,sats))<=0) {
                continue;
            }
        }
        //如果不是第一行则调用decode_obsdata()函数对该行观测数据进行数据解码
        else if ((*flag<=2||*flag==6)&&n<MAXOBS) {
            data[n].time=time;
            data[n].sat=(uint8_t)sats[i-1];
            
            /* decode RINEX observation data */
            if (decode_obsdata(fp,buff,ver,mask,index,data+n)) n++;
        }
        else if (*flag==3||*flag==4) { /* new site or header info follows */
            
            /* decode RINEX observation data file header */
            decode_obsh(fp,buff,ver,tsys,tobs,NULL,sta);
        }
        if (++i>nsat) return n;
    }
    return -1;
}
```

#### **readrnxobs()：**读取o文件中全部观测值数据

```cpp
/* read RINEX observation data -----------------------------------------------*/
static int readrnxobs(FILE *fp, gtime_t ts, gtime_t te, double tint,
                      const char *opt, int rcv, double ver, int *tsys,
                      char tobs[][MAXOBSTYPE][4], obs_t *obs, sta_t *sta)
{
    obsd_t *data;
    uint8_t slips[MAXSAT][NFREQ+NEXOBS]={{0}};
    int i,n,flag=0,stat=0;
    
    trace(4,"readrnxobs: rcv=%d ver=%.2f tsys=%d\n",rcv,ver,tsys);
    
    if (!obs||rcv>MAXRCV) return 0;
    
    if (!(data=(obsd_t *)malloc(sizeof(obsd_t)*MAXOBS))) return 0;
    
    /* read RINEX observation data body */
    while ((n=readrnxobsb(fp,opt,ver,tsys,tobs,&flag,data,sta))>=0&&stat>=0) {
        
        for (i=0;i<n;i++) {
            
            /* UTC -> GPST */
            if (*tsys==TSYS_UTC) data[i].time=utc2gpst(data[i].time);
            
            /* save cycle slip */
            saveslips(slips,data+i);
        }
        /* screen data by time */
        if (n>0&&!screent(data[0].time,ts,te,tint)) continue;
        
        for (i=0;i<n;i++) {
            
            /* restore cycle slip */
            restslips(slips,data+i);
            
            data[i].rcv=(uint8_t)rcv;
            
            /* save obs data */
            if ((stat=addobsdata(obs,data+i))<0) break;
        }
    }
    trace(4,"readrnxobs: nobs=%d stat=%d\n",obs->n,stat);
    
    free(data);
    
    return stat;
}
```

### 读取NAV星历文件

#### decode_navh()：

```cpp
/* decode RINEX NAV header ---------------------------------------------------*/
static void decode_navh(char *buff, nav_t *nav)
{
    int i,j;
    char *label=buff+60;
    
    trace(4,"decode_navh:\n");
    
    if      (strstr(label,"ION ALPHA"           )) { /* opt ver.2 */
        if (nav) {
            for (i=0,j=2;i<4;i++,j+=12) nav->ion_gps[i]=str2num(buff,j,12);
        }
    }
    else if (strstr(label,"ION BETA"            )) { /* opt ver.2 */
        if (nav) {
            for (i=0,j=2;i<4;i++,j+=12) nav->ion_gps[i+4]=str2num(buff,j,12);
        }
    }
    else if (strstr(label,"DELTA-UTC: A0,A1,T,W")) { /* opt ver.2 */
        if (nav) {
            for (i=0,j=3;i<2;i++,j+=19) nav->utc_gps[i]=str2num(buff,j,19);
            for (;i<4;i++,j+=9) nav->utc_gps[i]=str2num(buff,j,9);
        }
    }
    else if (strstr(label,"IONOSPHERIC CORR"    )) { /* opt ver.3 */
        if (nav) {
            if (!strncmp(buff,"GPSA",4)) {
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_gps[i]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"GPSB",4)) {
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_gps[i+4]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"GAL",3)) {
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_gal[i]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"QZSA",4)) { /* v.3.02 */
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_qzs[i]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"QZSB",4)) { /* v.3.02 */
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_qzs[i+4]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"BDSA",4)) { /* v.3.02 */
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_cmp[i]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"BDSB",4)) { /* v.3.02 */
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_cmp[i+4]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"IRNA",4)) { /* v.3.03 */
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_irn[i]=str2num(buff,j,12);
            }
            else if (!strncmp(buff,"IRNB",4)) { /* v.3.03 */
                for (i=0,j=5;i<4;i++,j+=12) nav->ion_irn[i+4]=str2num(buff,j,12);
            }
        }
    }
    else if (strstr(label,"TIME SYSTEM CORR"    )) { /* opt ver.3 */
        if (nav) {
            if (!strncmp(buff,"GPUT",4)) {
                nav->utc_gps[0]=str2num(buff, 5,17);
                nav->utc_gps[1]=str2num(buff,22,16);
                nav->utc_gps[2]=str2num(buff,38, 7);
                nav->utc_gps[3]=str2num(buff,45, 5);
            }
            else if (!strncmp(buff,"GLUT",4)) {
                nav->utc_glo[0]=-str2num(buff,5,17); /* tau_C */
            }
            else if (!strncmp(buff,"GLGP",4)) {
                nav->utc_glo[1]=str2num(buff, 5,17); /* tau_GPS */
            }
            else if (!strncmp(buff,"GAUT",4)) { /* v.3.02 */
                nav->utc_gal[0]=str2num(buff, 5,17);
                nav->utc_gal[1]=str2num(buff,22,16);
                nav->utc_gal[2]=str2num(buff,38, 7);
                nav->utc_gal[3]=str2num(buff,45, 5);
            }
            else if (!strncmp(buff,"QZUT",4)) { /* v.3.02 */
                nav->utc_qzs[0]=str2num(buff, 5,17);
                nav->utc_qzs[1]=str2num(buff,22,16);
                nav->utc_qzs[2]=str2num(buff,38, 7);
                nav->utc_qzs[3]=str2num(buff,45, 5);
            }
            else if (!strncmp(buff,"BDUT",4)) { /* v.3.02 */
                nav->utc_cmp[0]=str2num(buff, 5,17);
                nav->utc_cmp[1]=str2num(buff,22,16);
                nav->utc_cmp[2]=str2num(buff,38, 7);
                nav->utc_cmp[3]=str2num(buff,45, 5);
            }
            else if (!strncmp(buff,"SBUT",4)) { /* v.3.02 */
                nav->utc_sbs[0]=str2num(buff, 5,17);
                nav->utc_sbs[1]=str2num(buff,22,16);
                nav->utc_sbs[2]=str2num(buff,38, 7);
                nav->utc_sbs[3]=str2num(buff,45, 5);
            }
            else if (!strncmp(buff,"IRUT",4)) { /* v.3.03 */
                nav->utc_irn[0]=str2num(buff, 5,17);
                nav->utc_irn[1]=str2num(buff,22,16);
                nav->utc_irn[2]=str2num(buff,38, 7);
                nav->utc_irn[3]=str2num(buff,45, 5);
                nav->utc_irn[8]=0.0; /* A2 */
            }
        }
    }
    else if (strstr(label,"LEAP SECONDS"        )) { /* opt */
        if (nav) {
            nav->utc_gps[4]=str2num(buff, 0,6);
            nav->utc_gps[7]=str2num(buff, 6,6);
            nav->utc_gps[5]=str2num(buff,12,6);
            nav->utc_gps[6]=str2num(buff,18,6);
        }
    }
}
```

#### readrnxnavb()：读取一个历元的星历数据，添加到eph结构体中

```cpp
/* read RINEX navigation data body -------------------------------------------*/
static int readrnxnavb(FILE *fp, const char *opt, double ver, int sys,
                       int *type, eph_t *eph, geph_t *geph, seph_t *seph)
{
    gtime_t toc;
    double data[64];
    int i=0,j,prn,sat=0,sp=3,mask;
    char buff[MAXRNXLEN],id[8]="",*p;
    
    trace(4,"readrnxnavb: ver=%.2f sys=%d\n",ver,sys);
    
    /* set system mask */
    mask=set_sysmask(opt);
    
    while (fgets(buff,MAXRNXLEN,fp)) {
        
        if (i==0) {
            
            /* decode satellite field */
            if (ver>=3.0||sys==SYS_GAL||sys==SYS_QZS) { /* ver.3 or GAL/QZS */
                sprintf(id,"%.3s",buff);
                sat=satid2no(id);
                sp=4;
                if (ver>=3.0) {
                    sys=satsys(sat,NULL);
                    if (!sys) {
                        sys=(id[0]=='S')?SYS_SBS:((id[0]=='R')?SYS_GLO:SYS_GPS);
                    }
                }
            }
            else {
                prn=(int)str2num(buff,0,2);
                
                if (sys==SYS_SBS) {
                    sat=satno(SYS_SBS,prn+100);
                }
                else if (sys==SYS_GLO) {
                    sat=satno(SYS_GLO,prn);
                }
                else if (93<=prn&&prn<=97) { /* extension */
                    sat=satno(SYS_QZS,prn+100);
                }
                else sat=satno(SYS_GPS,prn);
            }
            /* decode Toc field */
            if (str2time(buff+sp,0,19,&toc)) {
                trace(2,"rinex nav toc error: %23.23s\n",buff);
                return 0;
            }
            /* decode data fields */
            for (j=0,p=buff+sp+19;j<3;j++,p+=19) {
                data[i++]=str2num(p,0,19);
            }
        }
        else {
            /* decode data fields */
            for (j=0,p=buff+sp;j<4;j++,p+=19) {
                data[i++]=str2num(p,0,19);
            }
            /* decode ephemeris */
            if (sys==SYS_GLO&&i>=15) {
                if (!(mask&sys)) return 0;
                *type=1;
                return decode_geph(ver,sat,toc,data,geph);
            }
            else if (sys==SYS_SBS&&i>=15) {
                if (!(mask&sys)) return 0;
                *type=2;
                return decode_seph(ver,sat,toc,data,seph);
            }
            else if (i>=31) {
                if (!(mask&sys)) return 0;
                *type=0;
                return decode_eph(ver,sat,toc,data,eph);
            }
        }
    }
    return -1;
}
```

#### readrnxnav()：读取星历文件，添加到nav结构体中

```cpp
/* read RINEX navigation data ------------------------------------------------*/
static int readrnxnav(FILE *fp, const char *opt, double ver, int sys,
                      nav_t *nav)
{
    eph_t eph;
    geph_t geph;
    seph_t seph;
    int stat,type;
    
    trace(3,"readrnxnav: ver=%.2f sys=%d\n",ver,sys);
    
    if (!nav) return 0;
    
    /* read RINEX navigation data body */
    while ((stat=readrnxnavb(fp,opt,ver,sys,&type,&eph,&geph,&seph))>=0) {
        
        /* add ephemeris to navigation data */
        if (stat) {
            switch (type) {
                case 1 : stat=add_geph(nav,&geph); break;
                case 2 : stat=add_seph(nav,&seph); break;
                default: stat=add_eph (nav,&eph ); break;
            }
            if (!stat) return 0;
        }
    }
    return nav->n>0||nav->ng>0||nav->ns>0;
}
```

### 

