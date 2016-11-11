---
layout: post
title: Android M 权限最佳实践
subtitle: " 详述Android 6.0+ 权限申请相关的处理策略 "
date: 2016-11-10 00:43:14
author: "Hander"
header-img: "permission-header.jpg"
tags:
    - Android
    - Permission
---

<!-- toc -->

> 万物基于MIUI.

## 前言

Google在Android 6.0 上开始原生支持应用权限管理，再不是安装应用时的一刀切。权限管理虽然很大程度上增加了用户的可操作性，但是却苦了广大Android开发者。由于权限管理涉及到应用的各个方面，为了避免背锅，很多大厂App的`targetSdkVersion`仍然停留在22。

现在Android 7.0 已经发布，是时候收拾这个烂摊子了:neutral_face::neutral_face::neutral_face:

<img src="http://7xs83t.com1.z0.glb.clouddn.com/android-permission.jpg"/>

## 权限分类

Android的权限分为三类：

* 普通权限(Normal Permissions)
* 危险权限(Dangerous Permissions)
* 特殊权限(Special Permissions)

### 普通权限(Normal Permissions)

普通权限不会对用户的隐私和安全产生太大的风险，所以只需要在**AndroidManifest.xml**中声明即可.

[普通权限对照表](#normal_permission_table)

### 危险权限(Dangerous Permissions)

| Permission Group | Permissions |
| --- | --- |
| CALENDAR | READ_CALENDAR <br> WRITE_CALENDAR |
| CAMERA | CAMERA |
| CONTACTS | READ_CONTACTS <br> WRITE_CONTACTS <br> GET_ACCOUNTS |
| LOCATION | ACCESS_FINE_LOCATION <br> ACCESS_COARSE_LOCATION |
| MICROPHONE | RECORD_AUDIO |
| PHONE	| READ_PHONE_STATE <br> CALL_PHONE <br> READ_CALL_LOG <br> WRITE_CALL_LOG <br> ADD_VOICEMAIL <br> USE_SIP <br> PROCESS_OUTGOING_CALLS |
| SENSORS | BODY_SENSORS |
| SMS | SEND_SMS <br> RECEIVE_SMS <br> READ_SMS <br> RECEIVE_WAP_PUSH <br> RECEIVE_MMS |
| STORAGE | READ_EXTERNAL_STORAGE <br> WRITE_EXTERNAL_STORAGE |

危险权限基本都涉及到用户的隐私，诸如拍照、读取短信、写存储、录音等。

> 便于记忆：涉及隐私的就是危险权限

Android系统将这些危险权限分为9组，获取分组中某个权限的同时也就获取了同组中的其他权限。

例如，在应用中申请`READ_EXTERNAL_STORAGE`权限，用户同意授权后，则应用同时具有`READ_EXTERNAL_STORAGE` 和 `WRITE_EXTERNAL_STORAGE` 权限。

危险权限不仅需要在**AndroidManifest.xml**中注册，还需要动态的申请权限。

下图为某信申请的权限( 九组权限，申请了八组，除了日历...:fearful::fearful::fearful: )

<img src="http://7xs83t.com1.z0.glb.clouddn.com/%E5%BE%AE%E4%BF%A1%E6%9D%83%E9%99%90.png" width="540" height="960" />

### 特殊权限(Special Permissions)

| Special Permissions |
| --- |
| SYSTEM_ALERT_WINDOW 设置悬浮窗 |
| WRITE_SETTINGS 修改系统设置 |

看权限名就知道**特殊权限**比**危险权限**更危险，`特殊权限`需要在manifest中申请**并且**通过发送Intent让用户在设置界面进行勾选.

#### 申请SYSTEM_ALERT_WINDOW权限

{% codeblock lang:java %}
private static final int REQUEST_CODE = 1;
private  void requestAlertWindowPermission() {
    Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
    intent.setData(Uri.parse("package:" + getPackageName()));
    startActivityForResult(intent, REQUEST_CODE);
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode == REQUEST_CODE) {
        if (Settings.canDrawOverlays(this)) {
            Log.i(LOGTAG, "onActivityResult granted");
        }
    }
}
{% endcodeblock %}

#### 申请WRITE_SETTINGS权限
    
{% codeblock lang:java %}
private static final int REQUEST_CODE_WRITE_SETTINGS = 2;
private void requestWriteSettings() {
    Intent intent = new Intent(Settings.ACTION_MANAGE_WRITE_SETTINGS);
    intent.setData(Uri.parse("package:" + getPackageName()));
    startActivityForResult(intent, REQUEST_CODE_WRITE_SETTINGS );
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    if (requestCode == REQUEST_CODE_WRITE_SETTINGS) {
        if (Settings.System.canWrite(this)) {
            Log.i(LOGTAG, "onActivityResult write settings granted" );
        }
    }
}
{% endcodeblock %}

## 何时需要动态申请权限？

> 1. 危险权限
> 2. Android 版本 >= 6.0
> 3. targetSdkVersion >= 23

三个条件缺一不可

如果项目的`targetSdkVersion < 23`, 在Android 6.0＋的手机上，会默认给予所有在AndroidManifest.xml中申请的权限。

是不是觉得这样就完事大吉了？

<img src="http://7xs83t.com1.z0.glb.clouddn.com/too-young.jpg">

如果用户在应用的权限页面手动收回权限，将会导致应用Crash.:broken_heart:

稳妥的处理当然是遵循Google的权限申请机制。

## 权限申请的一般流程

### API

为方便开发者实现权限管理，Google提供了4个API: 

| API | 作用 |
| --- | --- |
| checkSelfPermission( ) | 判断权限是否具有某项权限 |
| requestPermissions( ) | 申请权限 |
| onRequestPermissionsResult( ) | 申请权限回调方法 |
| shouldShowRequestPermissionRationale( ) | 是否要提示用户申请该权限的缘由 |

### 申请权限

以发送短信为例

1. 在AndroidManifest.xml中声明权限

{% codeblock lang:xml %}
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.SEND_SMS"/>
    
    <application ... >
        ...
    </application>
</manifest>
{% endcodeblock %}

2. 判断是否已经获取该权限，若未获取权限，则申请权限

{% codeblock lang:java %}
int permissionCheck = ContextCompat.checkSelfPermission(thisActivity,
        Manifest.permission.SEND_SMS);
if (permissionCheck == PackageManager.PERMISSION_GRANTED) {
    // 发送短信
    ... ...
} else {
    // 申请权限
    ActivityCompat.requestPermissions(thisActivity,
                new String[]{Manifest.permission.SEND_SMS},
                PERMISSIONS_REQUEST_SEND_SMS);
}
{% endcodeblock %}

3. 接收授权回调

{% codeblock lang:java %}
@Override
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) {
    switch (requestCode) {
        case PERMISSIONS_REQUEST_SEND_SMS: {
            if (grantResults.length > 0
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {

                // 已授予权限
                doSomething();

            } else {
                // 申请权限被拒
                Toast.show("...");
            }
            return;
        }
    }
}
{% endcodeblock %}

### 流程图

<img src="http://7xs83t.com1.z0.glb.clouddn.com/Android%20M%20%E7%94%B3%E8%AF%B7%E6%9D%83%E9%99%90%E4%B8%80%E8%88%AC%E6%B5%81%E7%A8%8B.png" />

所谓权限申请就这么简单？？？EXO ME？？？

<img src="http://7xs83t.com1.z0.glb.clouddn.com/question.jpg" />

进度条暴露了一切，事情并没有这么简单。

如果用户任性的勾选了“不再询问”，那么在执行`requestPermissions( )`后，`onRequestPermissionsResult( )`会永远返回`PERMISSION_DENIED`，这样应用原本的操作将永远无法执行。

<img src="http://7xs83t.com1.z0.glb.clouddn.com/permission-not-ask-again.png" width="540" height="960"/>

## 权限申请的正确姿势

上文有提到Google提供了4个新的API，还有一个`shouldShowRequestPermissionRationale( )`方法没有用到。

### shouldShowRequestPermissionRationale( )

| Returns | Explain |
| --- | --- |
| boolean | 是否应该提示用户申请该权限的缘由 | 

如果返回为`true`，一般情况下，应用应该弹出Dialog说明申请该权限的缘由

<img src="http://7xs83t.com1.z0.glb.clouddn.com/rationale.png" width="540" height="960" />

当第一次申请权限时，`shouldShowRequestPermissionRationale( )`会返回`false`，意味着第一次不需要告知用户申请该权限的理由。

如果第一次申请权限被拒，再次申请时，`shouldShowRequestPermissionRationale( )`会返回`true`，也就是说用户之前拒绝了该权限的授予，此时应该告知用户应用为什么需要该权限。

注意，此时系统弹出的Dialog会有一个checkbox选项，提示是否`不再询问`！！！

如果此时，用户勾选了“不再询问”，再次调用“shouldShowRequestPermissionRationale( )”会返回`false`。

综上，`shouldShowRequestPermissionRationale( )`会在两种情况下返回`false`，两次的含义并不相同。

> 1. 第一次申请权限
> 2. 用户拒绝申请权限，且勾选了“不再询问”

而`shouldShowRequestPermissionRationale( )`只会在一种情况下返回`true`

> 用户上一次拒绝申请权限，但是并未勾选“不再询问”

下表举例说明了`shouldShowRequestPermissionRationale( )`的返回

| 序号 | 用户是否授予权限 | shouldShowRationale( ) 返回 | 是否勾选“不再询问” |
| :---: | :---: | :---: | :---: |
| 1 | 否 | false| - |
| 2 | 否 | true | 否 |
| 3 | 否 | true | 否 |
| ... | ... | ... |... |
| i | 否 | true | 是 |
| i + 1 | - | false | - |

> shouldShowRequestPermissionRationale( )方法名太长，在表格中简写

第i次用户勾选了“不再询问”，同时也没有给予应用权限，则第i + 1次应用将无法唤起请求权限的Dialog，**只能**引导用户进入设置界面，手动勾选所需权限。

### 如何判断用户勾选了“不再询问”？

从上面的表格可以看出，如果上次`shouldShowRequestPermissionRationale( )`返回了`true`，而这次调用该方法返回了`false`，则说明用户在上次勾选了“不再询问”。此时，我们需要引导用户进入设置界面进行权限授予。

由于涉及到上一次调用`shouldShowRequestPermissionRationale( )`的结果，所以需要将其持久化保存，`SharedPreferences`或者数据库均可。


### 正确姿势

{% codeblock lang:java %}
private void requestPermission(Activity activity, final String permission) {
    boolean flag = ActivityCompat.shouldShowRequestPermissionRationale(activity, permission);
    if (getLastRequestState() && !flag) {
        //当用户勾选`不再询问`时, 进入设置界面
        Uri uri = Uri.fromParts("package", this.getPackageName(), null);
        Intent intent = new Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS, uri);
        startActivityForResult(intent, COME_CODE);
    } else if (flag) {
        //之前有过`拒绝`授权时，提醒用户需要某权限
        showRationaleDialog();
        
        //同时保存返回值
        SharedPrefsUtils.setBooleanPreference(getApplicationContext(), KEY_RESUEST_SOME_PERMISSION, flag);
    } else {
        //第一次申请权限时，直接申请权限
        ActivityCompat.requestPermissions(activity, new String[]{permission}, REQUEST_PERMISSION_CODE);
    }
}
{% endcodeblock %}

### 流程图

<img src="http://7xs83t.com1.z0.glb.clouddn.com/Android%20M%20%E6%9D%83%E9%99%90%E7%94%B3%E8%AF%B7%28%E9%9C%80%E4%BF%9D%E5%AD%98%E7%8A%B6%E6%80%81%29.png"/>

## 最佳实践

上面的解决方案是可行的，但是每次申请权限需要依赖于上一次调用`shouldShowRequestPermissionRationale( )`方法的返回值，如果SharedPreferences被修改或者被删除，会影响正常的申请流程。

Google提供了一个非常好的思路，详见[EasyPermissions  ](https://github.com/googlesamples/easypermissions).

EasyPermissions并没有存储上一次`shouldShowRequestPermissionRationale( )`的返回值，而是在申请权限被拒后调用`shouldShowRequestPermissionRationale( )`方法，如果此时返回`false`则说明用户勾选了“不再询问”。

| 序号 | 用户是否授予权限 | shouldShowRationale( ) 返回 | 是否勾选“不再询问” | 再次调用shouldShowRationale( )返回 |
| :---: | :---: | :---: | :---: | :---: |
| 1 | 否 | false| - | true |
| 2 | 否 | true | 否 | true |
| 3 | 否 | true | 否 | true |
| ... | ... | ... |... | ... |
| i | 否 | true | 是 | false |
| i + 1 | - | false | - | - |

### 简化判断“不再询问”的条件
> 1. 未获得授权
> 2. shouldShowRequestPermissionRationale( )返回false

### 流程图

<img src="http://7xs83t.com1.z0.glb.clouddn.com/Android%20M%20%E6%9D%83%E9%99%90%E7%94%B3%E8%AF%B7%28%E6%97%A0%E9%9C%80%E4%BF%9D%E5%AD%98%E7%8A%B6%E6%80%81%29.png"/>

## 还能再优化吗？

拜读了EasyPermissions后，我做了一些微小的工作，简单的封装可以减少很多样板代码。

具体见[PermissionBestPractice](https://github.com/HanderWei/PermissionBestPractice)

> 将通用的操作转移到`BaseActivity`和`BaseFragment`中

每个Activity或者Fragment都需要覆写`onRequestPermissionsResult( )`方法，这部分可以统一放到`BaseActivity`和`BaseFragment`中

{% codeblock lang:java %}
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    PermissionUtils.onRequestPermissionsResult(requestCode, permissions, grantResults, this);
}
{% endcodeblock %}

另外权限授权和拒绝也可以在基类里统一处理
{% codeblock lang:java %}
@Override
public void onPermissionGranted(int requestCode, List<String> perms) {
    Log.d(TAG, perms.size() + " permissions granted.");
}

@Override
public void onPermissionDenied(int requestCode, List<String> perms) {
    Log.e(TAG, perms.size() + " permissions denied.");
    if (PermissionUtils.somePermissionsPermanentlyDenied(this, perms)) {
        // 勾选了“不再询问”，进入应用设置界面
        magic code ...
    }
}
{% endcodeblock %}

这样，在Activity或者Fragment只需做很小的修改就可以实现6.0上的权限管理了

{% codeblock lang:java %}
// 1. 定义Request Code
private static final int REQUEST_CAMERA_PERMISSION = 0x01;

// 某项操作需要Camera权限
public void doSomethingNeedCamera(View view) {
    // 2. 判断是否具有该权限
    if (PermissionUtils.hasPermisssions(this, Manifest.permission.CAMERA)) {
        openCamera();
    } else {
        // 3. 如果没有权限，则申请权限
        PermissionUtils.requestPermissions(this, getString(R.string.rationale_camera), REQUEST_CAMERA_PERMISSION, Manifest.permission.CAMERA);
    }
}

// 4. 为执行操作添加注解
@AfterPermissionGranted(REQUEST_CAMERA_PERMISSION)
private void openCamera() {
    // 唤起照相机代码...
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    if (intent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(intent, REQUEST_OPEN_CAMERA);
    }
}
{% endcodeblock %}

如果某项操作需要多个权限？

{% codeblock lang:java %}
// 1. 定义Request Code
private static final int REQUEST_CALENDAR_AND_CONTACTS = 0x02;

// 某项操作需要多个权限
public void needTwoPermissions(View view) {
    String[] perms = new String[]{Manifest.permission.READ_CALENDAR, Manifest.permission.READ_CONTACTS};
    
    // 2. 判断是否具有这些权限
    if (PermissionUtils.hasPermisssions(this, perms)) {
        twoPermissionsGranted();
    } else {
        // 3. 如果没有权限，则申请权限
        PermissionUtils.requestPermissions(this, getString(R.string.rationale_calendar_and_contacts), REQUEST_CALENDAR_AND_CONTACTS, perms);
    }
}

// 4. 为执行操作添加注解
@AfterPermissionGranted(REQUEST_CALENDAR_AND_CONTACTS)
private void twoPermissionsGranted() {
    Toast.makeText(this, "授权成功", Toast.LENGTH_SHORT).show();
}
{% endcodeblock %}

## 附表
<p id = "normal_permission_table"></p>

### 普通权限

| Normal Permissions |
| ------ |
| ACCESS_LOCATION_EXTRA_COMMANDS |
| ACCESS_NETWORK_STATE |
| ACCESS_NOTIFICATION_POLICY |
| ACCESS_WIFI_STATE |
| BLUETOOTH |
| BLUETOOTH_ADMIN |
| BROADCAST_STICKY |
| CHANGE_NETWORK_STATE |
| CHANGE_WIFI_MULTICAST_STATE |
| CHANGE_WIFI_STATE |
| DISABLE_KEYGUARD |
| EXPAND_STATUS_BAR |
| GET_PACKAGE_SIZE |
| INSTALL_SHORTCUT |
| INTERNET |
| KILL_BACKGROUND_PROCESSES |
| MODIFY_AUDIO_SETTINGS |
| NFC |
| READ_SYNC_SETTINGS |
| READ_SYNC_STATS |
| RECEIVE_BOOT_COMPLETED |
| REORDER_TASKS |
| REQUEST_IGNORE_BATTERY_OPTIMIZATIONS |
| REQUEST_INSTALL_PACKAGES |
| SET_ALARM |
| SET_TIME_ZONE |
| SET_WALLPAPER |
| SET_WALLPAPER_HINTS |
| TRANSMIT_IR |
| UNINSTALL_SHORTCUT |
| USE_FINGERPRINT |
| VIBRATE |
| WAKE_LOCK |
| WRITE_SYNC_SETTINGS |

## Demo地址

[https://github.com/HanderWei/PermissionBestPractice](https://github.com/HanderWei/PermissionBestPractice)

## 参考文档

[Requesting Permissions at Run Time](https://developer.android.com/training/permissions/requesting.html)

[EasyPermissions](https://github.com/googlesamples/easypermissions)

[Android M 运行时权限实践全解析](http://www.jianshu.com/p/33a31c967d5e)

[聊一聊 Android 6.0 的运行时权限](https://blog.coding.net/blog/understanding-marshmallow-runtime-permission)
