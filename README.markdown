CWAC Wakeful: Staying Awake At Work
===================================

**UPDATE 2017-09-28**: Users of this library should migrate [to `JobIntentService`](https://developer.android.com/reference/android/support/v4/app/JobIntentService.html) from the Android Support Libraries. That class covers the primary `WakefulIntentService` use case, is officially supported, and helps with the Android 8.0+ limitation on the life of background services. For the classic `WakefulIntentService` pattern of just overriding `doWakefulWork()`: changing the superclass, refactoring that method to `onHandleWork()`, and using `enqueueWork()` to start the work, should be all that is needed to switch to `JobIntentService`.

This library is officially discontinued, unless some use case arises where `JobIntentService` is unsuitable. If you think that you have one, file an issue here.

The rest of the documentation remains for the historical (and possibly hysterical) record.

-----

The recommended pattern for Android's equivalent to cron
jobs and Windows scheduled tasks is to use `AlarmManager`.
This works well when coupled with an `IntentService`, as the
service will do its work on a background thread and shut down
when there is no more work to do.

There's one small problem: `IntentService` does nothing to keep
the device awake. If the alarm was a `WAKEUP` variant, the phone
will only stay awake on its own while the `BroadcastReceiver`
handling the alarm is in its `onReceive()` method. Otherwise,
the phone may fall back asleep.

`WakefulIntentService` attempts to combat this by combining
the ease of `IntentService` with a partial `WakeLock`.

This is [available as a JAR file](https://github.com/commonsguy/cwac-wakeful/releases),
or as an artifact for use with Gradle. To use that, add the following
blocks to your `build.gradle` file:

```groovy
repositories {
    maven {
        url "https://s3.amazonaws.com/repo.commonsware.com"
    }
}

dependencies {
    compile 'com.commonsware.cwac:wakeful:1.0.+'
}
```

Or, if you cannot use SSL, use `http://repo.commonsware.com` for the repository
URL.

The project itself is set up as an Android library project,
in case you wish to use the source code in that fashion.

**NOTE**: The JAR name, as of v1.0.2, has a `cwac-` prefix, to help distinguish it from other JARs.

Basic Usage
-----------
Any component that wants to send work to a
`WakefulIntentService` subclass needs to call either:

`WakefulIntentService.sendWakefulWork(context, MyService.class);`

(where `MyService.class` is the `WakefulIntentService` subclass)

or:

`WakefulIntentService.sendWakefulWork(context, intentOfWork);`

(where `intentOfWork` is an `Intent` that will be used to call
`startService()` on your `WakefulIntentService` subclass)

Implementations of `WakefulIntentService` must override
`doWakefulWork()` instead of `onHandleIntent()`. `doWakefulWork()`
will be processed within the bounds of a `WakeLock`. Otherwise,
the semantics of `doWakefulWork()` are identical to `onHandleIntent()`.
`doWakefulWork()` will be passed the `Intent` supplied to
`sendWakefulWork()` (or an `Intent` created by the `sendWakefulWork()`
method, depending on which flavor of that method you use).

And that's it. `WakefulIntentService` handles the rest.

NOTE: this only works with local services. You have no means
of accessing the static `WakeLock` of a remote service.

NOTE #2: Your application must hold the `WAKE_LOCK` permission.

NOTE #3: If you get an "`WakeLock` under-locked" exception, make sure
that you are not starting your service by some means other than
`sendWakefulWork()`.

Alarm Usage
-----------
If you want to slightly simplify your use of `WakefulIntentService`
in conjunction with `AlarmManager`, you can do the following:

First, implement your `WakefulIntentService` and `doWakefulWork()`
as described above.

Next, create a class implementing the `WakefulIntentService.AlarmListener`
interface. This class needs to have a no-argument public constructor
in addition to the interface method implementations. One method
is `scheduleAlarms()`, where you are passed in an `AlarmManager`,
a `PendingIntent`, and a `Context`, and your mission is to schedule
your alarms using the supplied `PendingIntent`. You also implement
`sendWakefulWork()`, which is passed a `Context`, and is where
you call `sendWakefulWork()` upon your `WakefulIntentService`
implementation. And, you need to implement `getMaxAge(Context)`, which
should return the time in milliseconds after which, if we have
not seen an alarm go off, you should assume that the alarms were
canceled (e.g., application was force-stopped by the user) and
should reschedule them.

Then, create an XML metadata file where you identify the class
that implements `WakefulIntentService.AlarmListener` from the
previous step, akin to:

    <WakefulIntentService
      listener="com.commonsware.cwac.wakeful.demo.AppListener"
    />

Next, register `com.commonsware.cwac.wakeful.AlarmReceiver`
as a `<receiver>` in your manifest, set to respond to
`ACTION_BOOT_COMPLETED` broadcasts, and with a `com.commonsware.cwac.wakeful`
`<meta-data>` element pointing to the XML resource from 
the previous step, akin to:

    <receiver android:name="com.commonsware.cwac.wakeful.AlarmReceiver">
      <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
      </intent-filter>

      <meta-data
        android:name="com.commonsware.cwac.wakeful"
        android:resource="@xml/wakeful"/>
    </receiver>

Also, add the `RECEIVE_BOOT_COMPLETED` permission to your manifest.

Finally, when you wish to manually set up the alarms (e.g., on
first run of your app), create an instance of your `AlarmListener`
and call `scheduleAlarms()` on the `WakefulIntentService`
class, passing in the `AlarmListener` and a `Context` (e.g.,
the activity that is trying to set up the alarms). If you are only
scheduling alarms using the single provided `PendingIntent`, you
can also call `cancelAlarms()` on the `WakefulIntentService` class
to cancel any outstanding alarms. Note that `scheduleAlarms()`
and `cancelAlarms()` perform disk I/O and should be called on a
background thread.

For production use, ProGuard may rename your `AlarmListener`
class, which will foul up access to your metadata. To stop this
from happening, you
[will need to add a `-keep` line to your ProGuard configuration file](http://developer.android.com/guide/developing/tools/proguard.html#configuring)
(e.g., `proguard.cfg`) to stop ProGuard from renaming it.

Limitations
-----------
Your `BroadcastReceiver` and your `WakefulIntentService` need to be in the same process.

Dependencies
------------
None.

This project should work on API Level 7 and higher, except for any portions that
may be noted otherwise in this document. Please report bugs if you find features
that do not work on API Level 7 and are not noted as requiring a higher version.

Version
-------
This is version v1.1.0 of this module, meaning it is for realz.

Demo
----
In the `demo/` project directory and `com.commonsware.cwac.wakeful.demo` package you will find
an `AppListener`, which is an implementation of `AlarmListener`,
and `AppService`, which
extends `WakefulIntentService`. `AppService` pretends to do some work in a background
thread. All of this is set up via a `DemoActivity` (required
to move the application out of the "stopped" state on Android 3.1+),
and if needed on a reboot.

Additional Documentation
------------------------
[The Busy Coder's Guide to Android Development](https://commonsware.com/Android)
has two chapters on `AlarmManager` and `JobScheduler` that demonstrate
this library and its use cases. Also, the main series of tutorials
use `WakefulIntentService` as well, so you can see its use in a somewhat
more complex app.

License
-------
The code in this project is licensed under the Apache
Software License 2.0, per the terms of the included LICENSE
file.

Questions
---------
If you have questions regarding the use of this code, please post a question
on [Stack Overflow](http://stackoverflow.com/questions/ask) tagged with
`commonsware-cwac` and `android` after [searching to see if there already is an answer](https://stackoverflow.com/search?q=[commonsware-cwac]+wakefulintentservice). Be sure to indicate
what CWAC module you are having issues with, and be sure to include source code 
and stack traces if you are encountering crashes.

If you have encountered what is clearly a bug, please post an [issue](https://github.com/commonsguy/cwac-wakeful/issues). Be certain to include complete steps
for reproducing the issue.
The [contribution guidelines](CONTRIBUTING.md)
provide some suggestions for how to create a bug report that will get
the problem fixed the fastest.

You are also welcome to join
[the CommonsWare Community](https://community.commonsware.com/)
and post questions
and ideas to [the CWAC category](https://community.commonsware.com/c/cwac).

Do not ask for help via social media.

Also, if you plan on hacking
on the code with an eye for contributing something back,
please open an issue that we can use for discussing
implementation details. Just lobbing a pull request over
the fence may work, but it may not.
Again, the [contribution guidelines](CONTRIBUTING.md) should help here.

Release Notes
-------------
- v1.1.0: upgraded Gradle/build tools, added Context parameter to getMaxAge()
- v1.0.5: updated to Android Studio 1.0 and new AAR publishing system
- v1.0.4: added exception handler to catch any under-locked `WakeLock` runtime errors
- v1.0.3: fixed bug in `cancelAlarms()`
- v1.0.2: fixed manifest for merging, added `cwac-` prefix to JAR
- v1.0.1: added Gradle build files and published AAR as an artifact
- v1.0.0: anointed major release
- v0.6.2: added more fail-safes around `WakeLock` acquisition and release
- v0.6.1: replaced `AlarmListener` `Log` lines with `RuntimeExceptions`
- v0.6.0: added `cancelAlarms()` to `WakefulIntentService`
- v0.5.1: semi-automatically handle canceled alarms (e.g., app force-stopped)
- v0.5.0: added the `AlarmListener` portion of the framework
- v0.4.5: completed switch to `Application` as the `Context` for the `WakeLock`
- v0.4.4: switched to `Application` as the `Context` for the `WakeLock`
- v0.4.3: added better recovery from an `Intent` redelivery condition
- v0.4.2: added `volatile` keyword to static `WakeLock` for better double-checked locking implementation
- v0.4.1: added `setIntentRedelivery()` call, nuked extraneous permissions check
- v0.4.0: switched to `onStartCommand()`, requiring Android 2.0+ (API level 5 or higher)
- v0.3.0: converted to Android library project, added test for `WAKE_LOCK` permission

