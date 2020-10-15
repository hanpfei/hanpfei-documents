---
title: 一种 Android 用户事件的自适应分发方法
date: 2020-10-16 22:05:49
categories: 虚拟化
tags:
- 虚拟化
- Android开发
---

Android 设备的远程操作控制中，用户可以在控制端看到远程 Android 设备的屏幕，并通过在控制端执行操作，控制远端 Android 上应用程序及系统的行为。控制端可以是任意的系统及平台，如 Windows，Android 等。
<!--more-->
控制端捕获用户操作的事件，将事件传输到远端的 Android 系统中，控制远端的 Android 系统。

控制端和远程 Android 设备之间的通信中，用户事件通过事件的类型和点击/触摸事件的归一化屏幕坐标描述。在控制端支持多点触控的情况下，用户事件可能同时产生于两个不同的坐标上。用户事件的定义如下面这样：
```
message TouchEvent{
  required ActionMode actionMode = 1;
  required float x1Ratio = 2;
  required float y1Ratio = 3;
  optional float x2Ratio = 4;
  optional float y2Ratio = 5;

  enum ActionMode{
    ACTION_DOWN = 1;
    ACTION_UP = 2;
    ACTION_MOVE = 3;
    ACTION_MOVE2 = 4;
    ACTION_POINTER_DOWN = 5;
    ACTION_POINTER_UP0 = 6;
    ACTION_POINTER_UP1 = 7;
  }
}
```

(x1Ratio, y1Ratio) 和 (x2Ratio, y2Ratio) 分别是事件发生的两个归一化坐标。

在被控制的 Android 设备端接收到事件之后，需要将事件派发进系统，进而传递给应用程序，控制系统及应用程序的行为。将事件派发给系统的一种比较方便的方法是，将事件写入 Android 的输入设备文件中。

Android 设备上事件的派发进程首先需要根据屏幕的尺寸，将事件的归一化坐标转化为该 Android 设备上的实际屏幕坐标，然后写入 Android 设备的输入设备文件中。

Android 系统中的输入设备分为多种不同的类型，分别称为类型 A 与类型 B。即使经过了前面对归一化事件坐标的屏幕坐标转换，转换后的事件坐标依然不能直接写入 Android 的输入设备文件，而是需要按照不同输入设备类型的协议将事件写入 Anroid 的输入设备文件。

Android 系统提供了一个系统应用程序 `getevent`，可以帮助开发者获取 Android 系统中所有输入设备文件支持的输入事件类型等信息，并据此判断输入设备文件的类型。具体而言，可以根据 Android 输入设备支持的 ABS 事件类型来判断输入设备文件的类型，支持如下 ABS 输入事件类型的输入设备文件为类型 A 的输入设备文件：
```
ABS_MT_TOUCH_MAJOR (0x30)
ABS_MT_WIDTH_MAJOR (0x32)
ABS_MT_WIDTH_MINOR (0x33)
ABS_MT_POSITION_X (0x35)
ABS_MT_POSITION_Y (0x36)
ABS_MT_PRESSURE (0x3a)
```

支持如下 ABS 输入事件类型的输入设备文件为类型 B 的输入设备文件：
```
ABS_MT_SLOT (0x2f)
ABS_MT_TOUCH_MAJOR (0x30)
ABS_MT_POSITION_X (0x35)
ABS_MT_POSITION_Y (0x36)
ABS_MT_TRACKING_ID (0x39)
ABS_MT_PRESSURE (0x3a)
```

如 adb shell 进入 Android 6.0 的 X86 版模拟器，执行 `getevent -p` 可以看到如下信息：
```
root@generic_x86:/ # getevent -p                                               
add device 1: /dev/input/event0
  name:     "Power Button"
  events:
    KEY (0001): 0074 
  input props:
    <none>
add device 2: /dev/input/event1
  name:     "qwerty2"
  events:
    KEY (0001): 0001  0002  0003  0004  0005  0006  0007  0008 
                . . . . . .
    ABS (0003): 0000  : value 0, min 0, max 32767, fuzz 0, flat 0, resolution 0
                0001  : value 0, min 0, max 32767, fuzz 0, flat 0, resolution 0
                0002  : value 0, min 0, max 1, fuzz 0, flat 0, resolution 0
                002f  : value 0, min 0, max 9, fuzz 0, flat 0, resolution 0
                0030  : value 0, min 0, max 2147483647, fuzz 0, flat 0, resolution 0
                0035  : value 0, min 0, max 32767, fuzz 0, flat 0, resolution 0
                0036  : value 0, min 0, max 32767, fuzz 0, flat 0, resolution 0
                0039  : value 0, min 0, max 10, fuzz 0, flat 0, resolution 0
                003a  : value 0, min 0, max 256, fuzz 0, flat 0, resolution 0
    SW  (0005): 0000  0002  0004 
  input props:
    <none>
could not get driver version for /dev/input/mice, Not a typewriter
```

在判断出输入设备文件的类型之后，即可执行相应的逻辑来派发输入事件。对于类型 A 的输入设备文件，将事件写入输入设备文件并派发的方法如下：
```
static void single_touch_proto_a(int fd, int pressure, int coord_x, int coord_y) {
    struct timeval tv;
    gettimeofday(&tv, 0);

    struct input_event event;
    memset(&event, 0, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_PRESSURE;
    event.value = pressure;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_POSITION_X;
    event.value = coord_x;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_POSITION_Y;
    event.value = coord_y;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_SYN;
    event.code = SYN_MT_REPORT;
    event.value = 0;
    event.time = tv;
    write(fd, &event, sizeof(event));
}

static void sys_report_proto_a(int fd) {
    struct timeval tv;
    gettimeofday(&tv, 0);

    struct input_event event;
    memset(&event, 0, sizeof(event));

    event.type = EV_SYN;
    event.code = SYN_REPORT;
    event.value = 0;
    event.time = tv;
    write(fd, &event, sizeof(event));
}

//单点触摸
static int write_single_touch_proto_a(int fd, int slot, int coord_x, int coord_y) {
    single_touch_proto_a(fd, 1, coord_x, coord_y);
    sys_report_proto_a(fd);
    return 0;
}

//多点触摸
static int write_multi_touch_proto_a(int fd, int coord_x1, int coord_y1, int coord_x2,
        int coord_y2) {
    single_touch_proto_a(fd, 1, coord_x1, coord_y1);
    single_touch_proto_a(fd, 2, coord_x2, coord_y2);
    sys_report_proto_a(fd);
    return 0;
}

//释放触摸
static int write_up_proto_a(int fd, int slot) {
    struct timeval tv;
    gettimeofday(&tv, 0);
    struct input_event event;
    memset(&event, 0, sizeof(event));

    event.type = EV_SYN;
    event.code = SYN_MT_REPORT;
    event.value = 0;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_SYN;
    event.code = SYN_REPORT;
    event.value = 0;
    event.time = tv;
    write(fd, &event, sizeof(event));
    return 0;
}

struct touch_event_ops {
    int (*write_single_touch)(int fd, int slot, int x, int y);
    int (*write_up)(int fd, int slot);
    int (*write_multi_touch)(int fd, int x1, int y1, int x2, int y2);
};

static struct touch_event_ops proto_a_ops = {
        .write_single_touch = write_single_touch_proto_a,
        .write_up = write_up_proto_a,
        .write_multi_touch = write_multi_touch_proto_a,
};
```

对于类型 B 的输入设备文件，事件写入输入设备文件并派发的方法如下：
```
static int single_touch_proto_b(int fd, int slot, int x, int y) {
    struct timeval tv;
    gettimeofday(&tv, 0);

    struct input_event event;
    memset(&event, 0, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_SLOT;
    event.value = slot;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_TRACKING_ID;
    event.value = slot;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_PRESSURE;
    event.value = 1;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_TOUCH_MINOR;
    event.value = 1;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_TOUCH_MAJOR;
    event.value = 1;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_POSITION_X;
    event.value = x;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_POSITION_Y;
    event.value = y;
    event.time = tv;
    write(fd, &event, sizeof(event));

    return 0;
}

static void sys_report_proto_b(int fd) {
    struct timeval tv;
    gettimeofday(&tv, 0);

    struct input_event event;
    memset(&event, 0, sizeof(event));

    event.type = EV_SYN;
    event.code = SYN_REPORT;
    event.value = 0;
    event.time = tv;
    write(fd, &event, sizeof(event));
}

static int write_single_touch_proto_b(int fd, int slot, int x, int y) {
    single_touch_proto_b(fd, slot, x, y);
    sys_report_proto_b(fd);
    return 0;
}

static int write_multi_touch_proto_b(int fd, int x1, int y1, int x2, int y2) {
    single_touch_proto_b(fd, 0, x1, y1);
    single_touch_proto_b(fd, 1, x2, y2);
    sys_report_proto_b(fd);
    return 0;
}

static int write_up_proto_b(int fd, int slot) {
    struct timeval tv;
    gettimeofday(&tv, 0);

    struct input_event event;
    memset(&event, 0, sizeof(event));

    event.type = EV_SYN;
    event.code = ABS_MT_SLOT;
    event.value = slot;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_ABS;
    event.code = ABS_MT_TRACKING_ID;
    event.value = -1;
    event.time = tv;
    write(fd, &event, sizeof(event));

    event.type = EV_SYN;
    event.code = SYN_REPORT;
    event.value = 0;
    event.time = tv;
    write(fd, &event, sizeof(event));
    return 0;
}

struct touch_event_ops {
    int (*write_single_touch)(int fd, int slot, int x, int y);
    int (*write_up)(int fd, int slot);
    int (*write_multi_touch)(int fd, int x1, int y1, int x2, int y2);
};

static struct touch_event_ops proto_b_ops = {
        .write_single_touch = write_single_touch_proto_b,
        .write_up = write_up_proto_b,
        .write_multi_touch = write_multi_touch_proto_b,
};
```

其它无需支持多种不同类型设备的输入事件派发的应用，可以事先根据 `getevent -p` 的输出判断输入设备的类型，并编写相应的适用于该设备的输入事件派发程序，但这种方法适应性比较差，难以用于需要支持多种不同输入设备类型的应用中。

输入设备文件支持的事件类型信息，不仅仅可以通过 `getevent -p` 命令获得，还可以通过系统调用接口 `ioctl()` 获取。

本发明基于系统调用接口 `ioctl()`，先获得设备文件支持的输入事件类型信息，判断出输入设备的类型信息，并据输入设备的类型，选择不同的事件写入操作集，执行不同的事件派发逻辑。具体方法如下：
```
static const char *get_label(const struct label *labels, int value) {
    while (labels->name && value != labels->value) {
        labels++;
    }
    return labels->name;
}

static int get_input_props(int fd, uint8_t *bits, int bitsSize) {
     int res = ioctl(fd, EVIOCGPROP(bitsSize), bits);
    return res;
}

static bool has_prop(uint8_t *bits, int prop) {
    int index = prop / 8;
    int bitIndex = prop % 8;
    return (bits[index] & (1 << bitIndex)) > 0;
}

static int get_abs_events(int fd, uint8_t *bits, int bits_size) {
    int res = ioctl(fd, EVIOCGBIT(EV_ABS, bits_size), bits);
    return res;
}

static bool support_event(uint8_t *bits, int event) {
    int index = event / 8;
    int bitIndex = event % 8;
    return (bits[index] & (1 << bitIndex)) > 0;
}

static bool support_proto_a_touch_event(uint8_t *absbits) {
    return support_event(absbits, ABS_MT_TOUCH_MAJOR)
            && support_event(absbits, ABS_MT_WIDTH_MAJOR)
            && support_event(absbits, ABS_MT_WIDTH_MINOR)
            && support_event(absbits, ABS_MT_POSITION_X)
            && support_event(absbits, ABS_MT_POSITION_Y)
            && support_event(absbits, ABS_MT_PRESSURE);
}

static bool support_proto_b_touch_event(uint8_t *absbits) {
    return support_event(absbits, ABS_MT_SLOT)
            && support_event(absbits, ABS_MT_TOUCH_MAJOR)
            && support_event(absbits, ABS_MT_POSITION_X)
            && support_event(absbits, ABS_MT_POSITION_Y)
            && support_event(absbits, ABS_MT_TRACKING_ID)
            && support_event(absbits, ABS_MT_PRESSURE);
}

static bool is_touch_dev(const char *device, bool verbose = false) {
    int fd = open(device, O_RDWR);
    if (fd < 0) {
        fprintf(stderr, "could not open %s, %s\n", device, strerror(errno));
        return false;
    }
    if (verbose) {
        print_device_basic_info(device, fd);
    }

    uint8_t bits[INPUT_PROP_CNT / 8];
    bzero(bits, INPUT_PROP_CNT / 8);
    int res = get_input_props(fd, bits, INPUT_PROP_CNT / 8);
    if (verbose) {
        print_input_props(bits, res);
    }

    uint8_t absbits[ABS_CNT / 8];
    bzero(absbits, ABS_CNT / 8);
    res = get_abs_events(fd, absbits, ABS_CNT / 8);
    if (verbose) {
        fprintf(stderr, "ABS event res for %s is %d\n", device, res);
        print_abs_events(fd, absbits, res);
    }

    if (support_proto_a_touch_event(absbits)
            || support_proto_b_touch_event(absbits)) {
        close(fd);
        return true;
    }
    close(fd);
    return false;
}

typedef bool (*check_dev)(const char *device, bool verbose);

static bool scan_dir(const char *dirname, char *devname, check_dev check_func) {
    char *filename;
    DIR *dir;
    struct dirent *de;
    dir = opendir(dirname);
    if (dir == NULL)
        return -1;
    strcpy(devname, dirname);
    filename = devname + strlen(devname);
    *filename++ = '/';
    bool found = false;
    while ((de = readdir(dir))) {
        if (de->d_name[0] == '.'
                && (de->d_name[1] == '\0'
                        || (de->d_name[1] == '.' && de->d_name[2] == '\0')))
            continue;
        strcpy(filename, de->d_name);
        if (check_func(devname, false)) {
            found = true;
            break;
        }
    }
    closedir(dir);
    return found;
}

static struct touch_event_ops proto_a_ops = {
        .write_single_touch = write_single_touch_proto_a,
        .write_up = write_up_proto_a,
        .write_multi_touch = write_multi_touch_proto_a,
};

static struct touch_event_ops proto_b_ops = {
        .write_single_touch = write_single_touch_proto_b,
        .write_up = write_up_proto_b,
        .write_multi_touch = write_multi_touch_proto_b,
};

//打开设备
struct touch_dev_ops * open_touch_dev() {
    const char *device_path = "/dev/input";
    char devname[PATH_MAX];
    struct touch_dev_ops *dev_ops = NULL;
    if (!scan_dir(device_path, devname, &is_touch_dev)) {
        fprintf(stderr, "Can't find touch device.\n");
        return dev_ops;
    }
    fprintf(stderr, "find touch dev %s\n", devname);
    int fd_touch = open(devname, O_RDWR);
    if (fd_touch <= 0) {
        LOGE("open touch %s error:%s", devname, strerror(errno));
        return dev_ops;
    }

    uint8_t absbits[ABS_CNT / 8];
    bzero(absbits, ABS_CNT / 8);
    int res = get_abs_events(fd_touch, absbits, ABS_CNT / 8);

    dev_ops = new touch_dev_ops;
    dev_ops->touch_dev_fd = fd_touch;
    if (support_proto_a_touch_event(absbits)) {
        dev_ops->event_ops = &proto_a_ops;
    } else {
        dev_ops->event_ops = &proto_b_ops;
    }

    return dev_ops;
}
```

本文介绍的 Android 用户事件自适应分发程序，可以同时支持多种不同类型的 Android 输入设备文件。
