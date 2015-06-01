>These documents are Copyright (c) 2009-2010 by Nick Mathewson, and are made available under the Creative Commons Attribution-Noncommercial-Share Alike license, version 3.0. Future versions may be made available under a less restrictive license.

>Additionally, the source code examples in these documents are also licensed under the so-called "3-Clause" or "Modified" BSD license. See [the license_bsd file](http://www.wangafu.net/~nickm/libevent-book/license_bsd.html) distributed with these documents for the full terms.

>For the latest version of this document, see [TOC](http://www.wangafu.net/~nickm/libevent-book/TOC.html)

>To get the source for the latest version of this document, install git and run "git clone git://github.com/nmathewson/libevent-book.git"

#0. Preliminaries
Libevent is a library for writing fast portable nonblocking IO. Its design goals are:

- **Portability** A program written using Libevent should work across all the platform Libevent supports. Even when there is no really good way to do nonblocking IO, Libevent should support the so-so ways, so that your program can run in restricted environments.
- **Speed** Libevent tries to use the fastest available nonblocking IO implementations on each platform, and not to introduce much overhead as it does so.
- **Scalability** Libevent is designed to work well even with programs that need to have tens of thousands of active sockets.
- **Convenience** Whenever possible, the most natural way to write a program with Libevent should be the stable, portable way.

Libevent is divided into the following components:

- **evutil** Generic functionality to abstract out the difference between different platform's networking implementations.
- **event and event_base** This is the heart of Libevent. It provides an abstract API to the various platform-specific, event-base nonblocking IO backends. It can let you know when sockets are ready to read or write, do basic timeout functionality, and detect OS signals.
- **bufferevent** These functions provide a more convenient wrapper around Libevent's event-based core. They let your application request buffered reads and writes, and rather than informing you when sockets are ready to do, they let you know when IO has actually occurred.
- **evbuffer** This module implements the buffers underlying bufferevents, and provides functions for efficient and/or convenient access.
- **evhttp** A simple HTTP client/server implementation.
- **evdns** A simple DNS client/server implementation.
- **evrpc** A simple RPC implementation.

When Libevent is built, by default it installs the following libraries:

- **libevent_core** All core event and buffer functionality. This library contains all the event_base, evbuffer, bufferevent, and utility functions.
- **libevent_extra** This library defines protocol-specific functionality that you may or may not want for your application, including HTTP, DNS, and RPC.
- **libevent** This library exists for historical reasons; it contains contents of both libevent_core and libevent_extra. You shouldn't use it; it may go away in a future version of Libevent.

The following libraries are installed only on some platforms:

- **libevent_pthreads** This library adds threading and locking implementations based on the pthreads portable threading library. It is separated from libevent_core so that you don't need to link against pthreads to use Libevent unless you are actually using Libevent in a multithreaded way.
- **libevent_openssl** This library provides support for encrypted communications using bufferevents and the OpenSSL library. It is separated from libevent_core so that you don't need to link against OpenSSL to use Libevent unless you are actually using encrypted connections.

#1. Setting up the Libevent library
Libevent has a few global settings that are shared across the entire process. These affect the entire library.

## Log messages in Libevent
Libevent can log internal errors and warnings. It also logs debugging messages if it was compiled with logging support. By default, these messages are written to stderr. You can override this behavior by providing your own logging function.

###Interface
```C
#define EVENT_LOG_DEBUG 0
#define EVENT_LOG_MSG 1
#define EVENT_LOG_WARN 2
#define EVENT_LOG_ERR 3

/* Deprecated */
#define _EVENT_LOG_DEBUG EVENT_LOG_DEBUG
#define _EVENT_LOG_MSG EVENT_LOG_MSG
#define _EVENT_LOG_WARN EVENT_LOG_WARN
#define _EVENT_LOG_ERR EVENT_LOG_ERR

typedef void (*event_log_cb)(int severity, const char* msg);

void event_set_log_callback(event_log_cb cb);
``` 

To override Libevent's logging behavior, write your own function matching the signature of `event_log_cb`, and pass it as an argument to `event_set_log_callback()`. Whenever Libevent wants to log a message, it will pass it to the function you provided. You can have Libevent return to its default behavior by calling `event_set_log_callback()` again with NULL as an argument.

```C
#define EVENT_DBG_NONE 0
#define EVENT_DBG_ALL 0xffffffffu

void event_enable_debug_logging(ev_uint32_t which);
```

Debugging logs are verbose, and not necessarily useful under most circumstances. Calling `event_enable_debug_logging()` with `EVENT_DBG_NONE` gets default behavior; calling it with `EVENT_DBG_ALL` turns on all the supported debugging logs.

##Handling fatal errors
When Libevent detects a non-recoverable internal error(such as corrupted data structure), its default behavior is to call `exit()` or `abort()` to leave the currently running process. These errors almost always mean that there is a bug somewhere: either in your code, or in Libevent itself.

You can override Libevent's behavior if you want your application to handle fatal errors more gracefully, by providing a function that Libevent should call in lieu of exiting.

```C
typedef void (*event_fatal_cb)(int err);
void event_set_fatal_callback(event_fatal_cb cb);
```

##Memory management
By default, Libevent uses the C library's memory management functions to allocate memory from the heap. You can have Libevent use another memory manager by providing your own replacements for `malloc`, `realloc`, and `free`. You might want to do this if you have a more efficient allocator that you want Libevent to use, or if you have an instrumented allocator that you want Libevent to use in order to look for memory leaks.

```C
void event_set_mem_functions(void *(*malloc_fn)(size_t sz),
	void *(*realloc_fn)(void *ptr, size_t sz),
	void (*free_fn)(void *ptr));
```

##Locks and threading
As you probably know if you're writing multithreaded programs, it isn't always safe to access the same data from multiple threads at the same time.

Libevent structures can generally work three ways with multithreading.

- Some structures are inherently single-threaded: it is never safe to use them from more than one thread at the same time.
- Some structures are optionally locked: you can tell Libevent for each object whether you need to use it from multiple threads at once.
- Some structures are always locked: if Libevent is running with lock support, then they are always safe to use from multiple threads at once.

To get locking in Libevent, you must tell Libevent which locking functions to use. You need to do this before you call any Libevent function that allows a structure that needs to be shared between threads.

```C
#ifdef WIN32
int evthread_use_windows_threads(void);
#define EVTHREAD_USE_WINDOWS_THREADS_IMPLEMENTED
#endif
#ifdef _EVENT_HAVE_PTHREADS
int evthread_use_pthreads(void);
#define EVTHREAD_USE_PTHREAD_IMPLEMENTED
#endif
```

##Debugging lock usage
To help debug lock usage, Libevent has an optional "lock debugging" feature that wraps its locking calls in order to catch typical lock errors, including:

- unlocking a lock that we don't actually hold
- re-locking a non-recursive lock

If one of these lock errors occurs, Libevent exits with an assertion failure.

```C
void evthread_enable_lock_debugging(void);
#define evthread_enable_lock_debugging() evthread_enable_lock_debugging()
```

##Debugging event usage
There are some common errors in using events that Libevent can detect and report for you. They include:

- Treating an uninitialized struct event as though it were initialized.
- Try to reinitialize a pending struct event.

Tracking which events are initialized requires that Libevent use extra memory and CPU, so you should only enable debug mode when actually debugging your program.

```C
void event_enable_debug_mode(void);
```

##Displaying the version of Libevent
New versions of Libevent can add features and remove bugs. Sometimes you'll want to detect the Libevent version, so that you can:

- Detect whether the installed version of Libevent is good enough to build your program.
- Display the Libevent version for debugging.
- Detect the version of Libevent so that you can warn the user about bugs, or work around them.

```C
#define LIBEVENT_VERSION_NUMBER 0x02000300
#define LIBEVENT_VERSION "2.0.3-alpha"
const char *event_get_version(void);
ev_uint32_t event_get_version_number(void);
```

##Freeing global Libevent structures
Even when you've freed all the objects that you allocated with Libevent, there will be a few globally allocated structures left over. If you need to make sure that Libevent has released all internal library-global data structure, you can call:

```C
void libevent_global_shutdown(void);
```

#2. Getting an event base
Before you can use any interesting Libevent function, you need to allocate one or more `event_base` structures. Each `event_base` structure holds a set of events and can poll to determine which events are active.

If an `event_base` is set up to use locking, it is safe to access it between multiple threads. Its loop can only be run in a single thread, however. If you want to have multiple threads polling for IO, you need to have an `event_base` for each thread.

Each `event_base` has a "method", or a backend that it uses to determine which events are ready. The recognized methods are:

- select
- poll
- epoll
- kqueue
- devpoll
- evport
- win32

##Setting up a default `event_base`
The `event_base_new()` function allocates and returns a new event base with the default settings. It examines the environment variables and returns a pointer to a new `event_base`. If there is an error, it returns NULL.

When choosing among methods, it picks the fastest method that the OS supports.

```C
struct event_base *event_base_new(void);
```

##Setting up a complicated `event_base`
If you want more control over what kind of `event_base` you get, you need to use an `event_config`. An `event_config` is an opaque structure that holds information about your preference for an `event_base`. When you want an `event_base`, you pass the `event_config` to `event_base_new_with_config()`.

```C
struct event_config *event_config_new(void);
struct event_base *event_base_new_with_config(const struct event_config *cfg);
void event_config_free(struct event_config *cfg);

int event_config_avoid_method(struct event_config *cfg, const char *method);

enum event_method_feature {
	EV_FEATURE_ET = 0x01,
	EV_FEATURE_O1 = 0x02,
	EV_FEATURE_FDS = 0x04,
};

int event_config_require_features(struct event_config *cfg,
	enum event_method_feature feature);

enum event_base_config_flag {
	EVENT_BASE_FLAG_NOLOCK = 0x01,
	EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
	EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
	EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
	EVENT_BASE_FLAG_EPOLL_USE_CHAGNELIST = 0x10,
	EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};

int event_config_set_flag(struct event_config *cfg,
	enum event_base_config_flag flag);
```

To allocate an `event_base` with these features, you call `event_config_new()` to allocate a new `event_config`. Then, you call other functions on the `event_config` to tell it about your needs. Finally, you call `event_base_new_with_config()` to get a new `event_base`. When you are done, you can free the `event_config` with `event_config_free()`.

Calling `event_config_avoid_method` tells Libevent to avoid a specific available backend by name. Calling `event_config_require_feature()` tells Libevent not to use any backend that cannot supply all of a set of features. Calling `event_config_set_flag()` tells Libevent to set one or more of the run-time flags when constructing the event base.

```C
int event_config_set_num_cpus_hint(struct event_config *cfg, int cpus);
```

This function is currently only useful with Windows when using IOCP. Calling it tells the `event_config` that the `event_base` it generates should try to make good use of a given number of CPUs when multithreading.

```C
int event_config_set_max_dispatch_interval(struct event_config *cfg,
	const struct timeval *max_interval, int max_callbacks,
	int min_priority);
```

This function prevents priority inversion by limiting how many low-priority event callbacks can be invoked before checking for more high-priority events. If `max_interval` is non-null, the event loop checks the time after each callback, and re-scans for high-priority events if `max_interval` has passed. If `max_callbacks` is nonnegative, the event loop also checks for more events after `max_callbacks` callbacks have been invoked. These rules apply to any event of `min_priority` or higher.

##Examining an `event_base`' s backend method
Sometimes you want to see which features are actually available in an `event_base`, or which method it's using.

```C
const char **event_get_supported_methods(void);
```

The `event_get_supported_methods()` function returns a pointer to an array of the names of the methods supported in this version of Libevent. The last element in the array is NULL.

```C
const char *event_base_get_method(const struct event_base *base);
enum event_method_feature event_base_get_features(const struct event_base *base);
```

The `event_base_get_method()` call returns the name of the actual method in use by an `event_base`. The `event_base_get_features()` call returns a bitmask of the features that it supports.

##Deallocating an `event_base`
When you are finished with an `event_base`, you can deallocate it with `event_base_free()`.

```C
void event_base_free(struct event_base *base);
```

Note that this function does not deallocate any of the events that are currently associated with the `event_base`, or close any of their sockets, or free any of their pointers.

##Setting priorities on an `event_base`
Libevent supports setting multiple priorities on an event. By default, though, an `event_base` supports only a single priority level. You can set the number of priorities on an `event_base` by calling `event_base_priority_init()`.

```C
int event_base_priority_init(struct event_base *base, int n_priorities);
```

To find the number of priorities currently supported by a base, you can call `event_base_getnpriorities()`.

```C
int event_base_get_npriorities(struct event_base *base);
```

##Reinitializing an `event_base` after `fork()`
Not all event backends persist cleanly after a call to `fork()`. Thus, if your program uses `fork()` or a related system call in order to start a new process, and you want to continue using an `event_base` after you have forked, you may need to reinitialize it.

```C
int event_reinit(struct event_base *base);
```

##Obsolete `event_base` functions
Older versions of Libevent relied pretty heavily on the idea of a "current" `event_base`. The "current" `event_base` was a global setting shared across all threads. If you forgot to specify which `event_base` you wanted, you got the current one. Since event_bases weren't threadsafe, this could get pretty error-prone.

```C
struct event_base *event_init(void);
```

This function worked like `event_base_new()`, and set the current base to the allocated base. There was no other way to change the current base.

#Running with an event loop
##Running the loop
Once you have an event_base with some events registered(see the next section about how to create and register events), you will want Libevent to wait for events and alert you about them.

```C
#define EVLOOP_ONCE 0x01
#define EVLOOP_NONBLOCK 0x02
#define EVLOOP_NO_EXIT_ON_EMPTY 0x04

int event_base_loop(struct event_base *base, int flags);
```

By default, the `event_base_loop()` function runs an event_base until there are no more events registered in it. To run the loop, it repeatedly checks whether any of the registered events has triggered(for example, if a read event's file descriptor is ready to read, or if a timeout event's timeout is ready to expire). Once this happens, it marks all triggered events as "active", and starts to run them.

You can change the behavior of `event_base_loop()` by setting one or more flags in its flags argument. If `EVLOOP_ONCE` is set, then the loop will wait until some events become active, then run active events until there are no more to run, then return. If `EVLOOP_NONBLOCK` is set, then the loop will not wait for events to trigger: it will only check whether any events are ready to trigger immediately, and run their callbacks if so.
Ordinarily, the loop will exit as soon as it has no pending or active events. You can override this behavior by passing the `EVLOOP_NO_EXIT_ON_EMPTY` flag--for example, if you're going to be adding events from some other thread. If you do set `EVLOOP_NO_EXIT_ON_EMPTY`, the loop will keep running until somebody calls `event_base_loopbreak()`, or calls `event_base_loopexit()`, or an error occurs.
