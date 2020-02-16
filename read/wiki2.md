# 视频优化处理
#### 目录介绍
- 01.播放加载优化
- 02.前后台切换优化
- 03.视频播放异常优化
- 04.视频播放颤动优化
- 05.视频解码优化
- 06.SeekTo设置优化
- 07.关于so库优化
- 08.关于网络状态监听优化
- 09.关于代码规范优化
- 10.关于布局优化
- 11.选择SurfaceView还是TextureView



### 02.前后台切换优化
#### 2.1 第一种优化方式
- 从前台切到后台，当视频正在播放或者正在缓冲时，暂停视频；当从后台切换到前台，当视频暂停时或者缓冲暂停时，自动开启视频播放。比如常见优酷，爱奇艺就类似这样。
    - **如果是在Activity中的话，建议设置下面这段代码**
        ```
        @Override
        protected void onStop() {
            super.onStop();
            //从前台切到后台，当视频正在播放或者正在缓冲时，调用该方法暂停视频
            VideoPlayerManager.instance().suspendVideoPlayer();
        }

        @Override
        protected void onDestroy() {
            super.onDestroy();
            //销毁页面，释放，内部的播放器被释放掉，同时如果在全屏、小窗口模式下都会退出
            VideoPlayerManager.instance().releaseVideoPlayer();
        }

        @Override
        public void onBackPressed() {
            //处理返回键逻辑；如果是全屏，则退出全屏；如果是小窗口，则退出小窗口
            if (VideoPlayerManager.instance().onBackPressed()){
                return;
            }else {
                //销毁页面
                VideoPlayerManager.instance().releaseVideoPlayer();
            }
            super.onBackPressed();
        }

        @Override
        protected void onRestart() {
            super.onRestart();
            //从后台切换到前台，当视频暂停时或者缓冲暂停时，调用该方法重新开启视频播放
            VideoPlayerManager.instance().resumeVideoPlayer();
        }
        ```


#### 2.2 第二种优化方式
- 当视频播放时，切换后台页面处于不可见时，则销毁视频资源。从后台切换到前台时，用户需要手动触发点击才能继续看视频。
    - **如果是在Activity中的话，建议设置下面这段代码**
        ```
        @Override
        protected void onStop() {
            super.onStop();
            VideoPlayerManager.instance().releaseVideoPlayer();
        }

        @Override
        public void onBackPressed() {
            if (VideoPlayerManager.instance().onBackPressed()) return;
            super.onBackPressed();
        }
        ```
    - **如果是在Fragment中的话，建议设置下面这段代码**
        ```
        //在宿主Activity中设置代码如下
        @Override
        protected void onStop() {
            super.onStop();
            VideoPlayerManager.instance().releaseVideoPlayer();
        }

        @Override
        public void onBackPressed() {
            if (VideoPlayerManager.instance().onBackPressed()) return;
            super.onBackPressed();
        }

        //--------------------------------------------------

        //在此Fragment中设置代码如下
        @Override
        public void onStop() {
            super.onStop();
            VideoPlayerManager.instance().releaseVideoPlayer();
        }
        ```



### 05.视频解码优化
- ijkplayer和ffplay在打开rtmp串流视频时，大多数都会遇到5~10秒的延迟，在ffplay播放时，如果加上-fflags nobuffer可以缩短播放的rtmp视频延迟在1s内，而在IjkMediaPlayer中加入
    ```
    mMediaPlayer = new IjkMediaPlayer();
    int PLAYER = IjkMediaPlayer.OPT_CATEGORY_PLAYER;
    int CODEC = IjkMediaPlayer.OPT_CATEGORY_CODEC;
    int FORMAT = IjkMediaPlayer.OPT_CATEGORY_FORMAT;
    //设置播放前的最大探测时间
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "analyzemaxduration", 100L);
    //播放前的探测Size，默认是1M, 改小一点会出画面更快
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "probesize", 10240L);
    //每处理一个packet之后刷新io上下文
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "flush_packets", 1L);
    //是否开启预缓冲，一般直播项目会开启，达到秒开的效果，不过带来了播放丢帧卡顿的体验
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "packet-buffering", 0L);
    //跳帧处理,放CPU处理较慢时，进行跳帧处理，保证播放流程，画面和声音同步
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "framedrop", 1L);
    ```
- jkPlayer支持硬解码和软解码。 软解码时不会旋转视频角度这时需要你通过onInfo的what == IMediaPlayer.MEDIA_INFO_VIDEO_ROTATION_CHANGED去获取角度，自己旋转画面。或者开启硬解硬解码，不过硬解码容易造成黑屏无声（硬件兼容问题），下面是设置硬解码相关的代码
    ```
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "mediacodec", 0);
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "mediacodec-auto-rotate", 1);
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "mediacodec-handle-resolution-change", 1);
    ```
- 其他更多关于解码相关的内容
    ```
    int PLAYER = IjkMediaPlayer.OPT_CATEGORY_PLAYER;
    int CODEC = IjkMediaPlayer.OPT_CATEGORY_CODEC;
    int FORMAT = IjkMediaPlayer.OPT_CATEGORY_FORMAT;

    //设置ijkPlayer播放器的硬件解码相关参数
    //设置播放前的最大探测时间
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "analyzemaxduration", 100L);
    //设置播放前的探测时间 1,达到首屏秒开效果
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "analyzeduration", 1L);
    //播放前的探测Size，默认是1M, 改小一点会出画面更快
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "probesize", 10240L);
    //设置是否开启变调isModifyTone?0:1
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER,"soundtouch",0);
    //每处理一个packet之后刷新io上下文
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "flush_packets", 1L);
    //是否开启预缓冲，一般直播项目会开启，达到秒开的效果，不过带来了播放丢帧卡顿的体验
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "packet-buffering", 0L);
    //播放重连次数
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "reconnect", 5);
    //最大缓冲大小,单位kb
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "max-buffer-size", 10240L);
    //跳帧处理,放CPU处理较慢时，进行跳帧处理，保证播放流程，画面和声音同步
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "framedrop", 1L);
    //最大fps
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "max-fps", 30L);
    //SeekTo设置优化
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "enable-accurate-seek", 1L);
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "opensles", 0);
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "overlay-format", IjkMediaPlayer.SDL_FCC_RV32);
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "framedrop", 1);
    ((IjkMediaPlayer)mMediaPlayer).setOption(PLAYER, "start-on-prepared", 0);
    ((IjkMediaPlayer)mMediaPlayer).setOption(FORMAT, "http-detect-range-support", 0);
    //设置是否开启环路过滤: 0开启，画面质量高，解码开销大，48关闭，画面质量差点，解码开销小
    ((IjkMediaPlayer)mMediaPlayer).setOption(CODEC, "skip_loop_filter", 48);
    ```


### 06.SeekTo设置优化
- 某些视频在SeekTo的时候，会跳回到拖动前的位置，这是因为视频的关键帧的问题，通俗一点就是FFMPEG不兼容，视频压缩过于厉害，seek只支持关键帧，出现这个情况就是原始的视频文件中i 帧比较少
    ```
    mediaPlayer.setOption(IjkMediaPlayer.OPT_CATEGORY_PLAYER, "enable-accurate-seek", 1);
    ```



### 10.关于布局优化
- 关于正式项目中视频可能存在的布局
    - 视频播放器初始化状态布局
    - 视频播放加载状态布局
    - 视频播放器顶部布局[返回键，分享键，下载键，更多键，或者其他控件]
    - 视频播放器底部布局[播放，暂停按钮，视频进度条，切换全屏控件]
    - 视频试看模式状态布局
    - 视频试看结束后状态布局
    - 视频观看结束后状态布局
    - 音视频之间切换效果[状态]
    - 视频播放错误时状态布局
    - 会员权限等布局，非会员时显示布局
    - 全屏播放时的布局
- 建议这样写，不要在一个布局写这么复杂的布局，这里充分体现了include的优势
    ```
    <?xml version="1.0" encoding="utf-8"?>
    <RelativeLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <!--https://github.com/yangchong211-->
        <!--如果你觉得好，请给个star，让更多人使用，避免重复造轮子-->
        <!--底图，主要是显示视频缩略图-->
        <ImageView
            android:id="@+id/image"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:scaleType="fitXY"
            android:visibility="visible"/>
        <!--加载动画view-->
        <include layout="@layout/custom_video_player_loading"/>
        <!--改变播放位置-->
        <include layout="@layout/custom_video_player_change_position"/>
        <!--改变亮度-->
        <include layout="@layout/custom_video_player_change_brightness"/>
        <!--改变声音-->
        <include layout="@layout/custom_video_player_change_volume"/>
        <!--播放完成，你也可以自定义-->
        <include layout="@layout/custom_video_player_completed"/>
        <!--播放错误-->
        <include layout="@layout/custom_video_player_error"/>
        <!--顶部控制区-->
        <include layout="@layout/custom_video_player_top"/>
        <!--底部控制区-->
        <include layout="@layout/custom_video_player_bottom"/>

        //这里省略部分代码
    </RelativeLayout>
    ```



### 11.选择SurfaceView还是TextureView
#### 11.1 SurfaceView优缺点
- 优点：
    - 可以在一个独立的线程中进行绘制，不会影响主线程；使用双缓冲机制，播放视频时画面更流畅
- 缺点：
    - Surface不在View hierachy中，它的显示也不受View的属性控制，所以不能进行平移，缩放等变换，也不能放在其它ViewGroup中。SurfaceView 不能嵌套使用。


#### 11.2 TextureView优缺点
- 优点：
    - 支持移动、旋转、缩放等动画，支持截图。具有view的属性
- 缺点：
    - 必须在硬件加速的窗口中使用，占用内存比SurfaceView高，在5.0以前在主线程渲染，5.0以后有单独的渲染线程。








