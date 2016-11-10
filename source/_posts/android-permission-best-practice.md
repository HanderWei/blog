---
layout: post
title: Android M 权限最佳实践
subtitle: " 详述Android 6.0+ 权限申请相关的处理策略 "
date: 2016-11-10 00:43:14
author: "Hander"
header-img: "permission-header.jpg"
tags:
	- Android
---

> 万物基于MIUI.

<!-- toc -->

## 前言

Google在Android 6.0 上开始原生支持应用权限管理，再不是安装应用时的一刀切。权限管理虽然很大程度上增加了用户的可操作性，但是却苦了广大Android开发者。由于权限管理涉及到应用的各个方面，为了避免背锅，很多大厂App的`targetSdkVersion`仍然停留在22。现在Android 7.0 已经发布，及早的解决该问题才能避免问题越滚越大。:neutral_face::neutral_face::neutral_face:

## 权限分类

Google将Android的权限分为三大类：

* 普通权限(Normal Permissions)
* 危险权限(Dangerous Permissions)
* 特殊权限(Special Permissions)

### 普通权限(Normal Permissions)

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

普通权限不会对用户的隐私和安全产生太大的风险，所以只需要在**AndroidManifest.xml**中声明即可.

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

危险权限基本都涉及到用户的隐私，诸如拍照、读取短信、写存储、录音等。(注：涉及隐私的就是危险权限)

Android系统将这些危险权限分组，获取分组中某个权限的同时也就获取了同组中的其他权限。例如，在应用中申请`READ_EXTERNAL_STORAGE`权限，用户同意授权后，则应用同时具有`READ_EXTERNAL_STORAGE` 和 `WRITE_EXTERNAL_STORAGE` 权限。

危险权限不仅需要在**AndroidManifest.xml**中注册，还需要动态的申请权限。

下图为某信申请的权限(九组权限，申请了八组，除了日历...:fearful::fearful::fearful:)

<img src="http://7xs83t.com1.z0.glb.clouddn.com/%E5%BE%AE%E4%BF%A1%E6%9D%83%E9%99%90.png" width="540" height="960" />

### 特殊权限(Special Permissions)

| Special Permissions |
| --- |
| SYSTEM_ALERT_WINDOW 设置悬浮窗 |
| WRITE_SETTINGS 修改系统设置 |

看权限名就知道*特殊权限*比*危险权限更*危险，所以需要在manifest中申请**并且**通过发送Intent让用户在设置界面进行勾选.

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

三个条件缺一不可。

如果项目的`targetSdkVersion < 23`, 在Android 6.0＋的机子上，会默认给予所有在AndroidManifest.xml中申请的权限。

是不是觉得这样就完事大吉了？

<img src="http://7xs83t.com1.z0.glb.clouddn.com/too-young.jpg">

如果用户手动在应用的权限页面收回权限，必然会导致应用的Crash.:broken_heart:

稳妥的处理当然是遵循Google的权限申请机制。

## 权限申请的一般流程

### API

| API | 作用 |
| --- | --- |
| checkSelfPermission( ) | 判断权限是否具有某项权限 |
| requestPermissions( ) | 申请权限 |
| onRequestPermissionsResult( ) | 申请权限回调方法 |
| shouldShowRequestPermissionRationale( ) | 是否要提示用户申请该权限的缘由 |

### shouldShowRequestPermissionRationale()分析

| 序号 | 用户是否授予权限 | shouldShowRationale( ) 返回 | 是否勾选“不再询问” |
| :---: | :---: | :---: | :---: |
| 1 | 否 | false| - |
| 2 | 否 | true | 否 |
| 3 | 否 | true | 否 |
| ... | ... | ... |... |
| i | 否 | true | 是 |
| i + 1 | - | false | - |

> shouldShowRequestPermissionRationale( )方法名太长，在表格中简写

第i次用户勾选了“不再询问”，同时也没有给予应用权限，则第i + 1次应用将无法唤起请求权限的Dialog，只能引导用户进入设置界面，手动勾选所需权限。

### 如果判断用户勾选了“不再询问”？




