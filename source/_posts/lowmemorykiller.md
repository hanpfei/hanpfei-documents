---
title: Android low memory killer 机制
date: 2015-10-04 16:05:49
categories: Android开发
tags:
- Android开发
---

Android中，进程的生命周期都是由系统控制的。即使用户在界面上关掉一个应用，切换到了别的应用，那个应用的进程依然是存在于内存之中的。这样设计的目的是为了下次启动应用能更加快速。当然，随着系统运行时间的增长，内存中的进程可能会越来越多，而可用的内存则将会越来越少。Android Kernel会定时执行一次检查，杀死一些进程，释放掉内存。
<!--more-->
那么，如何来判断，哪些进程是需要杀死的呢？答案就是：low memory killer 机制。

Android 的 low memory killer 是基于 linux 的OOM（out of memory）规则改进而来的。OOM 通过一些比较复杂的评分机制，对进程进行打分，然后将分数高的进程判定为 bad进程，杀死进程并释放内存。OOM 只有当系统内存不足的时候才会启动检查，而 low memory killer 则不仅是在应用程序分配内存发现内存不足时启动检查，它也会定时地进行检查。

Low memory killer 主要是通过进程的 oom_adj 来判定进程的重要程度的。oom_adj 的大小和进程的类型以及进程被调度的次序有关。

Low memory killer 的具体实现在 kernel，比如对于 android 4.4.3 所用的 kernel_3.4，代码在：`linux/kernel/drivers/staging/android/lowmemorykiller.c`。

其原理很简单，‍**在 linux 中，存在一个名为 kswapd 的内核线程，当linux回收存放分页的时候，kswapd 线程将会遍历一张 shrinker 链表，并执行回调，或者某个app分配内存，发现可用内存不足时，则内核会阻塞请求分配内存的进程分配内存的过程，并在该进程中去执行lowmemorykiller来释放内存**。

struct shrinker 的定义在 `linux/kernel/include/linux/shrinker.h`，具体如下：
```
struct shrinker {
    int (*shrink)(struct shrinker *, struct shrink_control *sc);
    int seeks;/* seeks to recreate an obj */
    long batch;/* reclaim batch size, 0 = default */

    /* These are for internal use */
    struct list_head list;
    atomic_long_t nr_in_batch; /* objs pending delete */
};
#define DEFAULT_SEEKS 2 /* A good number if you don't know better. */
extern void register_shrinker(struct shrinker *);
extern void unregister_shrinker(struct shrinker *);
#endif
```

‍所以，**只要注册 shrinker，就可以在内存分页回收时根据规则释放内存**。下面我们来看看lowmemorykiller 的具体实现。首先是定义 shrinker 结构体，lowmem_shrink 为回调函数的指针，当有内存分页回收的时候，获其他有需要的时机，这个函数将会被调用。
```
static struct shrinker lowmem_shrinker = {
    .shrink = lowmem_shrink,
    .seeks = DEFAULT_SEEKS * 16
};
```

初始化模块时进行注册，模块退出时注销。
```
static int __init lowmem_init(void)
{
    register_shrinker(&lowmem_shrinker);
    return 0;
}
static void __exit lowmem_exit(void)
{
    unregister_shrinker(&lowmem_shrinker);
}
```

Android 中，存在着一张内存阈值表，这张阈值表是可以在 init.rc 中进行配置的，合理配置这张表，对于小内存设备有非常重要的作用。我们来看 lowmemorykiller.c 中这张默认的阈值表：
```
static int lowmem_adj[6] = {
    0,
    1,
    6,
    12,
};
static int lowmem_adj_size = 4;
static int lowmem_minfree[6] = {
    3 * 512,/* 6MB */
    2 * 1024,/* 8MB */
    4 * 1024,/* 16MB */
    16 * 1024,/* 64MB */
};
static int lowmem_minfree_size = 4;
```

lowmem_adj 中各项数值代表阈值的警戒级数，lowmem_minfree 代表对应级数的剩余内存。也就是说，当系统的可用内存小于6MB时，警戒级数为0；当系统可用内存小于8M而大于6M时，警戒级数为1；当可用内存小于64M大于16MB时，警戒级数为12。Low memory killer的规则就是根据当前系统的可用内存多少来获取当前的警戒级数，如果进程的oom_adj大于警戒级数并且最大，进程将会被杀死（具有相同omm_adj的进程，则杀死占用内存较多的）。omm_adj越小，代表进程越重要。一些前台的进程，oom_adj会比较小，而后台的服务，omm_adj会比较大，所以当内存不足的时候，Low memory killer必然先杀掉的是后台服务而不是前台的进程。

OK，现在我们来看具体代码，也就是lowmem_shrink这个回调函数，先来：
```
static int lowmem_minfree_size = 4;

static unsigned long lowmem_deathpending_timeout;

#define lowmem_print(level, x...)\
    do {\
        if (lowmem_debug_level >= (level))\
            printk(x);\
    } while (0)

static int lowmem_shrink(struct shrinker *s, struct shrink_control *sc)
{
    struct task_struct *tsk;
    struct task_struct *selected = NULL;
    int rem = 0;
    int tasksize;
    int i;
    int min_score_adj = OOM_SCORE_ADJ_MAX + 1;
    int selected_tasksize = 0;
    int selected_oom_score_adj;
    int array_size = ARRAY_SIZE(lowmem_adj);
    int other_free = global_page_state(NR_FREE_PAGES) - totalreserve_pages;
    int other_file = global_page_state(NR_FILE_PAGES) - global_page_state(NR_SHMEM);

    if (lowmem_adj_size < array_size)
        array_size = lowmem_adj_size;
    if (lowmem_minfree_size < array_size)
        array_size = lowmem_minfree_size;
    for (i = 0; i < array_size; i++) {
        if (other_free < lowmem_minfree[i] &&
            other_file < lowmem_minfree[i]) {
            min_score_adj = lowmem_adj[i];
            break;
        }
    }
    if (sc->nr_to_scan > 0)
        lowmem_print(3, "lowmem_shrink %lu, %x, ofree %d %d, ma %d\n",
            sc->nr_to_scan, sc->gfp_mask, other_free,
            other_file, min_score_adj);
    rem = global_page_state(NR_ACTIVE_ANON) +
        global_page_state(NR_ACTIVE_FILE) +
        global_page_state(NR_INACTIVE_ANON) +
        global_page_state(NR_INACTIVE_FILE);
    if (sc->nr_to_scan <= 0 || min_score_adj == OOM_SCORE_ADJ_MAX + 1) {
        lowmem_print(5, "lowmem_shrink %lu, %x, return %d\n",
             sc->nr_to_scan, sc->gfp_mask, rem);
        return rem;
    }
    selected_oom_score_adj = min_score_adj;

    rcu_read_lock();
    for_each_process(tsk) {
        struct task_struct *p;
        int oom_score_adj;

        if (tsk->flags & PF_KTHREAD)
            continue;

        p = find_lock_task_mm(tsk);
        if (!p)
            continue;

        if (test_tsk_thread_flag(p, TIF_MEMDIE) &&
          time_before_eq(jiffies, lowmem_deathpending_timeout)) {
            task_unlock(p);
            rcu_read_unlock();
            return 0;
        }
        oom_score_adj = p->signal->oom_score_adj;
        if (oom_score_adj < min_score_adj) {
            task_unlock(p);
            continue;
        }
        tasksize = get_mm_rss(p->mm);
        task_unlock(p);
        if (tasksize <= 0)
            continue;
        if (selected) {
            if (oom_score_adj < selected_oom_score_adj)
                continue;
            if (oom_score_adj == selected_oom_score_adj &&
                tasksize <= selected_tasksize)
                continue;
        }
        selected = p;
        selected_tasksize = tasksize;
        selected_oom_score_adj = oom_score_adj;
        lowmem_print(2, "select %d (%s), adj %d, size %d, to kill\n",
             p->pid, p->comm, oom_score_adj, tasksize);
    }
    if (selected) {
        lowmem_print(1, "send sigkill to %d (%s), adj %d, size %d\n",
         selected->pid, selected->comm,
         selected_oom_score_adj, selected_tasksize);
        lowmem_deathpending_timeout = jiffies + HZ;
        send_sig(SIGKILL, selected, 0);
        set_tsk_thread_flag(selected, TIF_MEMDIE);
        rem -= selected_tasksize;
    }
    lowmem_print(4, "lowmem_shrink %lu, %x, return %d\n",
         sc->nr_to_scan, sc->gfp_mask, rem);
    rcu_read_unlock();
    return rem;
}
```

第22行至第35行。通过global_page_state()函数获取系统当前可用的内存大小，然后根据可用内存大小及内存阈值表，来计算系统当前的警戒等级。

第40行，计算rem值。

第44行至第48行这个if-block，发现此时系统的内存状况还是很不错的，于是就什麽也不做，直接返回了。

第49行至第90行，遍历系统中所有的进程，选中将要杀掉的那个进程。第56行，跳过内核线程，内核线程不参加这个杀进程的游戏。第69行至第73行，进程的oom_score_adj小于警戒阈值，则无视。第74行，获取进程所占用的内存大小，RSS值。第78行至第84行，如果这个进程的oom_score_adj小于我们已经选中的那个进程的oom_score_adj，或者这个进程的oom_score_adj等于我们已经选中的那个进程的oom_score_adj，但其所占用的内存大小tasksize小于我们已经选中的那个进程所占用内存大小，则继续寻找下一个进程。第85至第89行，选中正在遍历的这个的进程，更新selected_tasksize为这个进程所占用的内存大小tasksize，更新selected_oom_score_adj为这个进程的oom_score_adj。

第91行至第99行，杀死选中的进程。先是更新lowmem_deathpending_timeout，然后便是给进程发送一个signal SIGKILL，接下来是设置进程的标记为TIF_MEMDIE，最后则是更新一下rem的值。

# adj值的设置

在 lowmemorykiller 的code中我们有看到一个默认的阈值表。如我们前面所提到的那样，各种各样的设备、产品也可以自己定制这个值。这种定制主要是通过导出一些内核变量来实现的，具体代码如下：
```
#ifdef CONFIG_ANDROID_LOW_MEMORY_KILLER_AUTODETECT_OOM_ADJ_VALUES
static int lowmem_oom_adj_to_oom_score_adj(int oom_adj)
{
    if (oom_adj == OOM_ADJUST_MAX)
        return OOM_SCORE_ADJ_MAX;
    else
        return (oom_adj * OOM_SCORE_ADJ_MAX) / -OOM_DISABLE;
}

static void lowmem_autodetect_oom_adj_values(void)
{
    int i;
    int oom_adj;
    int oom_score_adj;
    int array_size = ARRAY_SIZE(lowmem_adj);

    if (lowmem_adj_size < array_size)
        array_size = lowmem_adj_size;

    if (array_size <= 0)
        return;

    oom_adj = lowmem_adj[array_size - 1];
    if (oom_adj > OOM_ADJUST_MAX)
        return;

    oom_score_adj = lowmem_oom_adj_to_oom_score_adj(oom_adj);
    if (oom_score_adj <= OOM_ADJUST_MAX)
        return;

    lowmem_print(1, "lowmem_shrink: convert oom_adj to oom_score_adj:\n");
    for (i = 0; i < array_size; i++) {
        oom_adj = lowmem_adj[i];
        oom_score_adj = lowmem_oom_adj_to_oom_score_adj(oom_adj);
        lowmem_adj[i] = oom_score_adj;
        lowmem_print(1, "oom_adj %d => oom_score_adj %d\n",
             oom_adj, oom_score_adj);
    }
}

static int lowmem_adj_array_set(const char *val, const struct kernel_param *kp)
{
    int ret;

    ret = param_array_ops.set(val, kp);

    /* HACK: Autodetect oom_adj values in lowmem_adj array */
    lowmem_autodetect_oom_adj_values();

    return ret;
}

static int lowmem_adj_array_get(char *buffer, const struct kernel_param *kp)
{
    return param_array_ops.get(buffer, kp);
}

static void lowmem_adj_array_free(void *arg)
{
    param_array_ops.free(arg);
}

static struct kernel_param_ops lowmem_adj_array_ops = {
    .set = lowmem_adj_array_set,
    .get = lowmem_adj_array_get,
    .free = lowmem_adj_array_free,
};

static const struct kparam_array __param_arr_adj = {
    .max = ARRAY_SIZE(lowmem_adj),
    .num = &lowmem_adj_size,
    .ops = &param_ops_int,
    .elemsize = sizeof(lowmem_adj[0]),
    .elem = lowmem_adj,
};
#endif

module_param_named(cost, lowmem_shrinker.seeks, int, S_IRUGO | S_IWUSR);
#ifdef CONFIG_ANDROID_LOW_MEMORY_KILLER_AUTODETECT_OOM_ADJ_VALUES
__module_param_call(MODULE_PARAM_PREFIX, adj,
        &lowmem_adj_array_ops,
        .arr = &__param_arr_adj,
        S_IRUGO | S_IWUSR, -1);
__MODULE_PARM_TYPE(adj, "array of int");
#else
module_param_array_named(adj, lowmem_adj, int, &lowmem_adj_size,
         S_IRUGO | S_IWUSR);
#endif
module_param_array_named(minfree, lowmem_minfree, uint, &lowmem_minfree_size,
         S_IRUGO | S_IWUSR);
module_param_named(debug_level, lowmem_debug_level, uint, S_IRUGO | S_IWUSR);
```

lowmemorykiller 在此处总共导出了4个内核变量，其中两个是内核数组。我们可以在如下位置访问到这些内核变量：`/sys/module/lowmemorykiller/parameters`。在 init.rc 文件中会设置这些内核变量的属性(system/core/rootdir/init.rc)，以便于后面系统对这些值进行修改：
```
281    chown root system /sys/module/lowmemorykiller/parameters/adj
282    chmod 0664 /sys/module/lowmemorykiller/parameters/adj
283    chown root system /sys/module/lowmemorykiller/parameters/minfree
284    chmod 0664 /sys/module/lowmemorykiller/parameters/minfree
```

那通常情况下，比如在Nexus 7或类似的设备上，adj值又是在什麽时候设置的呢？设置的那些值依据是什麽呢，是一个经验值呢还是有什麽算法来算出那些值？

搜索android4.4.3_r1.1的整个codebase，我们发现lowmemorykiller所导出的那些值只在ProcessList.updateOomLevels()方法中被修改了。接着我们就来具体看一下这个方法的实现(实现此方法的文件位置为frameworks/base/services/java/com/android/server/am/ProcessList.java)：
```
    // OOM adjustments for processes in various states:

    // Adjustment used in certain places where we don't know it yet.
    // (Generally this is something that is going to be cached, but we
    // don't know the exact value in the cached range to assign yet.)
    static final int UNKNOWN_ADJ = 16;

    // This is a process only hosting activities that are not visible,
    // so it can be killed without any disruption.
    static final int CACHED_APP_MAX_ADJ = 15;
    static final int CACHED_APP_MIN_ADJ = 9;

    // The B list of SERVICE_ADJ -- these are the old and decrepit
    // services that aren't as shiny and interesting as the ones in the A list.
    static final int SERVICE_B_ADJ = 8;

    // This is the process of the previous application that the user was in.
    // This process is kept above other things, because it is very common to
    // switch back to the previous app.  This is important both for recent
    // task switch (toggling between the two top recent apps) as well as normal
    // UI flow such as clicking on a URI in the e-mail app to view in the browser,
    // and then pressing back to return to e-mail.
    static final int PREVIOUS_APP_ADJ = 7;

    // This is a process holding the home application -- we want to try
    // avoiding killing it, even if it would normally be in the background,
    // because the user interacts with it so much.
    static final int HOME_APP_ADJ = 6;

    // This is a process holding an application service -- killing it will not
    // have much of an impact as far as the user is concerned.
    static final int SERVICE_ADJ = 5;

    // This is a process with a heavy-weight application.  It is in the
    // background, but we want to try to avoid killing it.  Value set in
    // system/rootdir/init.rc on startup.
    static final int HEAVY_WEIGHT_APP_ADJ = 4;

    // This is a process currently hosting a backup operation.  Killing it
    // is not entirely fatal but is generally a bad idea.
    static final int BACKUP_APP_ADJ = 3;

    // This is a process only hosting components that are perceptible to the
    // user, and we really want to avoid killing them, but they are not
    // immediately visible. An example is background music playback.
    static final int PERCEPTIBLE_APP_ADJ = 2;

    // This is a process only hosting activities that are visible to the
    // user, so we'd prefer they don't disappear.
    static final int VISIBLE_APP_ADJ = 1;

    // This is the process running the current foreground app.  We'd really
    // rather not kill it!
    static final int FOREGROUND_APP_ADJ = 0;

    // This is a system persistent process, such as telephony.  Definitely
    // don't want to kill it, but doing so is not completely fatal.
    static final int PERSISTENT_PROC_ADJ = -12;

    // The system process runs at the default adjustment.
    static final int SYSTEM_ADJ = -16;

    // Special code for native processes that are not being managed by the system (so
    // don't have an oom adj assigned by the system).
    static final int NATIVE_ADJ = -17;

    // Memory pages are 4K.
    static final int PAGE_SIZE = 4*1024;

    // These are the various interesting memory levels that we will give to
    // the OOM killer.  Note that the OOM killer only supports 6 slots, so we
    // can't give it a different value for every possible kind of process.
    private final int[] mOomAdj = new int[] {
            FOREGROUND_APP_ADJ, VISIBLE_APP_ADJ, PERCEPTIBLE_APP_ADJ,
            BACKUP_APP_ADJ, CACHED_APP_MIN_ADJ, CACHED_APP_MAX_ADJ
    };
    // These are the low-end OOM level limits.  This is appropriate for an
    // HVGA or smaller phone with less than 512MB.  Values are in KB.
    private final long[] mOomMinFreeLow = new long[] {
            8192, 12288, 16384,
            24576, 28672, 32768
    };
    // These are the high-end OOM level limits.  This is appropriate for a
    // 1280x800 or larger screen with around 1GB RAM.  Values are in KB.
    private final long[] mOomMinFreeHigh = new long[] {
            49152, 61440, 73728,
            86016, 98304, 122880
    };
    // The actual OOM killer memory levels we are using.
    private final long[] mOomMinFree = new long[mOomAdj.length];

    private final long mTotalMemMb;

    private void updateOomLevels(int displayWidth, int displayHeight, boolean write) {
        // Scale buckets from avail memory: at 300MB we use the lowest values to
        // 700MB or more for the top values.
        float scaleMem = ((float)(mTotalMemMb-300))/(700-300);

        // Scale buckets from screen size.
        int minSize = 480*800;  //  384000
        int maxSize = 1280*800; // 1024000  230400 870400  .264
        float scaleDisp = ((float)(displayWidth*displayHeight)-minSize)/(maxSize-minSize);
        if (false) {
            Slog.i("XXXXXX", "scaleMem=" + scaleMem);
            Slog.i("XXXXXX", "scaleDisp=" + scaleDisp + " dw=" + displayWidth
                    + " dh=" + displayHeight);
        }

        StringBuilder adjString = new StringBuilder();
        StringBuilder memString = new StringBuilder();

        float scale = scaleMem > scaleDisp ? scaleMem : scaleDisp;
        if (scale < 0) scale = 0;
        else if (scale > 1) scale = 1;
        int minfree_adj = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAdjust);
        int minfree_abs = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAbsolute);
        if (false) {
            Slog.i("XXXXXX", "minfree_adj=" + minfree_adj + " minfree_abs=" + minfree_abs);
        }

        for (int i=0; i<mOomAdj.length; i++) {
            long low = mOomMinFreeLow[i];
            long high = mOomMinFreeHigh[i];
            mOomMinFree[i] = (long)(low + ((high-low)*scale));
        }

        if (minfree_abs >= 0) {
            for (int i=0; i<mOomAdj.length; i++) {
                mOomMinFree[i] = (long)((float)minfree_abs * mOomMinFree[i] / mOomMinFree[mOomAdj.length - 1]);
            }
        }

        if (minfree_adj != 0) {
            for (int i=0; i<mOomAdj.length; i++) {
                mOomMinFree[i] += (long)((float)minfree_adj * mOomMinFree[i] / mOomMinFree[mOomAdj.length - 1]);
                if (mOomMinFree[i] < 0) {
                    mOomMinFree[i] = 0;
                }
            }
        }

        // The maximum size we will restore a process from cached to background, when under
        // memory duress, is 1/3 the size we have reserved for kernel caches and other overhead
        // before killing background processes.
        mCachedRestoreLevel = (getMemLevel(ProcessList.CACHED_APP_MAX_ADJ)/1024) / 3;

        for (int i=0; i<mOomAdj.length; i++) {
            if (i > 0) {
                adjString.append(',');
                memString.append(',');
            }
            adjString.append(mOomAdj[i]);
            memString.append((mOomMinFree[i]*1024)/PAGE_SIZE);
        }

        // Ask the kernel to try to keep enough memory free to allocate 3 full
        // screen 32bpp buffers without entering direct reclaim.
        int reserve = displayWidth * displayHeight * 4 * 3 / 1024;
        int reserve_adj = Resources.getSystem().getInteger(com.android.internal.R.integer.config_extraFreeKbytesAdjust);
        int reserve_abs = Resources.getSystem().getInteger(com.android.internal.R.integer.config_extraFreeKbytesAbsolute);

        if (reserve_abs >= 0) {
            reserve = reserve_abs;
        }

        if (reserve_adj != 0) {
            reserve += reserve_adj;
            if (reserve < 0) {
                reserve = 0;
            }
        }

        //Slog.i("XXXXXXX", "******************************* MINFREE: " + memString);
        if (write) {
            writeFile("/sys/module/lowmemorykiller/parameters/adj", adjString.toString());
            writeFile("/sys/module/lowmemorykiller/parameters/minfree", memString.toString());
            SystemProperties.set("sys.sysctl.extra_free_kbytes", Integer.toString(reserve));
        }
        // GB: 2048,3072,4096,6144,7168,8192
        // HC: 8192,10240,12288,14336,16384,20480
    }
```

adj值是一组写死的固定的值，具体可以参考mOomAdj的定义。

第123行至第127行，是第一轮计算各个adj所对应的minimum free memory阈值。计算各个值的算式为(long)(low + ((high-low)*scale))。low值和high值都是预定义的固定的经验值，比较关键的是那个scale值。在前面计算scale的部分，我们可以看到，它会先计算一个memory的scale值(为((float)(mTotalMemMb-300))/(700-300))，再计算一个屏幕分辨率的scale值(((float)(displayWidth*displayHeight)-minSize)/(maxSize-minSize))，首先取scale值为这两个scale值中较大的那一个。再然后是一些容错处理，将scale值限制在0～1之间，以防止设备内存小于300MB，同时设备分辨率小于480*800；或者，设备内存大于700MB，或设备分辨率大于1280*800的情况出现时，出现太不合理的阈值。

第129行至133行，是第二轮计算各个adj所对应的minimum free memory阈值。计算各个值的算式为(long)((float)minfree_abs * mOomMinFree[i] / mOomMinFree[mOomAdj.length - 1])。此处给了各设备对low memory阈值进行定制的机会。各个设备可以在framework的config文件中定义config_lowMemoryKillerMinFreeKbytesAbsolute，以指定最大的adj所对应的free memory阈值，其他各个adj所对应的free memory阈值将依比例算出。

第135行至第142行，是第三轮计算各个adj所对应的minimum free memory阈值。计算各个值的算式为mOomMinFree[i] += (long)((float)minfree_adj * mOomMinFree[i] / mOomMinFree[mOomAdj.length - 1])。此处是给特定设备微调low memory的阈值提供机会。不过我们仔细来看第二轮和第三轮的计算，这两轮计算似乎是可以合并为：(long)((float)(minfree_abs + minfree_adj)  * mOomMinFree[i] / mOomMinFree[mOomAdj.length - 1])。

后面则是构造适合设置给lowmemorykiller导出参数的字符串，并最终将构造的这组参数设置进去。

那这个ProcessList.updateOomLevels()方法又是在什麽时候会被调用到呢？是在ProcessList.applyDisplaySize()方法中：
```
    void applyDisplaySize(WindowManagerService wm) {
        if (!mHaveDisplaySize) {
            Point p = new Point();
            wm.getBaseDisplaySize(Display.DEFAULT_DISPLAY, p);
            if (p.x != 0 && p.y != 0) {
                updateOomLevels(p.x, p.y, true);
                mHaveDisplaySize = true;
            }
        }
    }
```

在这个方法中，当能够从WMS中获取有效的屏幕分辨率时，会去更新oom levels，并且更新之后就不会再次去更新。我们再来追查ProcessList.applyDisplaySize()，是在ActivityManagerService.updateConfiguration()：
```
    public void updateConfiguration(Configuration values) {
        enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                "updateConfiguration()");

        synchronized(this) {
            if (values == null && mWindowManager != null) {
                // sentinel: fetch the current configuration from the window manager
                values = mWindowManager.computeNewConfiguration();
            }

            if (mWindowManager != null) {
                mProcessList.applyDisplaySize(mWindowManager);
            }

            final long origId = Binder.clearCallingIdentity();
            if (values != null) {
                Settings.System.clearConfiguration(values);
            }
            updateConfigurationLocked(values, null, false, false);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

我们总结一下OOM levels的更新：时机是在系统启动后，第一个configuration change的消息到AMS的时候；adj值为一组固定的预定义的值；各个adj所对应的min free阈值则根据系统的内存大小和屏幕的分辨率计算得出，各个设备还可以通过framework的config config_lowMemoryKillerMinFreeKbytesAdjust和config_lowMemoryKillerMinFreeKbytesAbsolute来定制最终各个adj所对应的min free阈值。

我们可以根据系统所吐出来的log看一下，事实是否如我们上面对code的分析那样。由某个设备抓到的系统内核吐出来的log，可以发现如下的这些行(只出现一次)：
```
I/KERNEL  (  609): [   14.064905] lowmemorykiller: lowmem_shrink: convert oom_adj to oom_score_adj:
I/KERNEL  (  609): [   14.064932] lowmemorykiller: oom_adj 0 => oom_score_adj 0
I/KERNEL  (  609): [   14.064938] lowmemorykiller: oom_adj 1 => oom_score_adj 58
I/KERNEL  (  609): [   14.064942] lowmemorykiller: oom_adj 2 => oom_score_adj 117
I/KERNEL  (  609): [   14.064947] lowmemorykiller: oom_adj 3 => oom_score_adj 176
I/KERNEL  (  609): [   14.064951] lowmemorykiller: oom_adj 9 => oom_score_adj 529
I/KERNEL  (  609): [   14.064956] lowmemorykiller: oom_adj 15 => oom_score_adj 1000
```

由前面lowmemorykiller的code不难看出，这些log在lowmem_autodetect_oom_adj_values()函数中吐出，是在内核空间吐出来的。吐出这些log的进程的pid为609。我们再来看一下这个进程到底是何方神圣：
```
desktop:~/develop_tools$ adb shell ps | grep 609
system    609   235   1031532 94240 ffffffff ffffe430 S system_server
```

谜底揭晓，是system_server。不难推测出来，这应当是在系统启动之后，AMS第一次接到configuration change的消息，去更新OOM level，lowmemorykiller的lowmem_autodetect_oom_adj_values()函数被调到，更新lowmemorykiller的lowmem_adj时而吐出来的。与我们前面对AMS code的分析基本一致。

还需要我们厘清的问题，lowmemorykiller的lowmem_shrink具体在什麽时机会被执行？

Done。

