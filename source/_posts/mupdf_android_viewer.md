---
title: mupdf-android-viewer 设计与实现浅析
date: 2018-06-11 19:05:49
categories: Android开发
tags:
- Android开发
---

目前在 Android 应用开发中，可用的 PDF 文档展示的开源项目好几个，最为方便的是 [AndroidPdfViewer](https://github.com/barteksc/AndroidPdfViewer)，它基于 [PdfiumAndroid](https://github.com/mshockwave/PdfiumAndroid) 开发而来，而后者则是由 AOSP 中的 pdfium 封转而来。另外一个 PDF 文档显示的开源项目 mupdf 也非常强大。本文简单分析 MuPDF 库的 Android 封装。
<!--more-->
MuPDF 是一个用于查看和改变 PDF，XPS 和 E-book 文档的开源软件框架。它为多个不同的平台提供了查看器，命令行工具，以及软件库以用于构建工具和应用程序。

mupdf-android-viewer 是 MuPDF 为 Android 平台提供的查看器，它的代码可以通过 Git 下载得到：
```
$ git clone git://git.ghostscript.com/mupdf-android-viewer.git
```

mupdf-android-viewer 工程的目录结构很简单：
```
mupdf-android-viewer$ tree
.
├── app
│   ├── build.gradle
│   └── src
│       └── main
│           ├── AndroidManifest.xml
│           ├── java
│           │   └── com
│           │       └── artifex
│           │           └── mupdf
│           │               └── viewer
│           │                   └── app
│           │                       └── LibraryActivity.java
│           └── res
│               ├── drawable
│               │   └── ic_mupdf.xml
│               └── values
│                   └── strings.xml
├── build.gradle
├── COPYING
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── jni
├── lib
│   ├── build.gradle
│   └── src
│       └── main
│           ├── AndroidManifest.xml
│           ├── java
│           │   └── com
│           │       └── artifex
│           │           └── mupdf
│           │               └── viewer
│           │                   ├── CancellableAsyncTask.java
│           │                   ├── CancellableTaskDefinition.java
│           │                   ├── DocumentActivity.java
│           │                   ├── MuPDFCancellableTaskDefinition.java
│           │                   ├── MuPDFCore.java
│           │                   ├── OutlineActivity.java
│           │                   ├── PageAdapter.java
│           │                   ├── PageView.java
│           │                   ├── ReaderView.java
│           │                   ├── SearchTask.java
│           │                   ├── SearchTaskResult.java
│           │                   └── Stepper.java
│           └── res
│               ├── drawable
│               │   ├── button.xml
│               │   ├── ic_chevron_left_white_24dp.xml
│               │   ├── ic_chevron_right_white_24dp.xml
│               │   ├── ic_close_white_24dp.xml
│               │   ├── ic_link_white_24dp.xml
│               │   ├── ic_search_white_24dp.xml
│               │   ├── ic_toc_white_24dp.xml
│               │   ├── page_indicator.xml
│               │   ├── seek_line.xml
│               │   └── seek_thumb.xml
│               ├── layout
│               │   └── document_activity.xml
│               └── values
│                   ├── colors.xml
│                   └── strings.xml
├── Makefile
├── README
└── settings.gradle

27 directories, 41 files
```

其中 `settings.gradle` 内容如下：
```
include ':jni'
include ':lib'
include ':app'
```

mupdf-android-viewer 主要由三个模块组成，分别为 app，lib 和 jni，app 是 Android MuPDF 查看器的应用程序主工程，jni 是 MuPDF 的核心库，lib 是基于 MuPDF 核心库封装出来的用于显示 PDF 文档的 UI 控件库。Android MuPDF 查看器应用程序基于 lib 库构建而成。

MuPDF 的 核心库可以有两个来源，一是远程 Maven 仓库，其中 Maven 仓库的地址为 `http://maven.ghostscript.com/`，Android 核心库的坐标为 `com.artifex.mupdf:fitz:1.13.0`，这可以从 `mupdf-android-viewer/build.gradle` 和 `mupdf-android-viewer/lib/build.gradle` 文件中看出来；二是由源码编译而来，编译的方法如 [为 Android 编译 MuPDF 查看器](https://www.wolfcstech.com/2018/06/06/android-build-viewer/) 一文所示，MuPDF 的 核心库的源码下载之后位于 `mupdf-android-viewer/jni` 目录下。

MuPDF 的 Android 核心库又由三个部分组成，分别为它所依赖的第三方库，MuPDF 本地层库和 MuPDF 本地层库的 JNI 封装。其中 MuPDF 本地层库的 JNI 封装位于 `mupdf-android-viewer/jni/libmupdf/platform/java`，MuPDF 本地层库位于 `mupdf-android-viewer/jni/libmupdf/source`，依赖的第三方库则位于 `mupdf-android-viewer/jni/libmupdf/thirdparty`。

MuPDF Android 查看器的整体组件结构如下图所示：

![截图_2018-06-07_17-18-34.png](https://www.wolfcstech.com/images/1315506-50e57820c1107be9.png)

MuPDF UI 控件库的代码位于 `mupdf-android-viewer/lib`，它连接了上层的 Android 应用程序和下层的 MuPDF 核心库，其中 MuPDFCore 类封装了 MuPDF 核心库用 JNI 封装的底层 MuPDF 库的 Java 接口；`ReaderView`、`PageAdapter`、`PageView`、`SearchTask` 和 `SearchTaskResult` 基于 MuPDFCore 类实现 Android 应用程序的 UI 组件，以方便在 Android 应用中查看 PDF 文件；DocumentActivity 和 `OutlineActivity` 是两个 Activity，这两个页面分别用于显示 PDF 文档及 PDF 文档的目录。

MuPDFCore 类的定义如下：
```
package com.artifex.mupdf.viewer;

import com.artifex.mupdf.fitz.Cookie;
import com.artifex.mupdf.fitz.Document;
import com.artifex.mupdf.fitz.Outline;
import com.artifex.mupdf.fitz.Page;
import com.artifex.mupdf.fitz.Link;
import com.artifex.mupdf.fitz.DisplayList;
import com.artifex.mupdf.fitz.Rect;
import com.artifex.mupdf.fitz.RectI;
import com.artifex.mupdf.fitz.Matrix;
import com.artifex.mupdf.fitz.android.AndroidDrawDevice;

import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.PointF;
import android.graphics.RectF;

import java.util.ArrayList;

public class MuPDFCore
{
	private int resolution;
	private Document doc;
	private Outline[] outline;
	private int pageCount = -1;
	private int currentPage;
	private Page page;
	private float pageWidth;
	private float pageHeight;
	private DisplayList displayList;

	public MuPDFCore(String filename) {
		doc = Document.openDocument(filename);
		pageCount = doc.countPages();
		resolution = 160;
		currentPage = -1;
	}

	public MuPDFCore(byte buffer[], String magic) {
		doc = Document.openDocument(buffer, magic);
		pageCount = doc.countPages();
		resolution = 160;
		currentPage = -1;
	}

	public String getTitle() {
		return doc.getMetaData(Document.META_INFO_TITLE);
	}

	public int countPages() {
		return pageCount;
	}

	private synchronized void gotoPage(int pageNum) {
		/* TODO: page cache */
		if (pageNum > pageCount-1)
			pageNum = pageCount-1;
		else if (pageNum < 0)
			pageNum = 0;
		if (pageNum != currentPage) {
			currentPage = pageNum;
			if (page != null)
				page.destroy();
			page = null;
			if (displayList != null)
				displayList.destroy();
			displayList = null;
			page = doc.loadPage(pageNum);
			Rect b = page.getBounds();
			pageWidth = b.x1 - b.x0;
			pageHeight = b.y1 - b.y0;
		}
	}

	public synchronized PointF getPageSize(int pageNum) {
		gotoPage(pageNum);
		return new PointF(pageWidth, pageHeight);
	}

	public synchronized void onDestroy() {
		if (displayList != null)
			displayList.destroy();
		displayList = null;
		if (page != null)
			page.destroy();
		page = null;
		if (doc != null)
			doc.destroy();
		doc = null;
	}

	public synchronized void drawPage(Bitmap bm, int pageNum,
			int pageW, int pageH,
			int patchX, int patchY,
			int patchW, int patchH,
			Cookie cookie) {
		gotoPage(pageNum);

		if (displayList == null)
			displayList = page.toDisplayList(false);

		float zoom = resolution / 72;
		Matrix ctm = new Matrix(zoom, zoom);
		RectI bbox = new RectI(page.getBounds().transform(ctm));
		float xscale = (float)pageW / (float)(bbox.x1-bbox.x0);
		float yscale = (float)pageH / (float)(bbox.y1-bbox.y0);
		ctm.scale(xscale, yscale);

		AndroidDrawDevice dev = new AndroidDrawDevice(bm, patchX, patchY);
		displayList.run(dev, ctm, cookie);
		dev.destroy();
	}

	public synchronized void updatePage(Bitmap bm, int pageNum,
			int pageW, int pageH,
			int patchX, int patchY,
			int patchW, int patchH,
			Cookie cookie) {
		drawPage(bm, pageNum, pageW, pageH, patchX, patchY, patchW, patchH, cookie);
	}

	public synchronized Link[] getPageLinks(int pageNum) {
		gotoPage(pageNum);
		return page.getLinks();
	}

	public synchronized RectF[] searchPage(int pageNum, String text) {
		gotoPage(pageNum);
		Rect[] rs = page.search(text);
		RectF[] rfs = new RectF[rs.length];
		for (int i=0; i < rs.length; ++i)
			rfs[i] = new RectF(rs[i].x0, rs[i].y0, rs[i].x1, rs[i].y1);
		return rfs;
	}

	public synchronized boolean hasOutline() {
		if (outline == null)
			outline = doc.loadOutline();
		return outline != null;
	}

	private void flattenOutlineNodes(ArrayList<OutlineActivity.Item> result, Outline list[], String indent) {
		for (Outline node : list) {
			if (node.title != null)
				result.add(new OutlineActivity.Item(indent + node.title, node.page));
			if (node.down != null)
				flattenOutlineNodes(result, node.down, indent + "    ");
		}
	}

	public synchronized ArrayList<OutlineActivity.Item> getOutline() {
		ArrayList<OutlineActivity.Item> result = new ArrayList<OutlineActivity.Item>();
		flattenOutlineNodes(result, outline, "");
		return result;
	}

	public synchronized boolean needsPassword() {
		return doc.needsPassword();
	}

	public synchronized boolean authenticatePassword(String password) {
		return doc.authenticatePassword(password);
	}
}
```

由  MuPDFCore 类的实现不难看出，MuPDF 核心库在查看 PDF 文件方面所提供的功能如下：

**1.** ：解析 PDF 文档。这主要是在 `MuPDFCore` 类的构造函数里完成的，通过 `com.artifex.mupdf.fitz.Document` 类的 `openDocument()` 方法执行。PDF 文档解析完成之后，可以借助于 `com.artifex.mupdf.fitz.Document` 的对象获得 PDF 文档的标题，页数等信息。
**2.** ：跳转到 PDF 文档的特定页上。这主要通过 `gotoPage()` 方法完成。这将为目标页创建 `com.artifex.mupdf.fitz.Page` 对象，并可以获得这一页绘制之后的尺寸。
**3.** ：将 PDF 文档特定页的内容绘制到 Bitmap 上。绘制通过 `com.artifex.mupdf.fitz.Page` 对象创建的 `com.artifex.mupdf.fitz.DisplayList` 对象完成。在绘制的时候，还会根据 Bitmap 的尺寸和 PDF 目标文档的尺寸做一定的放缩。
**4.** ：在 PDF 文档中搜索特定的字符串。在 PDF 文档的特定页执行的搜索操作以找到的文本在该页中占据的矩形框的数组的形式返回。
**5.** ：获得 PDF 文档的目录信息。这主要是通过 `hasOutline()` 和 `getOutline()` 方法实现的。其中目录项由 `OutlineActivity.Item` 描述：
```
public class OutlineActivity extends ListActivity
{
	public static class Item implements Serializable {
		public String title;
		public int page;
		public Item(String title, int page) {
			this.title = title;
			this.page = page;
		}
		public String toString() {
			return title;
		}
	}
```
`OutlineActivity.Item` 描述了特定目录项的标题及该目录项在 PDF 文档中的页码。
**6.** ：判断查看 PDF 文档是否需要密码，以及配置认证密码。这主要是通过 `needsPassword()` 和 `authenticatePassword(String password)` 方法实现的。

MuPDF 核心库提供的最主要的功能是解析 PDF 文档，跳转到特定页，并将该页的实际内容绘制到 Bitmap 上。基于 MuPDF 核心库实现 Android 应用程序的 UI 控件，最主要要做的即是创建、管理并展示 PDF 文档中各页的 Bitmap 图片。
