---
title: "[SW.Modification] Android - Frameworks - How to add service? How to use?"
date: 2026-08-26 19:06:00
categories:
- SW.Modification
- Devnotes
---

# What is service?

A Service is an application component that can perform long-running operations in the background and does not provide a user interface. Another application component can start a service and it will continue to run in the background even if the user switches to another application. Additionally, a component can bind to a service to interact with it and even perform interprocess communication (IPC). For example, a service might handle network transactions, play music, perform file I/O, or interact with a content provider, all from the background.

# What we want?

We have an Android project which can be used at test for services. We have a sample SaikiService which runs in background and responds to a call for test function.

## How to insert service?

This is the route for service being added in boot:

``
new service->addService->registerService
``

* **new service**: New handle of service (must exist in initAndLoop), create with context parameter
* **addService**: Adding service to manager; Place a new @a service called @a name into the service manager.
* **registerService**: Link service to string

First, service name should be declared in Context (frameworks/base/core/java/android/content/Context.java):

```
public abstract class Context {
...
    //<------
   /**
    * my service
    * @hide
    */
    public static final String SAIKI_SERVICE = "saiki_kusuo";
    //------>
}
```

new service + addService -> frameworks/base/services/java/com/android/server/SystemServer.java: these are going to the initAndLoop function for boot progress initialization, these will start all necessary service: battery, account, vibrator, display etc. Our new service is added as follows:

```java
    public void initAndLoop() {
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN,
            SystemClock.uptimeMillis());
        /// M: BOOTPROF @{
        mMTPROF_disable = "1".equals(SystemProperties.get("ro.mtprof.disable"));
        addBootEvent(new String("Android:SysServerInit_START"));
        /// @}
...
        //Add for LTE DC project
        TelephonyRegistry telephonyRegistryLteDc = null; 
        ///M: add for MobileManagerService
        IMobileManagerService mom = null;

        ///M: add for hdmi feature
        IMtkHdmiManager hdmiManager = null;

        //<------
        SaikiService saiki = null;
        //------>

        // Create a handler thread just for the window manager to enjoy.
        HandlerThread wmHandlerThread = new HandlerThread("WindowManager");
        wmHandlerThread.start();
        Handler wmHandler = new Handler(wmHandlerThread.getLooper());
...

        boolean disableStorage = SystemProperties.getBoolean("config.disable_storage", false);
        boolean disableMedia = SystemProperties.getBoolean("config.disable_media", false);
        boolean disableBluetooth = SystemProperties.getBoolean("config.disable_bluetooth", false);
        boolean disableTelephony = SystemProperties.getBoolean("config.disable_telephony", false);
        boolean disableLocation = SystemProperties.getBoolean("config.disable_location", false);
        boolean disableSystemUI = SystemProperties.getBoolean("config.disable_systemui", false);
        boolean disableNonCoreServices = SystemProperties.getBoolean("config.disable_noncore", false);
        boolean disableNetwork = SystemProperties.getBoolean("config.disable_network", false);

        try {
            Slog.i(TAG, "Display Manager");
            display = new DisplayManagerService(context, wmHandler);
            ServiceManager.addService(Context.DISPLAY_SERVICE, display, true);

...

            inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
            inputManager.start();

            display.setWindowManager(wm);
            display.setInputManager(inputManager);

            //<------
            Slog.i(TAG, "SaikiService");
            saiki = new SaikiService(context);
            ServiceManager.addService(Context.SAIKI_SERVICE, saiki);
            //------>
...
        } catch (RuntimeException e) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting core service", e);
        }
...
    }
```

We have to declare service before hand, this is set as null. We add service as last from bunch, after display and input management setup. This creates a new handle with context parameter and then add our service.

## How to register service?

registerService -> frameworks/base/core/java/android/app/ContextImpl.java: the service created is placed under SYSTEM_SERVICE_MAP and fetcher context cache count is increased.

```java
    private static void registerService(String serviceName, ServiceFetcher fetcher) {
        if (!(fetcher instanceof StaticServiceFetcher)) {
            fetcher.mContextCacheIndex = sNextPerContextServiceCacheIndex++;
        }
        SYSTEM_SERVICE_MAP.put(serviceName, fetcher);
    }
```

With our register which is created, a fetcher is required to return the manager for it, initialized with service interface which is acquired from getService of ServiceManager.

* **getService**: Returns a reference to a service with the given name

```java
    static {
        registerService(ACCESSIBILITY_SERVICE, new ServiceFetcher() {
                public Object getService(ContextImpl ctx) {
                    return AccessibilityManager.getInstance(ctx);
                }});

        registerService(CAPTIONING_SERVICE, new ServiceFetcher() {
                public Object getService(ContextImpl ctx) {
                    return new CaptioningManager(ctx);
                }});
...
        //<------
        registerService(SAIKI_SERVICE, new ServiceFetcher() {
            public Object createService(ContextImpl ctx) {
                IBinder b = ServiceManager.getService(SAIKI_SERVICE);
                ISaikiService service = ISaikiService.Stub.asInterface(b);
                return new SaikiManager(service);
            }});
        //------>
    }
```

Before compile, please add the source files in:

1. frameworks/base/Android.mk:

```make
## READ ME: ########################################################
##
## When updating this list of aidl files, consider if that aidl is
## part of the SDK API.  If it is, also add it to the list below that
## is preprocessed and distributed with the SDK.  This list should
## not contain any aidl files for parcelables, but the one below should
## if you intend for 3rd parties to be able to send those objects
## across process boundaries.
##
## READ ME: ########################################################
LOCAL_SRC_FILES += \
...
ifeq ($(MTK_3GDONGLE_SUPPORT),yes)
 #LOCAL_SRC_FILES := $(filter-out  ../opt/telephony/src/java/com/android/internal/telephony/cdma/%,$(LOCAL_SRC_FILES))
 #LOCAL_SRC_FILES += $(call all-java-files-under, ../opt/telephony/src/java_tb)
  LOCAL_SRC_FILES := $(filter-out  telephony/java/android/telephony/cdma/%,$(LOCAL_SRC_FILES))
  LOCAL_SRC_FILES := $(filter-out  telephony/java/com/android/internal/telephony/cdma/%,$(LOCAL_SRC_FILES))
  LOCAL_SRC_FILES += $(call all-java-files-under, telephony/java_tb)
endif

# <-------
SAIKI_SERVICE_PATH := ../../vendor/saikikusuo/services/SaikiService
# Manager
LOCAL_SRC_FILES += $(SAIKI_SERVICE_PATH)/frameworks_java/com/saikikusuo/SaikiManager.java
# Aidl
LOCAL_SRC_FILES += $(SAIKI_SERVICE_PATH)/frameworks_java/com/saikikusuo/ISaikiService.aidl
# ------->
...
```

2. frameworks/base/services/java/Android.mk:

```
LOCAL_PATH:= $(call my-dir)

# the library
# ============================================================
include $(CLEAR_VARS)

LOCAL_SRC_FILES := \
            $(call all-subdir-java-files) \
	    com/android/server/EventLogTags.logtags \
	    com/android/server/am/EventLogTags.logtags

MTK_SERVICES_JAVA_PATH := ../../../../mediatek/frameworks-ext/base/services/java
LOCAL_SRC_FILES += $(call all-java-files-under,$(MTK_SERVICES_JAVA_PATH))

# <-------
SAIKI_SERVICE_PATH := ../../../../vendor/saikikusuo/services/SaikiService
# Service
LOCAL_SRC_FILES += $(SAIKI_SERVICE_PATH)/service_java/SaikiService.java
# Aidl
LOCAL_SRC_FILES += $(SAIKI_SERVICE_PATH)/frameworks_java/com/saikikusuo/ISaikiService.aidl
# ------->

LOCAL_MODULE:= services
...
```

## Testing

The image must be rebuilded with dependency. Afterwards, flash new system.img.

In adb shell, check the service exists:

```
shell@hexing72_cwet_lca_wd411_test:/ $ service list
Found 95 services:
0       phoneEx: [com.mediatek.common.telephony.ITelephonyEx]
1       phone: [com.android.internal.telephony.ITelephony]
2       iphonesubinfo2: [com.android.internal.telephony.IPhoneSubInfo]
3       simphonebook2: [com.android.internal.telephony.IIccPhoneBook]
4       isms2: [com.android.internal.telephony.ISms]
...
50      saiki_kusuo: [com.saikikusuo.ISaikiService] <----- our service
...
```

## How to use service in app?

Note: the app must have manager file source to get service handle. In our app makefile include the manager file of library.

```java
    SaikiManager mManager = (SaikiManager)getApplicationContext().getSystemService("saiki_kusuo");
    if (mManager == null)
    {
        Toast.makeText(getApplicationContext(),"fail no service found", Toast.LENGTH_SHORT).show();
        return;
    }
    mManager.TestMe();
```

If sound plays, the service has been added and working.

# Source

https://developer.android.com/develop/background-work/services