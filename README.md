# 今日校园自动签到/守护进程/定时自动唤醒/自动解锁
本人是长江师范的大三狗，闲来无事，就对今日校园下了狠手，以下内容仅供学习，切勿用于商业用途。

## 本项目由AccessibilityService服务开发实现了一下几大功能： 
### 自动签到： 
功能主要通过AccessibilityService服务进行自动化操作， 
每个同学的签到过程可能不一，有些同学可能需要拍照或者点击其他按钮，可以在源代码中适当修改一下。 
### 保活进程/定时拉活 
主要采用的BroadcastReceiver以及AlarmManager搭配定时广播保活方式守护进程， 
做这个保活进程的很大一部分原因是因为，高版本的Android系统采用了进一步的优化省电策略， 
用户在一段时间没有操作后，进程会被自动杀死，为了保证我们的app能够顺利得进行自动化签到， 
就必须要进行进程保活。这个主要是到达指定的精确时间，自动拉活我们的程序，小米/红米等部分机型 
需要手动开启应用的【后台弹出界面】权限 
###  定时自动唤醒
这个功能不用多解释，主要是AlarmManager定时服务来唤醒手机，
android8.0及以上版本在对手机息屏后都CPU都会进入休眠状态，
所有的线程、广播、服务都会停止。
这一块功能主要解决了AlarmManager定时不准确的问题，能适应手机相应的版本
### 自动解锁 
这一块功能相当费劲，原本我想通过让app自动解锁打开第三方软件的，
但是手机在有密码锁或者指纹锁等锁的权限时，息屏无法打开第三方(今日校园)的软件，
只能打开本软件的activity，主要是因为android的安全性，无法完成这一些操作；
最后实验无果，换了种思路就是手机在没有密码锁或者其他锁的时候,
但是在这种情况(没有密码锁的时候)手机依然是需要上滑解锁的，
为了解决这个问题我想了很多的办法，
比如用adb shell命令，或者用AccessibilityService服务的GestureDescription来实现锁屏时上滑解锁，
其实这里有很大的坑，就是adb shell命令无法在手机息屏时执行，需要root权限，用户体验太差，就放弃了这种思路；
利用GestureDescription手势解锁也有特别大的坑，因为上滑解锁的要求是非常高的。
因为谷歌考虑到放在包包里时上滑解锁可能会误触解锁，所以手势解锁解锁必须要有一个加速度，
也就是说滑动的时候要保持一个加速度的问题，但是GestureDescription无法设置加速度，换一种角度来说就是太麻烦了；
所以我放弃了这种思路，程序员是干什么吃的？就是要有钻研精神，就算翻互联网一个底朝天我也要弄出来！
最后我又去CSDN和Android的源码研究了一下解锁过程；
功夫不负有心人，正当我快被这个崩溃时发现了KeyguardManager这个宝藏，来看以下代码：
```
KeyguardManager km= (KeyguardManager) context.getSystemService(KEYGUARD_SERVICE);
KeyguardManager.KeyguardLock kl = km.newKeyguardLock("unLock");
//解锁
kl.disableKeyguard();
```
kl.disableKeyguard()的方法我去android API了解了一下，
他其实并不是解锁屏幕，而是把锁屏功能给禁用了。
锁屏界面其实就是一个activity
CSDN很多博主都说disableKeyguard()是解锁方法，其实这个是大错特错！
通过disableKeyguard()我们就可以实现在锁屏中显示我们的app（需要打开app的【锁屏显示】权限），
但是disableKeyguard()有个很严重的问题就是，使用此方法会禁用手机home键、菜单键、以及返回键。
为什么呢？因为谷歌担心你在使用此方法禁用锁屏界面时，绕过锁屏去触犯用户的隐私，
所以上滑解锁其实也阻碍了很多APP绕过解锁去执行一些违规操作，
但是他万万没想到我是谁？长师界的Jon Skeet(国外很火的一个编程大神)，我尽然
我们让app自动唤醒手机以后解锁



km.newKeyguardLock("unLock");这一段代码也被android废弃了，我们先不管这一行代码。


