# 音视频同步



![image-20231115173017257](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231115173017257.png)

![image-20231115210813520](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231115210813520.png)

#### 以音频时钟为准：

![image-20231115185855355](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231115185855355.png)

每一个新的frame都会更新这个函数：Drift。

![image-20231115211714488](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231115211714488.png)

get：设置新的参考时钟轴，视频时钟用来比较diff。

ptsDrift：是音频pts和当前时间的差值，而因为视频时钟要参考中间轴(**当前音频时间轴**)，所以base = cur+ptsDrift。

而视频pts - base = 当前视频帧和当前音频帧时间轴的差值。

![image-20231115211904567](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231115211904567.png)



#### 以外部时钟为准：

![image-20231118213718359](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231118213718359.png)

![image-20231115173941839](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231115173941839.png)



以时间轴为准，音视频帧的PTS进行对比。

因为操作系统存在误差，所以可以设置一个阈值，当audio_frame_time  -  time = diff <=阈值，即可播放。

或者延时等待播放，当time到达PTS，就播放pts。

![image-20231115174008660](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231115174008660.png)

使用 diff_ret 和前面获取的时间进行对比，pts比实际时间大，就调用sleep函数等一等，否则就直接播放出来。这样就达到了某种意义上的同步了。

缺点：音视频互相不了解彼此的状态，可能会导致不同步。

----------------

------------------

# 倍速播放：



### 视频倍速：

PTS进行变化，乘除运算。

当8倍速时，由于一秒播放的画面太多，所以可以drop帧

当16倍速时，有些包可以不解码，只解码关键帧。 

 ![image-20231116095508786](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116095508786.png)

![image-20231116100150530](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116100150530.png)

![image-20231116164622908](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116164622908.png)

### 音频倍速：

 因为采样点时刻有值，不能直接乘除运算PTS，有可能会忽略某些采样点。产生杂音。

所以需要重采样操作。

由于通过设置SDL的参数进行播放，当参数设置好，SDL只会按照参数读取数据。

![image-20231116103137522](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116103137522.png)

采样率为48000KHz时，说明一秒(或者理解为一次)取48000个采样点，那么当我们想快速播放时，只需要把每一帧的采样点变化缩短即可。比如一帧samples = 1024，重采样为512，那么SDL就会多读取很多帧，直到满足48000。

  持续时间不变是相对于每秒钟的总采样点数而言的。如果你在一秒内仍然保持了相同的总采样点数，就可以说持续时间没有变化。但这并不意味着音频播放速度没有改变。持续时间不变是相对于每秒总采样点数而言的，而不是相对于每帧的时间间隔。

![image-20231116110951524](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116110951524.png)

![image-20231116112412513](D:\typora-image\image-20231116112412513.png)

![image-20231116112429222](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116112429222.png)

### 实时变速倍速：

![image-20231118185818983](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231118185818983.png)

当视频PTS发生改变，同步时钟也需要改变。

![image-20231118191212476](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231118191212476.png)

**以外部时钟为基准变速：**

变速切换时，要根据上一次的倍速参数进行设置：

比如new_pts = last_pts * last_speed / new_speed.

而elapsed假设和pts同步，则计算公式同理。

**以音频为基准变速：**

直接对时间轴进行公式计算，然后重新设置drift。

----------

-------------



# 暂停：

![image-20231116115200137](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116115200137.png)

帧队列暂停：当pasue==ture时循环休眠即可。

![image-20231116115704741](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116115704741.png)

##### ’'以外部时钟为基准同步''：

记录暂停的时间t1，假设之后播放的时间是t2，记录diff = t2 - t1； start_time1 =start_time +  diff

当开始播放后，音视频同步计算：t2 - (start_time + diff) = base;

![image-20231116120455834](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116120455834.png)

![image-20231116120526066](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116120526066.png)![image-20231116120634712](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116120634712.png)休眠，不拿数据



##### '以音频时钟为基准同步''：

记录上次PtsDrift。记为Last_PtsDrift

因为正常计算视频帧时：视频帧pts - base，base = cur + PtsDrift ( Last_PtsDrift )

而当暂停后过了offset时刻后在播放，需要减去offset时间，：视频帧pts - base-x = 视频帧pts - cur - Last_PtsDrift - offset

而PtsDrift可以看做被更新为 PtsDrift = Last_PtsDrift - offset

### 暂停后播放：

![image-20231116120849888](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116120849888.png)

--------

--------

# 逐帧播放：

**对于视频帧逐帧播放：**

原理：先暂停，然后当点击逐帧按钮后，每播放一个视频帧就暂停一次。快速的播放+暂停。

当点击逐帧按钮 ： 

启动同步时钟函数(开始播放)，接着对解码帧队列的isStep==true，如下图执行帧处理逻辑。

取出一帧并且进行同步控制。最后播放

![image-20231116163101931](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116163101931.png)

![image-20231116165014919](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116165014919.png)![image-20231116170416878](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116170416878.png)

​                                    左图视频帧逐帧                                                       右图音频帧逐帧

播放完立即在**视频帧队列**的run()中**设置暂停同步时钟函数pauseSyncClock()**，并且while继续循环到上面，进行休眠。

因为音频帧比视频帧快，所以，有可能以音频逐帧播放而视频帧没有显示出来。

所以**暂停时钟函数**要在视频帧队列中显示：当显示出视频帧在暂停时钟。

-------

---------

# Seek跳转播放：



![image-20231116182418472](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116182418472.png)

![image-20231116182540795](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116182540795.png)

记录跳转时间，seek、清理解码器缓存和帧队列数据。

每一次跳转，serial++。并且设置对应的解码器serial，所以更新新的frame的serial

**更新时钟：** 

![](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116183005570.png)

![image-20231116183005570](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116183005570.png)

**以外部时钟为基准：**

相当于重新计算start_time:

因为av_gettime()返回的是当前程序运行时间，而av_gettime() - start_time = 当前总播放的持续时间total。

total - post_time是两者的差值diff，如果diff<0 ，说明是快进(向后跳转)、diff>0，说明是快退(向前跳转)



若想av_gettime() - start_time跟上跳转后的PTS，则需start_time1 = start_time + diff更新开始时间。

或者av_gettime() - start_time - diff == av_gettime() - start_time1.

**以音频时钟为基准：**

直接设置跳转时间为时间轴。

**seek优化：**

![image-20231116195439939](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231116195439939.png)

对比跳转位置帧的PTS和获取到的帧PTS进行对比，若不相等，则进行drop，不进行同步控制和写入操作。

![image-20231117114614079](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20231117114614079.png)