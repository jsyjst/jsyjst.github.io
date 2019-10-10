---
title: Android自定义View-简约风歌词控件
date: 2019-10-10 10:28:27
tags: 自定义View
categories: 自定义View
---

# Android自定义View-简约风歌词控件

# 前言

最近重构了之前的音乐播放器（音乐播放器的源码地址在文章底部），添加了许多功能，比如歌词，下载功能等。这篇文章就让我们聊聊歌词控件的实现（歌词控件也已经开源，地址也在文章底部），先上效果图，如果感觉海星，就继续瞧下去！

![](lrcView.gif)

看到这里，估计你对这个控件还有点感兴趣的吧，那接下来就让我们来瞧瞧实现这个歌词控件需要做些什么！（如果想直接使用就直接点击文末中的开源库地址，里面会有添加依赖库的说明）

# 一、 歌词解析

首先，我们得知道正常的歌词格式是怎样的，大概是长这个样子：

```java
[ti:喜欢你]
[ar:.]
[al:]
[by:]
[offset:0]
[00:00.10]喜欢你 - G.E.M. 邓紫棋 (Gem Tang)
[00:00.20]词：黄家驹
[00:00.30]曲：黄家驹
[00:00.40]编曲：Lupo Groinig
[00:00.50]
[00:12.65]细雨带风湿透黄昏的街道
[00:18.61]抹去雨水双眼无故地仰望
[00:24.04]望向孤单的晚灯
[00:26.91]
[00:27.44]是那伤感的记忆
[00:30.52]
[00:34.12]再次泛起心里无数的思念
[00:39.28]
[00:40.10]以往片刻欢笑仍挂在脸上
[00:45.49]愿你此刻可会知
[00:48.23]
[00:48.95]是我衷心的说声
[00:53.06]
[00:54.35]喜欢你 那双眼动人
[00:59.35]
[01:00.10]笑声更迷人
[01:02.37]
[01:03.15]愿再可 轻抚你
[01:08.56]
[01:09.35]那可爱面容
[01:12.40]挽手说梦话
[01:14.78]
[01:15.48]像昨天 你共我
[01:20.84]
[01:26.32]满带理想的我曾经多冲动
[01:32.45]屡怨与她相爱难有自由
[01:37.82]愿你此刻可会知
[01:40.40]
[01:41.25]是我衷心的说声
[01:44.81]
[01:46.39]喜欢你 那双眼动人
[01:51.72]
[01:52.42]笑声更迷人
[01:54.75]
[01:55.48]愿再可 轻抚你
[02:00.93]
[02:01.68]那可爱面容
[02:03.99]
[02:04.73]挽手说梦话
[02:07.13]
[02:07.82]像昨天 你共我
[02:14.53]
[02:25.54]每晚夜里自我独行
[02:29.30]随处荡 多冰冷
[02:35.40]
[02:37.83]以往为了自我挣扎
[02:41.62]从不知 她的痛苦
[02:52.02]
[02:54.11]喜欢你 那双眼动人
[03:00.13]笑声更迷人
[03:02.38]
[03:03.14]愿再可 轻抚你
[03:08.77]
[03:09.33]那可爱面容
[03:11.71]
[03:12.41]挽手说梦话
[03:14.61]
[03:15.45]像昨天 你共我
```

从上面可以看出这种格式前面是开始时间，从左往右一一对应分，秒，毫秒，后面就是歌词。所以我们要创建一个实体类来保存每一句的歌词信息。

### 1.歌词实体类LrcBean

```java
public class LrcBean {
    private String lrc;//歌词
    private long start;//开始时间
    private long end;//结束时间

    public String getLrc() {
        return lrc;
    }

    public void setLrc(String lrc) {
        this.lrc = lrc;
    }

    public long getStart() {
        return start;
    }

    public void setStart(long start) {
        this.start = start;
    }

    public long getEnd() {
        return end;
    }

    public void setEnd(long end) {
        this.end = end;
    }
}
```

每句歌词，我们需要开始时间，结束时间和歌词这些信息，那么你就会有疑问了？上面提到的歌词格式好像只有歌词开始时间，那我们怎么知道结束时间呢？其实很简单，这一句歌词的开始时间就是上一句歌词的结束时间。有了歌词实体类，我们就得开始对歌词进行解析了！

### 2. 解析歌词工具类LrcUtil

```java
public class LrcUtil {

    /**
     * 解析歌词，将字符串歌词封装成LrcBean的集合
     * @param lrcStr 字符串的歌词，歌词有固定的格式，一般为
     * [ti:喜欢你]
     * [ar:.]
     * [al:]
     * [by:]
     * [offset:0]
     * [00:00.10]喜欢你 - G.E.M. 邓紫棋 (Gem Tang)
     * [00:00.20]词：黄家驹
     * [00:00.30]曲：黄家驹
     * [00:00.40]编曲：Lupo Groinig
     * @return 歌词集合
     */
    public static List<LrcBean> parseStr2List(String lrcStr){
        List<LrcBean> res = new ArrayList<>();
        //根据转行字符对字符串进行分割
        String[] subLrc = lrcStr.split("\n");
        //跳过前四行，从第五行开始，因为前四行的歌词我们并不需要
        for (int i = 5; i < subLrc.length; i++) {
            String lineLrc = subLrc[i];
            //[00:00.10]喜欢你 - G.E.M. 邓紫棋 (Gem Tang)
            String min = lineLrc.substring(lineLrc.indexOf("[")+1,lineLrc.indexOf("[")+3);
            String sec = lineLrc.substring(lineLrc.indexOf(":")+1,lineLrc.indexOf(":")+3);
            String mills = lineLrc.substring(lineLrc.indexOf(".")+1,lineLrc.indexOf(".")+3);
            //进制转化，转化成毫秒形式的时间
            long startTime = getTime(min,sec,mills);
            //歌词
            String lrcText = lineLrc.substring(lineLrc.indexOf("]")+1);
            //有可能是某个时间段是没有歌词，则跳过下面
            if(lrcText.equals("")) continue;
            //在第一句歌词中有可能是很长的，我们只截取一部分，即歌曲加演唱者
            //比如 光年之外 (《太空旅客（Passengers）》电影中国区主题曲) - G.E.M. 邓紫棋 (Gem Tang)
            if (i == 5) {
                int lineIndex = lrcText.indexOf("-");
                int first = lrcText.indexOf("(");
                if(first<lineIndex&&first!=-1){
                    lrcText = lrcText.substring(0,first)+lrcText.substring(lineIndex);
                }
                LrcBean lrcBean = new LrcBean();
                lrcBean.setStart(startTime);
                lrcBean.setLrc(lrcText);
                res.add(lrcBean);
                continue;
            }
            //添加到歌词集合中
            LrcBean lrcBean = new LrcBean();
            lrcBean.setStart(startTime);
            lrcBean.setLrc(lrcText);
            res.add(lrcBean);
            //如果是最后一句歌词，其结束时间是不知道的，我们将人为的设置为开始时间加上100s
            if(i == subLrc.length-1){
                res.get(res.size()-1).setEnd(startTime+100000);
            }else if(res.size()>1){
                //当集合数目大于1时，这句的歌词的开始时间就是上一句歌词的结束时间
                res.get(res.size()-2).setEnd(startTime);
            }

        }
        return res;
    }

    /**
     *  根据时分秒获得总时间
     * @param min 分钟
     * @param sec 秒
     * @param mills 毫秒
     * @return 总时间
     */
    private static long getTime(String min,String sec,String mills){
        return Long.valueOf(min)*60*1000+Long.valueOf(sec)*1000+Long.valueOf(mills);
    }
}

```

相信上面的代码和注释已经将这个歌词解析解释的挺明白了，需要注意的是上面对i=5,也就是歌词真正开始的第一句做了特殊处理，因为i=5这句有可能是很长的，假设i=5是“光年之外 (《太空旅客（Passengers）》电影中国区主题曲) - G.E.M. 邓紫棋 (Gem Tang)”这句歌词，如果我们不做特殊处理，在后面绘制的时候，就会发现这句歌词会超过屏幕大小，很影响美观，所以我们只截取歌曲名和演唱者，有些说明直接省略掉了。解析好了歌词，接下来就是重头戏-歌词绘制！


# 二、歌词绘制

歌词绘制就涉及到了自定义View的知识，所以还未接触自定义View的小伙伴需要先去看看自定View的基础知识。歌词绘制的主要工作主要由下面几部分构成：

- 为歌词控件设置自定义属性，在构造方法中获取并设置自定义属性的默认值
- 初始化两支画笔。分别是歌词普通画笔，歌词高亮画笔。
- 获取当前播放歌词的位置
- 画歌词，根据当前播放歌词的位置来决定用哪支画笔画
- 歌词随歌曲播放同步滑动
- 重新绘制

### 1.设置自定View属性，在代码中设置默认值

在res文件中的values中新建一个attrs.xml文件，然后定义歌词的自定义View属性

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="LrcView">
        <attr name="highLineTextColor" format="color|reference|integer"/>
        <attr name="lrcTextColor" format="color|reference|integer"/>
        <attr name="lineSpacing" format="dimension"/>
        <attr name="textSize" format="dimension"/>
    </declare-styleable>
</resources>
```

这里只自定义了歌词颜色，歌词高亮颜色，歌词大小，歌词行间距的属性，可根据自己需要自行添加。

然后在Java代码中，设置默认值。

```java
    private int lrcTextColor;//歌词颜色
    private int highLineTextColor;//当前歌词颜色
    private int width, height;//屏幕宽高
    private int lineSpacing;//行间距
    private int textSize;//字体大小

    public LrcView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.LrcView);
        lrcTextColor = ta.getColor(R.styleable.LrcView_lrcTextColor, Color.GRAY);
        highLineTextColor = ta.getColor(R.styleable.LrcView_highLineTextColor, Color.BLUE);
        float fontScale = context.getResources().getDisplayMetrics().scaledDensity;
        float scale = context.getResources().getDisplayMetrics().density;
        //默认字体大小为16sp
        textSize = ta.getDimensionPixelSize(R.styleable.LrcView_textSize, (int) (16 * fontScale));
        //默认行间距为30dp
        lineSpacing = ta.getDimensionPixelSize(R.styleable.LrcView_lineSpacing, (int) (30 * scale));
        //回收
        ta.recycle();
    }
```

### 2. 初始化两支画笔

```java
    private void init() {
        //初始化歌词画笔
        dPaint = new Paint();
        dPaint.setStyle(Paint.Style.FILL);//填满
        dPaint.setAntiAlias(true);//抗锯齿
        dPaint.setColor(lrcTextColor);//画笔颜色
        dPaint.setTextSize(textSize);//歌词大小
        dPaint.setTextAlign(Paint.Align.CENTER);//文字居中

        //初始化当前歌词画笔
        hPaint = new Paint();
        hPaint.setStyle(Paint.Style.FILL);
        hPaint.setAntiAlias(true);
        hPaint.setColor(highLineTextColor);
        hPaint.setTextSize(textSize);
        hPaint.setTextAlign(Paint.Align.CENTER);
    }
```

我们把初始化的方法放到了构造方法中，这样就可以避免在重绘时再次初始化。另外由于我们把init方法只放到了第三个构造方法中，所以在上面两个构造方法需要将super改成this，这样就能保证哪个构造方法都能执行init方法

```java
	public LrcView(Context context) {
        this(context, null);
    }

    public LrcView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public LrcView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.LrcView);
        ......
        //回收
        ta.recycle();
        init();
    }

```

### 3. 重复执行onDraw方法

因为后面的步骤都是在onDraw方法中执行的，所以我们先贴出onDraw方法中的代码

```java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        
        getMeasuredWidthAndHeight();//得到测量后的宽高
        getCurrentPosition();//得到当前歌词的位置
        drawLrc(canvas);//画歌词
        scrollLrc();//歌词滑动
        postInvalidateDelayed(100);//延迟0.1s刷新
    }
```



**1.获得控件的测量后的宽高**

```java
    private int width, height;//屏幕宽高
	private void getMeasuredWidthAndHeight(){
        if (width == 0 || height == 0) {
            width = getMeasuredWidth();
            height = getMeasuredHeight();
        }
    }
```

为什么要获得控件的宽高呢？因为在下面我们需要画歌词，画歌词时需要画的位置，这时候就需要用到控件的宽高了。

**2. 得到当前歌词的位置**

```java
    private List<LrcBean> lrcBeanList;//歌词集合
    private int currentPosition;//当前歌词的位置
    private MediaPlayer player;//当前的播放器


    private void getCurrentPosition() {
        int curTime = player.getCurrentPosition();
        //如果当前的时间大于10分钟，证明歌曲未播放，则当前位置应该为0
        if (curTime < lrcBeanList.get(0).getStart()||curTime>10*60*1000) {
            currentPosition = 0;
            return;
        } else if (curTime > lrcBeanList.get(lrcBeanList.size() - 1).getStart()) {
            currentPosition = lrcBeanList.size() - 1;
            return;
        }
        for (int i = 0; i < lrcBeanList.size(); i++) {
            if (curTime >= lrcBeanList.get(i).getStart() && curTime <= lrcBeanList.get(i).getEnd()) {
                currentPosition = i;
            }
        }
    }
```

我们根据当前播放的歌曲时间来遍历歌词集合，从而判断当前播放的歌词的位置。细心的你可能会发现在currentPosition = 0中有个curTime>10 * 60 *1000的判断，这是因为在实际使用中发现当player还未播放时，这时候得到的curTime会很大，所以才有了这个判断（因为正常的歌曲不会超过10分钟）。

在这个方法我们会发现出现了歌词集合和播放器，你可能会感到困惑，这些不是还没赋值吗？困惑就对了，所以我们需要提供外部方法来给外部传给歌词控件歌词集合和播放器。

```java
    //将歌词集合传给到这个自定义View中
    public LrcView setLrc(String lrc) {
        lrcBeanList = LrcUtil.parseStr2List(lrc);
        return this;
    }

    //传递mediaPlayer给自定义View中
    public LrcView setPlayer(MediaPlayer player) {
        this.player = player;
        return this;
    }
```

外部方法中setLrc的参数必须是前面提到的标准歌词格式的字符串形式，这样我们就能利用上文的解析工具类LrcUtil中的解析方法将字符串解析成歌词集合。

**3. 画歌词**

```java
    private void drawLrc(Canvas canvas) {
        for (int i = 0; i < lrcBeanList.size(); i++) {
            if (currentPosition == i) {//如果是当前的歌词就用高亮的画笔画
                canvas.drawText(lrcBeanList.get(i).getLrc(), width / 2, height / 2 + i * lineSpacing, hPaint);
            } else {
                canvas.drawText(lrcBeanList.get(i).getLrc(), width / 2, height / 2 + i * lineSpacing, dPaint);
            }
        }
    }
```

知道了当前歌词的位置就很容易画歌词了。遍历歌词集合，如果是当前歌词，则用高亮的画笔画，其它歌词就用普通画笔画。这里需注意的是两支画笔画的位置公式都是一样的，坐标位置为x=宽的一半,y=高的一半+当前位置*行间距。随着当前位置的变化，就能画出上下句歌词来。所以其实绘制出来后你会发现歌词是从控件的正中央开始绘制的，这是为了方便与下面歌词同步滑动功能配合。

**4. 歌词同步滑动**

```java
    //歌词滑动
    private void scrollLrc() {
        //下一句歌词的开始时间
        long startTime = lrcBeanList.get(currentPosition).getStart();
        long currentTime = player.getCurrentPosition();

        //判断是否换行,在0.5内完成滑动，即实现弹性滑动
        float y = (currentTime - startTime) > 500 ? currentPosition * lineSpacing : lastPosition * lineSpacing + (currentPosition - lastPosition) * lineSpacing * ((currentTime - startTime) / 500f);
        scrollTo(0,(int)y);
        if (getScrollY() == currentPosition * lineSpacing) {
            lastPosition = currentPosition;
        }
    }
```

如果不实现弹性滑动的话，只要判断当前播放歌曲的时间是否大于当前位置歌词的结束时间，然后进行scrollTo(0,(int)currentPosition * lineSpacing)滑动即可。但是为了实现弹性滑动，我们需要将一次滑动分成若干次小的滑动并在一个时间段内完成，所以我们动态设置y的值，由于不断重绘，就能实现在0.5秒内完成View的滑动，这样就能实现歌词同步弹性滑动。

> 500其实就是0.5s,因为在这里currentTime和startTime的单位都是ms

```java
        float y = (currentTime - startTime) > 500 ? currentPosition * lineSpacing : lastPosition * lineSpacing + (currentPosition - lastPosition) * lineSpacing * ((currentTime - startTime) / 500f);
```

**5.不断重绘**

通过不断重绘才能实现歌词同步滑动，这里每隔0.1s进行重绘

```java
postInvalidateDelayed(100);//延迟0.1s刷新
```

你以为这样就结束了吗？其实还没有，答案下文揭晓！


# 三 、使用

然后我们兴高采烈的在xml中，引用这个自定义View

> LrcView前面的名称为你建这个类的完整包名

```java
    <com.example.library.view.LrcView
        android:id="@+id/lrcView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:lineSpacing="40dp"
        app:textSize="18sp"
        app:lrcTextColor="@color/colorPrimary"
        app:highLineTextColor="@color/highTextColor"
        />
```

在Java代码中给这个自定义View传入标准歌词字符串和播放器。

```java
lrcView.setLrc(lrc).setPlayer(player);

```

点击运行，满心期待自己的成果，接着你就会一脸懵逼，what?怎么是一片空白，什么也没有！其实这时候你重新理一下上面歌词绘制的流程，就会发现问题所在。**首先我们的自定义View控件引用到布局中时是先执行onDraw方法的，所以当你调用setLrc和setPlayer方法后，是不会再重新调用onDraw方法的，等于你并没有传入歌词字符串和播放器，所以当然会显示一片空白**

**解决方法**:我们在刚才自定义View歌词控件中添加一个外部方法来调用onDraw,刚好这个invalidate()就能够重新调用onDraw方法

```java
    public LrcView draw() {
        currentPosition = 0;
        lastPosition = 0;
        invalidate();
        return this;
    }

```

然后我们在主代码中，在调用setLrc和setPlayer后还得调用draw方法

```java
lrcView.setLrc(lrc).setPlayer(player).draw();

```

这样我们节约风的歌词控件就大功告成了。

# 相关源码地址

如果觉得不错的话，欢迎大家来star!

### [歌词控件源码地址](https://github.com/jsyjst/Yuan-LrcView)(点进去有添加该依赖库说明)

### [音乐播放器源码地址](https://github.com/jsyjst/Yuan-SxMusic)



