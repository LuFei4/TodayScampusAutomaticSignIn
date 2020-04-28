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
##### 出现的问题
这一块功能相当费劲，原本我想通过让app自动解锁打开第三方软件的，  
但是手机在有密码锁或者指纹锁等锁的权限时，息屏无法打开第三方(今日校园)的软件，  
只能打开本软件的activity，主要是因为android的安全性，无法完成这一些操作；  
最后实验无果，换了种思路就是手机在没有密码锁或者其他锁的时候,  
但是在这种情况(没有密码锁的时候)手机依然是需要上滑解锁的，  
为了解决这个问题我想了很多的办法。  
比如用adb shell命令，
或者用AccessibilityService服务的GestureDescription来实现锁屏时上滑解锁，  
其实这里有很大的坑，就是adb shell命令无法在手机息屏时执行，
需要root权限，用户体验太差，就放弃了这种思路；  
利用GestureDescription手势解锁也有特别大的坑，因为上滑解锁的要求是非常高的。  
因为谷歌考虑到放在包包里时上滑解锁可能会误触解锁，所以手势解锁解锁必须要有一个加速度，  
也就是说滑动的时候要保持一个加速度的问题，但是GestureDescription无法设置加速度，换一种角度来说就是太麻烦了；  
所以我放弃了这种思路，程序员是干什么吃的？就是要有钻研精神，就算翻互联网一个底朝天我也要弄出来！  
最后我又去CSDN和Android的源码研究了一下解锁过程。  

##### 解决的过程
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
通过disableKeyguard()我们就可以实现在锁屏中显示我们的app，  
但是disableKeyguard()有个很严重的问题就是，  
使用此方法会禁用手机home键、菜单键、以及返回键。  
为什么呢？因为谷歌担心你在使用此方法禁用锁屏界面时，绕过锁屏去触犯用户的隐私，  
所以上滑解锁其实也阻碍了很多APP绕过解锁去执行一些违规操作，  
但是他万万没想到我是谁？长师界的Jon Skeet(国外很火的一个编程大神).
不好意思ヽ(≧□≦)ノ我这个人太自恋了。。。

##### 我竟然发现了android一个奇怪的漏洞：  
我们让app通过BroadcastReceiver自动唤醒手机以后禁用解锁，
然后拉活我的app软件加载到主界面中，  
主界面跳转到第三方今日校园app，  
今日校园完成一系列自动签到后是无法返回到home主界面的，  
因为home键是被disableKeyguard()方法给禁用了，
怎么办了？我又是翻遍了整个互联网，  
正当千钧一发之际，我脑袋里突然来了灵感，  
AccessibilityService不是有那个模拟的全局home键吗？  
他可不可以绕过disableKeyguard()去打开home呢？  
说时迟那时快，吭吭吭代码就已经打上去了。  
```
AccessibilityService的GLOBAL_ACTION_HOME  
```
这个方法竟然能真的绕过disableKeyguard()去模拟home并回到主界面！  
最神奇的还不是这个，第二天我闹钟响了，因为闹钟是在锁屏时显示的，  
当时我睡意朦胧打开手机尽然发现锁屏闹钟不见了，只有锁屏界面和闹钟的声音，  
等我解锁后，我发现没有地方可以关闭闹钟！？  
这意味这什么？意味着我们现在是盗梦空间的梦中梦，  
意思就是我们这个锁屏之外还有一个锁屏，  
然后我再通知栏找到了闹钟的提醒，我单击下去，竟然又回到了第一个锁屏界面！  
这太有意思了！那么我们返回来想一想，那我们之前那个锁屏又去哪里了呢？  
那么我们在拓展一下思维，去做一些....
这个技术不可言喻，免得被你们拿去做羞羞的事情，
这个谜就等你们自己去探索吧，哈哈！  

### 手机需要的权限  
无障碍服务  
锁屏显示  
后台弹出界面【这个权限貌似只有小米和红米有】  

### 可能出现的问题  
这个软件可能因为不同品牌导致部分功能失效，  
其实我在JumpPermissionManagement这个类里面已经考虑了不同手机品牌，  
但是还是有可能出现很多问题：比如某些手机的品牌不同系统可能还是不能兼容。  
这个问题就需要程序员自己解决了，我只是提供一个大致的思路而已。  
不懂得可以提问，我抽时间解答。  
另外这个项目不定时更新，也可能会无限延期更新。  
喜欢弄的老哥fork一下吧  

