---
title: "Tracing FGS Restrictions in AOSP (API 28–34)"
description: "Why Android's biggest updates over the last few years have nothing to do with the UI, and everything to do with restricting background execution."
publishDate: "2026-10-03"
# ogImage: "/android-vers-s.png"
tags: ["android", "security", "aosp"]
---

When Android started releasing Android 17 to personal devices, a friend got the update notification and asked if I had downloaded it yet. After I replied positively and mentioned I found it pretty cool, he was puzzled by my reaction - and generally by me spending my free time reading Android version releases.

"Is it really different than the last version?" he asked. "I hear people talk about OS version releases, but they look the same to me. Could you actually spot the difference if asked?"

If I'm being honest, he was mostly right. On the surface, it does look the same. But that's just it - it's not the UI/UX changes I'm interested in. The most intriguing updates and new features are usually not in how the OS UI looks or behaves, but in the underlying security updates - the changes only the SDK consumers notice.

To justify my peculiar interest in Android versions, I will go over some older release docs, focusing mostly on security restrictions. As I see it, each new restriction in a version release is another way of saying `this vulnerability existed until today`.

Understanding the severity of these updates requires a basic grasp of **Foreground Services**. These services allow apps to run code in multiple app lifecycle stages - .foreground, background, and even after you swipe the app away from recents. To stop a persistent FGS, you actually have to force-stop the app entirely or kill the service directly from 'Running Services' in developer settings.

From a developer's perspective, this is an incredibly useful SDK feature. I have personally used it for keeping tasks running, like a live GPS connection, a media player, or communication channels for emergency communication apps. On the other end, from a user's standpoint, this is a catastrophe waiting to happen. Apps can use it to launch virtually any code and run it without any user initiation.

Back in 2018 with **Android 9 (API Level 28)**, the docs stated the following:

> Apps that target Android 9 or higher and use foreground services must request the **FOREGROUND_SERVICE permission**. If an app that targets Android 9 or higher attempts to create a foreground service without requesting FOREGROUND_SERVICE, the system throws a **SecurityException**.

```java
/**
 * AOSP: frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
 * Enforcing the baseline permission requirement introduced in API 28
 */
if (appInfo.targetSdkVersion >= Build.VERSION_CODES.P) {
    mAm.enforcePermission(
            android.Manifest.permission.FOREGROUND_SERVICE,
            r.app.getPid(), r.appInfo.uid, "startForeground");
}
```

When most people read this, they most likely think, 'Good, okay then.' When I read it, my immediate thought is: how was running these services without a required permission even allowed until 2018? What does it mean for the capabilities apps on my devices had up until now?

That said, while requiring this permission was a step forward, it didn't actually fix the clear security risk. The FGS permission was considered a "**Normal permission**" so once a developer declared it in their app's manifest file, permission was granted automatically.

> This is a normal permission, so the system automatically grants it to the requesting app.

Because the permission was auto-granted, an app using it didn't need to explicitly ask from the user for the permission. Right after downloading the app, the permission was auto-granted and the app could start the service without the user getting the opportunity to hit reject.
Further more, another major problem wasn't addressed yet. The OS didn't require the app to be foregrounded to start an FGS and run code on the device - it could easily start an FGS from the background without any user interaction. Finally, with the Android 12 (API level 31) release, the Android team addressed this exact loophole.

> Apps that target Android 12 (API level 31) or higher can't start foreground services while the app is running in the background, except for a few special cases. If an app tries to start a foreground service while the app runs in the background, and the foreground service doesn't satisfy one of the exceptional cases, the system throws a `ForegroundServiceStartNotAllowedException`.
> In addition, if an app wants to launch a foreground service that needs while-in-use permissions (for example, body sensor, camera, microphone, or location permissions), it cannot create the service while the app is in the background, even if the app falls into one of the exemptions from background start restrictions. The reason for this is explained in the section Restrictions on starting foreground services that need while-in-use permissions.

With Android's 12 release, the Android team addressed the issue mentioned above. A foreground service could no longer start from a backgrounded app. Still, the issue wasn't fully resolved - because of one of the special cases, an app could start a FGS from the background.

> After the device reboots and receives the ACTION_BOOT_COMPLETED, ACTION_LOCKED_BOOT_COMPLETED, or ACTION_MY_PACKAGE_REPLACED intent action in a broadcast receiver.

The `ACTION_BOOT_COMPLETED` intent is the intent the device broadcasts after a device restart/power on. It has a few legitimate purposes (e.g. rescheduling an alarm or event notification), and some apps rely on it to start automatically after a device restart. By declaring a broadcast receiver for the intent, they start long-running foreground services in response to the restart event.

```java
/**
 * AOSP: frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
 * Blocking illegal BOOT_COMPLETED starts
 */
if (!shouldAllowBootCompletedStart(r, foregroundServiceType)) {
    throw new ForegroundServiceStartNotAllowedException("FGS type "
            + ServiceInfo.foregroundServiceTypeToLabel(foregroundServiceType)
            + " not allowed to start from BOOT_COMPLETED!");
}
```

While useful for ensuring tasks and long running services continue running after a device restart, this feature could've and have been used maliciously.
Just like a FGS shouldn't start from the background, it also shouldn't auto-start after a restart. It poses a major risk for the device owner. Auto-starting on reboot grants apps persistent execution. This way, a malicious app could survive a device restart and immediately begin running background jobs, pinging remote servers, or draining battery - running perpetually before the user ever chooses to open the app again.

Fortunately, the issue was addressed in the Android 14 release. It required developers to specify the exact type for each foreground service in their app. The OS also enforced specific rules for each type. For example, for starting a location foreground service, you would have to get either ACCESS_COARSE_LOCATION or ACCESS_FINE_LOCATION permissions.

> If your app targets Android 14 (API level 34) or higher, it must specify at least one foreground service type for each foreground service within your app. You should choose a foreground service type that represents your app's use case. The system expects foreground services that have a particular type to satisfy a particular use case.

```java
/**
 * AOSP: frameworks/base/services/core/java/com/android/server/am/ActiveServices.java
 * Enforcing API 34+ type requirements
 */
throw new MissingForegroundServiceTypeException(
    "Starting FGS without a type callerApp=" + callerApp
);
```

So, I most likely couldn't spot the difference between versions by judging the UI, but the fixed vulnerabilities are much more interesting anyway.

## Resources

- [Android Developers: Foreground services overview](https://developer.android.com/guide/components/foreground-services)
- [Android Developers: Android 9 behavior changes](https://developer.android.com/about/versions/pie/android-9.0-changes-28#fg-svc)
- [Android Developers: Android 12 background start restrictions](https://developer.android.com/about/versions/12/foreground-services)
- [Android Developers: Android 14 foreground service types](https://developer.android.com/about/versions/14/changes/fgs-types-required)
- [AOSP: `ActiveServices.java` on Android Code Search](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/services/core/java/com/android/server/am/ActiveServices.java)

_Note: Android source code snippets are referenced from the Android Open Source Project (AOSP), licensed under Apache 2.0._
