---
title: jetBrains报错
date: 2024-06-14 17:07:36
categories:
  - error
tags:
  - tools
  - jetBrains
---

## 前言

jetBrains软件使用中的报错及处理方法汇总。

## Cannot invoke

### 具体报错信息

> 我的报错是出现在clion，jb其他的产品出现同样的报错时，应该也是同样的处理方法（尚未验证）

~~~text
java.lang.RuntimeException: Cannot invoke (class=AppEventListener, method=appClosing, topic=AppLifecycleListener)
	at com.intellij.util.messages.impl.MessageBusImplKt.invokeListener(MessageBusImpl.kt:676)
	at com.intellij.util.messages.impl.MessageBusImplKt.deliverMessage(MessageBusImpl.kt:422)
	at com.intellij.util.messages.impl.MessageBusImplKt.pumpWaiting(MessageBusImpl.kt:401)
	at com.intellij.util.messages.impl.MessageBusImplKt.access$pumpWaiting(MessageBusImpl.kt:1)
	at com.intellij.util.messages.impl.MessagePublisher.invoke(MessageBusImpl.kt:460)
	at jdk.proxy2/jdk.proxy2.$Proxy47.appClosing(Unknown Source)
	at com.intellij.openapi.application.impl.ApplicationImpl.doExit(ApplicationImpl.java:617)
	at com.intellij.openapi.application.impl.ApplicationImpl.exit(ApplicationImpl.java:595)
	at com.intellij.openapi.application.impl.ApplicationImpl.exit(ApplicationImpl.java:584)
	at com.intellij.openapi.application.ex.ApplicationEx.exit(ApplicationEx.java:77)
	at com.intellij.openapi.wm.impl.CloseProjectWindowHelper.quitApp(CloseProjectWindowHelper.kt:67)
	at com.intellij.openapi.wm.impl.CloseProjectWindowHelper.windowClosing(CloseProjectWindowHelper.kt:47)
	at com.intellij.openapi.wm.impl.ProjectFrameHelper.windowClosing(ProjectFrameHelper.kt:439)
	at com.intellij.openapi.wm.impl.WindowCloseListener.windowClosing(ProjectFrameHelper.kt:459)
	at java.desktop/java.awt.AWTEventMulticaster.windowClosing(AWTEventMulticaster.java:357)
	at java.desktop/java.awt.AWTEventMulticaster.windowClosing(AWTEventMulticaster.java:357)
	at java.desktop/java.awt.Window.processWindowEvent(Window.java:2113)
	at java.desktop/javax.swing.JFrame.processWindowEvent(JFrame.java:298)
	at java.desktop/java.awt.Window.processEvent(Window.java:2072)
	at java.desktop/java.awt.Component.dispatchEventImpl(Component.java:5027)
	at java.desktop/java.awt.Container.dispatchEventImpl(Container.java:2324)
	at java.desktop/java.awt.Window.dispatchEventImpl(Window.java:2808)
	at java.desktop/java.awt.Component.dispatchEvent(Component.java:4855)
	at java.desktop/java.awt.EventQueue.dispatchEventImpl(EventQueue.java:794)
	at java.desktop/java.awt.EventQueue$3.run(EventQueue.java:739)
	at java.desktop/java.awt.EventQueue$3.run(EventQueue.java:733)
	at java.base/java.security.AccessController.doPrivileged(AccessController.java:399)
	at java.base/java.security.ProtectionDomain$JavaSecurityAccessImpl.doIntersectionPrivilege(ProtectionDomain.java:86)
	at java.base/java.security.ProtectionDomain$JavaSecurityAccessImpl.doIntersectionPrivilege(ProtectionDomain.java:97)
	at java.desktop/java.awt.EventQueue$4.run(EventQueue.java:766)
	at java.desktop/java.awt.EventQueue$4.run(EventQueue.java:764)
	at java.base/java.security.AccessController.doPrivileged(AccessController.java:399)
	at java.base/java.security.ProtectionDomain$JavaSecurityAccessImpl.doIntersectionPrivilege(ProtectionDomain.java:86)
	at java.desktop/java.awt.EventQueue.dispatchEvent(EventQueue.java:763)
	at com.intellij.ide.IdeEventQueue.defaultDispatchEvent(IdeEventQueue.kt:690)
	at com.intellij.ide.IdeEventQueue._dispatchEvent$lambda$10(IdeEventQueue.kt:593)
	at com.intellij.openapi.application.impl.ApplicationImpl.runWithoutImplicitRead(ApplicationImpl.java:1485)
	at com.intellij.ide.IdeEventQueue._dispatchEvent(IdeEventQueue.kt:593)
	at com.intellij.ide.IdeEventQueue.access$_dispatchEvent(IdeEventQueue.kt:67)
	at com.intellij.ide.IdeEventQueue$dispatchEvent$processEventRunnable$1$1$1.compute(IdeEventQueue.kt:369)
	at com.intellij.ide.IdeEventQueue$dispatchEvent$processEventRunnable$1$1$1.compute(IdeEventQueue.kt:368)
	at com.intellij.openapi.progress.impl.CoreProgressManager.computePrioritized(CoreProgressManager.java:787)
	at com.intellij.ide.IdeEventQueue$dispatchEvent$processEventRunnable$1$1.invoke(IdeEventQueue.kt:368)
	at com.intellij.ide.IdeEventQueue$dispatchEvent$processEventRunnable$1$1.invoke(IdeEventQueue.kt:363)
	at com.intellij.ide.IdeEventQueueKt.performActivity$lambda$1(IdeEventQueue.kt:997)
	at com.intellij.openapi.application.TransactionGuardImpl.performActivity(TransactionGuardImpl.java:113)
	at com.intellij.ide.IdeEventQueueKt.performActivity(IdeEventQueue.kt:997)
	at com.intellij.ide.IdeEventQueue.dispatchEvent$lambda$7(IdeEventQueue.kt:363)
	at com.intellij.openapi.application.impl.ApplicationImpl.runIntendedWriteActionOnCurrentThread(ApplicationImpl.java:861)
	at com.intellij.ide.IdeEventQueue.dispatchEvent(IdeEventQueue.kt:405)
	at java.desktop/java.awt.EventDispatchThread.pumpOneEventForFilters(EventDispatchThread.java:207)
	at java.desktop/java.awt.EventDispatchThread.pumpEventsForFilter(EventDispatchThread.java:128)
	at java.desktop/java.awt.EventDispatchThread.pumpEventsForHierarchy(EventDispatchThread.java:117)
	at java.desktop/java.awt.EventDispatchThread.pumpEvents(EventDispatchThread.java:113)
	at java.desktop/java.awt.EventDispatchThread.pumpEvents(EventDispatchThread.java:105)
	at java.desktop/java.awt.EventDispatchThread.run(EventDispatchThread.java:92)
Caused by: java.lang.NoClassDefFoundError: com/intellij/ide/util/PropertiesComponentImpl
	at io.zhile.research.intellij.ier.common.Resetter.getEvalRecords(Resetter.java:72)
	at io.zhile.research.intellij.ier.listener.AppEventListener.appClosing(AppEventListener.java:41)
	at com.intellij.util.messages.impl.MessageBusImplKt.invokeMethod(MessageBusImpl.kt:696)
	at com.intellij.util.messages.impl.MessageBusImplKt.invokeListener(MessageBusImpl.kt:659)
	... 55 more
Caused by: java.lang.ClassNotFoundException: com.intellij.ide.util.PropertiesComponentImpl PluginClassLoader(plugin=PluginDescriptor(name=IDE Eval Reset, id=io.zhile.research.ide-eval-resetter, descriptorPath=plugin.xml, path=~\AppData\Roaming\JetBrains\CLion2023.2\plugins\ide-eval-resetter, version=2.3.5, package=null, isBundled=false), packagePrefix=null, state=active)
	at com.intellij.ide.plugins.cl.PluginClassLoader.loadClass(PluginClassLoader.kt:156)
	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:525)
	... 59 more
~~~

### 解决方法

这个错误是由于错误的第三方插件引起的，要解决此问题，请备份并删除插件的目录。

~~~text
# windows插件的目录位置
# username替换为自己实际用户目录
# CLion2023.2替换为自己对于版本目录
C:\Users\username\AppData\Roaming\JetBrains\CLion2023.2\plugins
~~~

### ref

- https://rider-support.jetbrains.com/hc/en-us/community/posts/12723703124370-My-IDE-cannot-start-because-of-a-Java-error
- https://www.jetbrains.com/help/rider/Directories_Used_by_the_IDE_to_Store_Settings_Caches_Plugins_and_Logs.html#plugins-directory

## 重新安装Goland后，无法启动

> jb其他产品出现同样的问题，也能用此方法解决

### 问题描述

Goland重现安装新的版本后，点击图标无响应，无法启动。卸载重装软件也没有效果。

### 解决方法

由于之前版本软件没有卸载干净，删除AppData\Roaming\JetBrains目录中对应软件文件夹（Local文件夹中也有JetBrains目录，好像可以不用删，但是建议也删了）

~~~text
# windows插件的目录位置
# username替换为自己实际用户目录
# CLion2023.2为需要删除的文件夹，替换为自己对于版本目录
C:\Users\username\AppData\Roaming\JetBrains\CLion2023.2
~~~

### ref

- https://blog.csdn.net/FFDreamer/article/details/125441858