---
title: 浅析Android10中的Cgroup
tags: 
- Android性能优化
- Android10
---

Cgroup(Control group)是Linux kernel提供的用于进程或线程(称为task)限制分配系统资源的机制，也是linux容器的基础，可以隔离和限制每个进程的系统资源

对系统而言，task不过是创建新的task struct，分配内存，并找到程序的入口，比如进程的main方法或者线程的routine方法入口开始执行而已

系统资源一般指的是cpu的时间片、RAM、disk

cgroup配置的初始化在系统开机的init过程中执行，挂载点也在这个时候执行

初始化配置代码 `system/core/libprocessgroup/cgrouprc/`

static constexpr const char* CGROUPS_RC_PATH = "/dev/cgroup_info/cgroup.rc";

理解cgroup对性能优化相当重要

android10调整了cgroup机制，使用文件cgroups.json来描述每个controller的基本信息，比如文件系统挂载点，读取这个文件就可以拿到这个controller的信息

使用文件task_profiles.json来描述某个task(进程或线程在Linux中结构体都为task_struct)的资源概要，这个概要的目的是给一个task指定一系列的action来限制资源使用

cgroup代码 `system/core/libprocessgroup/`

cpuset 摘录一下，cpuset是linux kernel的虚拟文件系统，用于控制task的cpu和内存的使用，挂载在目录/dev/cpuset

android10 提供了两种api设置任务profile的方式

`system/core/libprocessgroup/processgroup.cpp`

```cpp
bool SetProcessProfiles(uid_t uid, pid_t pid, const std::vector<std::string>& profiles) {
    return TaskProfiles::GetInstance().SetProcessProfiles(uid, pid, profiles);
}

bool SetTaskProfiles(int tid, const std::vector<std::string>& profiles, bool use_fd_cache) {
    return TaskProfiles::GetInstance().SetTaskProfiles(tid, profiles, use_fd_cache);
}
```

set_sched_policy用于设置某个线程的profiler，在android10上，这个api依然保留，不过新的方式是直接调用SetTaskProfiles

system/core/libprocessgroup/sched_policy.cpp

```cpp
int set_sched_policy(int tid, SchedPolicy policy) {
    if (tid == 0) {
        tid = GetThreadId();
    }
    policy = _policy(policy);

#if POLICY_DEBUG
    char statfile[64];
    char statline[1024];
    char thread_name[255];

    snprintf(statfile, sizeof(statfile), "/proc/%d/stat", tid);
    memset(thread_name, 0, sizeof(thread_name));

    unique_fd fd(TEMP_FAILURE_RETRY(open(statfile, O_RDONLY | O_CLOEXEC)));
    if (fd >= 0) {
        int rc = read(fd, statline, 1023);
        statline[rc] = 0;
        char* p = statline;
        char* q;

        for (p = statline; *p != '('; p++)
            ;
        p++;
        for (q = p; *q != ')'; q++)
            ;

        strncpy(thread_name, p, (q - p));
    }
    switch (policy) {
        case SP_BACKGROUND:
            SLOGD("vvv tid %d (%s)", tid, thread_name);
            break;
        case SP_FOREGROUND:
        case SP_AUDIO_APP:
        case SP_AUDIO_SYS:
        case SP_TOP_APP:
            SLOGD("^^^ tid %d (%s)", tid, thread_name);
            break;
        case SP_SYSTEM:
            SLOGD("/// tid %d (%s)", tid, thread_name);
            break;
        case SP_RT_APP:
            SLOGD("RT  tid %d (%s)", tid, thread_name);
            break;
        default:
            SLOGD("??? tid %d (%s)", tid, thread_name);
            break;
    }
#endif

    switch (policy) {
        case SP_BACKGROUND:
            return SetTaskProfiles(tid, {"SCHED_SP_BACKGROUND"}, true) ? 0 : -1;
        case SP_FOREGROUND:
        case SP_AUDIO_APP:
        case SP_AUDIO_SYS:
            return SetTaskProfiles(tid, {"SCHED_SP_FOREGROUND"}, true) ? 0 : -1;
        case SP_TOP_APP:
            return SetTaskProfiles(tid, {"SCHED_SP_TOP_APP"}, true) ? 0 : -1;
        case SP_SYSTEM:
            return SetTaskProfiles(tid, {"SCHED_SP_SYSTEM"}, true) ? 0 : -1;
        case SP_RT_APP:
            return SetTaskProfiles(tid, {"SCHED_SP_RT_APP"}, true) ? 0 : -1;
        default:
            return SetTaskProfiles(tid, {"SCHED_SP_DEFAULT"}, true) ? 0 : -1;
    }

    return 0;
}
```

所有的profile如下

system/core/libprocessgroup/include/processgroup/sched_policy.h

```cpp
typedef enum {
    SP_DEFAULT = -1,
    SP_BACKGROUND = 0,
    SP_FOREGROUND = 1,
    SP_SYSTEM = 2,
    SP_AUDIO_APP = 3,
    SP_AUDIO_SYS = 4,
    SP_TOP_APP = 5,
    SP_RT_APP = 6,
    SP_RESTRICTED = 7,
    SP_CNT,
    SP_MAX = SP_CNT - 1,
    SP_SYSTEM_DEFAULT = SP_FOREGROUND,
} SchedPolicy;
```

创建一个普通的进程，优先级如何？

退到后台，优先级如何？

进程中的ui线程优先级？普通线程优先级？设置了优先级的线程？

跟踪一下

`frameworks/base/core/jni/android_util_Process.cpp`

```cpp
   public static final native void setThreadPriority(int priority)
            throws IllegalArgumentException, SecurityException;
```

先拿到当前线程的优先级，如果没有改动的话就直接return，如果改动了

如果要设置的优先级比background还低（数值越高，优先级越低），就直接设置这个线程的SchedPolicy为SP_BACKGROUND，如果原来的优先级比background还低，也就是原来优先级比bg低，现在要设置一个比bg高的，那SchedPolicy修改为当前进程的SchedPolicy，所以一个线程的SchedPolicy区间基本上是[SP_BACKGROUND,所属进程的SchedPolicy]

system/core/libutils/Threads.cpp

```cpp

#if defined(__ANDROID__)
int androidSetThreadPriority(pid_t tid, int pri)
{
    int rc = 0;
    int lasterr = 0;
    int curr_pri = getpriority(PRIO_PROCESS, tid);

    if (curr_pri == pri) {
        return rc;
    }

    if (pri >= ANDROID_PRIORITY_BACKGROUND) {
        rc = SetTaskProfiles(tid, {"SCHED_SP_BACKGROUND"}, true) ? 0 : -1;
    } else if (curr_pri >= ANDROID_PRIORITY_BACKGROUND) {
        SchedPolicy policy = SP_FOREGROUND;
        // Change to the sched policy group of the process.
        get_sched_policy(getpid(), &policy);
        rc = SetTaskProfiles(tid, {get_sched_policy_profile_name(policy)}, true) ? 0 : -1;
    }

    if (rc) {
        lasterr = errno;
    }

    if (setpriority(PRIO_PROCESS, tid, pri) < 0) {
        rc = INVALID_OPERATION;
    } else {
        errno = lasterr;
    }

    return rc;
}
```