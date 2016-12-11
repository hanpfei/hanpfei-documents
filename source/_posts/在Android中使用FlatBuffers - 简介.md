---
title: 在Android中使用FlatBuffers - 简介
date: 2016-11-2 20:36:49
tags:
- 网络
---

JSON - 可能每个人都知道这个轻量的数据格式几乎被用在了所有的现代服务器中。相对于过去流行的一些东西，如可怕的XML，它更轻量，更可读，对开发更友好。JSON是语言独立的数据格式，但解析和格式转化，比如转为Java对象，耗费了我们的时间和内存资源。

几天以前，Facebook宣布，在它的Android app中的数据处理部分获得了巨大的性能提升。那与 几乎在整个app中丢弃JSON格式，而用FlatBuffers代替有关。请参考[这篇文章](https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/)来了解一些关于FlatBuffers的基本知识，以及从JSON转换到它的结果。

<!--more-->

尽管结果看上去非常好，但乍一看实现方式不是特别明显。Facebook也没有说太多。那也就是为什么我想在这篇文章中展示我们可以如何使用FlatBuffers。

# FlatBuffers 

简单地说，FlatBuffers是Google的一个跨平台序列化库，特别为游戏开发创建以及，如Facebook所展示的那样，遵循Android中的流畅的及响应式UI的16ms规则。

*但是注意了，在你扔掉一切，将你的所有数据迁移到FlatBuffers之前，请先确认你需要这样做。有时候对性能的影响将是极其微小的，而有时候数据安全性比计算速度上几十毫秒的差异重要得多。*

什么使FlatBuffers如此高效？

 * 由于平坦的二进制缓冲区，访问序列化后的数据时无需解析，即使是对于层次式的数据。多亏于这一点，我们不需要初始化解析器（这意味着构造复杂的字段映射），以及解析数据，这也耗费时间。
 * FlatBuffers数据不需要分配多于缓冲区本身所用大小的内存。我们不需要像在JSON中那样，为解析后的层次式数据分配额外的对象。

要获取真实的数字可以参考关于向FlatBuffers迁移的[facebook的文章](https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/)，或者[Google文档](http://google.github.io/flatbuffers/)。

# 实现

这篇文章将描述在Android app中使用FlatBuffers的最简单的方式：

 * 在app之外的*某个地方*将JSON数据转换为FlatBUffers格式(比如将二进制文件以文件的形式传送，或直接从API返回)。
 * 通过flatc (FlatBuffer编译器) 手动产生数据模型(Java 类)。
 * JSON文件有一些限制 (不能使用null字段，Data格式被解析为一个String)。

可能在未来我们将准备更复杂的方案。

## FlatBuffers编译器

首先我们要先获取**flatc** - FlatBuffers编译器。它可以通过位于Google的[flatbuffers仓库](https://github.com/google/flatbuffers)中的源代码来构建。让我们下载/clone它。整个的构建过程在[FlatBuffers构建](https://google.github.io/flatbuffers/md__building.html)文档中描述。如果你是Mac用户则你需要做的是：

1. 打开下载的代码中的`\{extract directory}\build\XcodeFlatBuffers.xcodeproj`这个文件。
2. 通过点击**Play**按钮或按下⌘ +  R键来运行**flatc** scheme(应该被选为默认选项)。
3. **flatc**可执行文件将出现在工程的根目录。

现在我们可以使用schema编译器（以Java，C#，Python，GO和C++语言）来为给定的模式产生模型类，以及将JSON转换为FlatBuffer二进制文件了。

##模式文件

现在我们必须要准备模式文件了，它定义了我们想要 反-/序列化 的数据结构。这个模式将被flatc用于创建Java模型以及从JSON到FlatBuffer二进制文件的转换。

这是我们的JSON文件的一部分：
```
{
  "repos": [
    {
      "id": 27149168,
      "name": "acai",
      "full_name": "google/acai",
      "owner": {
        "login": "google",
        "id": 1342004,
        ...
        "type": "Organization",
        "site_admin": false
      },
      "private": false,
      "html_url": "https://github.com/google/acai",
      "description": "Testing library for JUnit4 and Guice.",
      ...
      "watchers": 21,
      "default_branch": "master"
    },
    ...
  ]
}
```
repos_json_fragment.json [位于GitHub](https://gist.github.com/frogermcs/804af402e444e76bc012#file-repos_json_fragment-json)

可以在[这里](https://github.com/frogermcs/FlatBuffs/blob/master/flatbuffers/repos_json.json)获取完整的版本。它是通过调用如下的Github API所获数据经过了一点修改的版本：
[https://api.github.com/users/google/repos](https://api.github.com/users/google/repos).

编写FlatBuffer模式文件的[文档](https://google.github.io/flatbuffers/md__schemas.html)非常好，因而在这里我就不再深入说明了。在我们的例子中模式文件也不是非常复杂。我们所要做的所有事情就是创建3个表：`ReposList`，`Repo`和`User`，并定义`root_type`。下面是这个模式文件的重要部分：
```
table ReposList {
    repos : [Repo];
}

table Repo {
    id : long;
    name : string;
    full_name : string;
    owner : User;
    //...
    labels_url : string (deprecated);
    releases_url : string (deprecated);
}

table User {
    login : string;
    id : long;
    avatar_url : string;
    gravatar_id : string;
    //...
    site_admin : bool;
}

root_type ReposList;
```
[repos_schema_fragment.fbs](repos_schema_fragment.fbs hosted with ❤ by GitHub) 位于GitHub

完整的模式文件在[这里](https://github.com/frogermcs/FlatBuffs/blob/master/flatbuffers/repos_schema.fbs)。

# FlatBuffers数据文件

很好，现在我们所需要做的就是将`repos_json.json`转换为FlatBuffers二进制文件，并产生Java模型，其可以以Java友好的方式表示我们的数据（这个操作所需的所有的所有文件可以在我们的repository[获取](https://github.com/frogermcs/FlatBuffs/tree/master/flatbuffers)）：
```
$ ./flatc -j -b repos_schema.fbs repos_json.json
```

如果一切顺利，这将产生如下的文件：
 * repos_json.bin (将被重命名为repos_flat.bin)
 * Repos/Repo.java
 * Repos/ReposList.java
 * Repos/User.java

# Android app

现在让我们来创建我们的示例app，来看下FlatBuffers格式是如何在实际中使用的。这是它的截屏：
![screenshot.png](http://upload-images.jianshu.io/upload_images/1315506-412dbddb94e2d3bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ProgressBar将仅被用于展示不正确的数据处理（在UI线程中）可以如何影响用户界面的流畅度。

我们的app的`app/build.gradle`文件看起来如下面这样：
```
apply plugin: 'com.android.application'
apply plugin: 'com.jakewharton.hugo'

android {
    compileSdkVersion 22
    buildToolsVersion "23.0.0 rc2"

    defaultConfig {
        applicationId "frogermcs.io.flatbuffs"
        minSdkVersion 15
        targetSdkVersion 22
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:22.2.1'
    compile 'com.google.code.gson:gson:2.3.1'
    compile 'com.jakewharton:butterknife:7.0.1'
    compile 'io.reactivex:rxjava:1.0.10'
    compile 'io.reactivex:rxandroid:1.0.0'
}
```
当然，在我们的例子中没必要使用Rx或ButterKnife，但为什么不让这个app更好一点呢😉 ？

让我们把repos_flat.bin和repos_json.json文件放到res/raw/目录下。

这里是[RawDataReader](https://github.com/frogermcs/FlatBuffs/blob/master/app/src/main/java/frogermcs/io/flatbuffs/utils/RawDataReader.java) util，它可以帮我们在Android app中读取raw文件。

最后，把`Repo`，`ReposList`和`User`放在项目源代码中的某个位置。

###FlatBuffers库

FlatBuffers提供了Java库来在Java中直接处理这种数据格式。这里是 [flatbuffers-java-1.2.0-SNAPSHOT.jar](https://github.com/frogermcs/FlatBuffs/blob/master/app/libs/flatbuffers-java-1.2.0-SNAPSHOT.jar)文件。如果你想要手动产生它，你需要移回到下载的FlatBuffers源代码，进入`java/`目录，并使用Maven产生这个库：
```
$ mvn install
```
现在把.jar文件放到你的Android工程的`app/libs/`目录下。

非常好，现在我们需要做的就是实现`MainActivity`类。这是它的完整的代码：
```
public class MainActivity extends AppCompatActivity {

    @Bind(R.id.tvFlat)
    TextView tvFlat;
    @Bind(R.id.tvJson)
    TextView tvJson;

    private RawDataReader rawDataReader;

    private ReposListJson reposListJson;
    private ReposList reposListFlat;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        rawDataReader = new RawDataReader(this);
    }

    @OnClick(R.id.btnJson)
    public void onJsonClick() {
        rawDataReader.loadJsonString(R.raw.repos_json).subscribe(new SimpleObserver<String>() {
            @Override
            public void onNext(String reposStr) {
                parseReposListJson(reposStr);
            }
        });
    }

    private void parseReposListJson(String reposStr) {
        long startTime = System.currentTimeMillis();
        reposListJson = new Gson().fromJson(reposStr, ReposListJson.class);
        for (int i = 0; i < reposListJson.repos.size(); i++) {
            RepoJson repo = reposListJson.repos.get(i);
            Log.d("FlatBuffers", "Repo #" + i + ", id: " + repo.id);
        }
        long endTime = System.currentTimeMillis() - startTime;
        tvJson.setText("Elements: " + reposListJson.repos.size() + ": load time: " + endTime + "ms");
    }

    @OnClick(R.id.btnFlatBuffers)
    public void onFlatBuffersClick() {
        rawDataReader.loadBytes(R.raw.repos_flat).subscribe(new SimpleObserver<byte[]>() {
            @Override
            public void onNext(byte[] bytes) {
                loadFlatBuffer(bytes);
            }
        });
    }

    private void loadFlatBuffer(byte[] bytes) {
        long startTime = System.currentTimeMillis();
        ByteBuffer bb = ByteBuffer.wrap(bytes);
        reposListFlat = frogermcs.io.flatbuffs.model.flat.ReposList.getRootAsReposList(bb);
        for (int i = 0; i < reposListFlat.reposLength(); i++) {
            Repo repos = reposListFlat.repos(i);
            Log.d("FlatBuffers", "Repo #" + i + ", id: " + repos.id());
        }
        long endTime = System.currentTimeMillis() - startTime;
        tvFlat.setText("Elements: " + reposListFlat.reposLength() + ": load time: " + endTime + "ms");

    }
}
```
[MainActivity.java](https://gist.github.com/frogermcs/804af402e444e76bc012#file-mainactivity-java) 位于[GitHub](https://github.com/)

最吸引我们的方法应该是这些：
 * `parseReposListJson(String reposStr)` - 这个方法初始化Gson解析器并将json字符串转换为Java对象。
 * `loadFlatBuffer(byte[] bytes)` - 这个方法将bytes（我们的 repos_flat.bin文件）转为Java对象。

# 结果

现在让我们来可视化JSON和FlatBuffers加载时间及资源消耗的差别。测试是在安装了Android M (beta) 的Nexus 5上进行的。

## 加载时间

测量的操作是转换Java文件并迭代所有的 (90 个) 元素。

JSON - 200ms (范围：180ms - 250ms) - 我们的JSON文件的平均加载时间 （大小：478kB）
FlatBuffers - 5ms (范围：3ms - 10ms) - FlatBuffers二进制文件（大小：362kB）的平均加载时间

记得我们的[16ms规则](https://www.youtube.com/watch?v=CaMTIgxCSqU)么？因为我们在UI线程中调用这些方法。看一下在这个例子中我们的界面的行为如何：

### JSON加载

![json.gif](http://upload-images.jianshu.io/upload_images/1315506-38130dd2acaebf01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### FlatBuffer加载

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1315506-98a87c349eb1e6f9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

看到差别了么？Json加载的ProgressBar停了一会儿，使我们的界面出现了不愉快的情形（操作耗时超过了 16ms）。

## 分配，CPU等等

想做更多地测试？这时可能是尝试[Android Studio 1.3](http://android-developers.blogspot.com/2015/07/get-your-hands-on-android-studio-13.html)和诸如Allocation Tracker，Memory Viewer和Method Tracer这样的新功能的好时机。

## 源代码

这里描述的工程的完整的代码在Github [repository](https://github.com/frogermcs/FlatBuffs)中。你不需要处理FlatBuffers工程 - 你需要的所有东西都在flatbuffers/目录下。

[原文地址](http://frogermcs.github.io/flatbuffers-in-android-introdution/)。
