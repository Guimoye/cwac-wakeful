CWAC Wakeful: Staying Awake At Work
===================================

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

This is available as a JAR file from the downloads area of this GitHub repo.
The project itself is set up as an Android library project,
in case you wish to use the source code in that fashion.

**NOTE**: `WakefulIntentService` v0.4.0 and newer requires Android 2.0+, so it
can take advantage of `onStartCommand()` for better handling of
crashed services. Use earlier versions of `WakefulIntentService` if
you wish to try to use it on older versions of Android, though this
is not supported.

Usage
-----
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

Dependencies
------------
None.

Version
-------
This is version v0.4.3 of this module, meaning it is being beaten
to a pulp by reusers, prompting some revisions.

Demo
----
In the `demo/` project directory and `com.commonsware.cwac.wakeful.demo` package you will find
an `OnBootReceiver` designed to be attached to the `BOOT_COMPLETED`
broadcast `Intent`. `OnBootReceiver` schedules an alarm, which is sent
to `OnAlarmReceiver`. `OnAlarmReceiver` in turn asks `AppService` (which
extends `WakefulIntentService`) to do some work in a background
thread. It uses
the library project itself to access the source code and
resources of the `WakefulIntentService` library.

Note that when you build the JAR via `ant jar`, the sample
activity is not included, nor any resources -- only the
compiled classes for the actual library are put into the JAR.

License
-------
The code in this project is licensed under the Apache
Software License 2.0, per the terms of the included LICENSE
file.

Questions
---------
If you have questions regarding the use of this code, please
join and ask them on the [cw-android Google Group][gg]. Be sure to
indicate which CWAC module you have questions about.

Release Notes
-------------
- v0.4.3: added better recovery from an `Intent` redelivery condition
- v0.4.2: added `volatile` keyword to static `WakeLock` for better double-checked locking implementation
- v0.4.1: added `setIntentRedelivery()` call, nuked extraneous permissions check
- v0.4.0: switched to `onStartCommand()`, requiring Android 2.0+ (API level 5 or higher)
- v0.3.0: converted to Android library project, added test for `WAKE_LOCK` permission

[gg]: http://groups.google.com/group/cw-android
