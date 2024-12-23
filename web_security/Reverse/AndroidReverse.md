# 一、安卓App

​		在AndroidStudio项目中，`AndroidManifest.xml`文件非常重要，一般位于`app > src > main > AndroidManifest.xml`的目录下，在其中声明了核心配置，包括名字、权限、组件等。在安卓系统中，通信主要采用`intent`机制，即发送一个`intent`，其中包含了要发送的信息。

## 1、四大组件

### 1.1 广播

​		广播是一种回调函数，在接收到期望的信息以后，执行相关动作

#### 设置开机自启动

​		在`manifest.xml`文件中申请权限

```xml
<manifest>
    <!-- 申请接收开机广播 -->
	<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
</manifest>
```

​		在安卓系统开机以后，等启动完成，会发送一个全局广播`intent`，其中的`action`字段设置为`"android.intent.action.BOOT_COMPLETED"`，我们设置完接收开机广播以后，需要在`manifest.xml`中注册我们的广播接收器

```xml
<application>
    <receiver
    	android:name="(接收器的全类名如)com.example.BroadcastReceiver"
        android:enable="true"
        android:exported="true"
		<intent-filter>
    		<action android:name="android.intent.action.BOOT_COMPLETED"/>
    		<category android:name="android.intent.category.DEFAULT"/>
		</intent-filter>
</application>
```

```java
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

// 所有的接收器都要继承自抽象类BraodcastReceiver
public class BootReceiver extends BroadcastReceiver {
    private static final String ACTION = "android.intent.action.BOOT_COMPLETED";

    @Override
    // 继承抽象类以后需要重写抽象方法onReceive
    public void onReceive (Context context, Intent intent){
        // 如果是开机广播，则启动
        if (ACTION.equals(intent.getAction())) {
            /* 这里是你启动自己的代码 */
        }
    }
}
```

### 1.2 服务

​		服务有两种，一种是

# 二、安卓逆向

组合拳

​	反编译工具

​		JEB、JADX

​	插桩分析工具

​		Frida

​		Xposed

​	网络流量分析

​		BurpSuite/ Fiddler/ Charles (for Mac)

​		Wireshark

​		r0capture

安卓App

​	安卓组件

​			App由各种可以单独调用的组件构成

​			四大组件

​					Activity

​							一个activity通常对应一个单独的UI窗口

​							Activity需要在AndroidManifest.xml文件中进行声明，作为<application>的子元素<activity>

​					Service

​							用于在后台完成用户的指定操作，长时间运行在后台，没有UI界面

​							必须在manifest文件中声明service

​					Broadcast receiver

​							对收到的广播进行过滤和处理，系统或应用会在某些特定事件完成时自动发出广播，即使是没有在运行的app也可以接收到广播

​							两种注册方法，静态注册：Manifest文件，动态注册

​							需要重写onReceive()函数进行对应的处理

​					Content provider

​							用于多个app之间共享数据，通过一个ContentProvider类实现

​							开发人员通过ContentResolver对象实现对ContentProvider操作，使用URI来唯一标识数据集，以 content:// 作为前缀

​							需要在Manifest文件中配置

​	安卓核心配置文件AndroidManifest.xml

​			每个app都有，路径为/app/src/main/AndroidManifest.xml

​			为系统提供了app的基本信息，包名、权限、组件、API Level等 

​	安卓权限机制Android Permission

​			安卓是基于权限管理的系统

​			Permission限制了app对数据和资源的访问，如相机、GPS数据等

​			应用需要在manifest文件中声明所需权限，部分权限还需要向用户动态申请