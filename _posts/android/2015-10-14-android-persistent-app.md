---
layout: post
title: "android persistent app"
description:
category: android
tags: [android, framework]
mathjax: 
chart:
comments: false
---

###1. android persistent app

在android 系统中， 有一种永久性的app， 它们会在开机时自启动， 并且在出现异常时， 能够自动重启， 永久性的应用， 是通过在 AndroidManifest.xml 文件中声明 android:persistent="true" 来实现的

###2. persistent app的实现

在PackageManagerService中有一个hash表， 用于记录所有app的信息

	final HashMap<String, PackageParser.Package> mPackages = new HashMap<String, PackageParser.Package>();

其中 PackageParser.Package.applicationInfo 成员保存了app的相关信息， 当app被声明为persistent app时(android:persistent="true"), 其对应的 PackageParser.Package.applicationInfo.flag 成员的 FLAG_PERSISTENT 位会被置1， 标记其为persistent app

PackageManagerService.getPersistentApplications()可用于获取所有的persistent app的 ApplicationInfo

####2.1 persistent app的自启动

在ActivityManagerService.systemReady()中， 会通过 PackageManagerService 获取所有的persistent app并且启动它们

	public void systemReady(final Runnable goingCallback) {
		......
		synchronized (this) {
            		if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
                		try {
                    		List apps = AppGlobals.getPackageManager().getPersistentApplications(STOCK_PM_FLAGS);
                    		if (apps != null) {
                        			int N = apps.size();
                        			int i;
                        			for (i=0; i<N; i++) {
                            			ApplicationInfo info = (ApplicationInfo)apps.get(i);
                            			if (info != null && !info.packageName.equals("android")) {
                                				addAppLocked(info, false, null /* ABI override */);
                            			}
                        			}
                    		}
                		} catch (RemoteException ex) {
                    		// pm is in same process, this will never happen.
                		}
            	}
		......
	}

对所有包名不为“android”的persistent app执行 addAppLocked(), addAppLocked()中会添加一个与App进程对应的ProcessRecord节点(不会重复添加)， 然后检查到若该app进程是否启动，若未启动，则会调用startProcessLocked()启动该app进程

启动进程的过程是异步的， 一旦目标进程启动完毕后， 会走到ActivityManagerService.attachApplicationLocked()， 其中会删除对应的ProcessRecord节点

####2.2 persistent app的自动重启

每个ActivityThread中会有一个专门和AMS通信的binder实体——final ApplicationThread mAppThread， 在启动app进程完成后， ActivityManagerService.attachApplicationLocked()中会为该ApplicationThread注册一个死亡监听

	try {
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            app.resetPackageList(mProcessStats);
            startProcessLocked(app, "link fail", processName);
            return false;
        }

当监听的 AtivityThread 死亡后， AppDeathRecipient.binderDied()会被回调， 最终会重启

	AppDeathRecipient.binderDied()
		appDiedLocked()
			handleAppDiedLocked()
	
	private final boolean cleanUpApplicationRecordLocked()	{
		......
		} else if (!app.removed) {
            		// This app is persistent, so we need to keep its record around.
            		// If it is not already on the pending app list, add it there
            		// and start a new process for it.
            		if (mPersistentStartingProcesses.indexOf(app) < 0) {
                			mPersistentStartingProcesses.add(app);
                			restart = true;
            		}
        		}
		......
		if (restart && !app.isolated) {
            		// We have components that still need to be running in the
            		// process, so re-launch it.
            		if (index < 0) {
                			ProcessList.remove(app.pid);
            		}
            		mProcessNames.put(app.processName, app.uid, app);
            		startProcessLocked(app, "restart", app.processName);
            		return true;
		......
	}
			
