---
layout: post
title:  "Gentoo 升级后 GNOME 3 崩溃的一次分析"
date:   2015-03-19 14:58:09
categories: jekyll update
---
一次常规系统升级，结果 gnome crash 了，提示 oh no！...
查看日志发现 gnome-shell 段错误 ,并提示 Unable to initialize Clutter

重新编译调试版的 gnome-shell

````
CFLAGS=”-ggdb”
FEATURES=”splitdebug”
```

重新编译 gnome-shell

开启内核 coredump

````
echo “/corefile/core.%e” > /proc/sys/kernel/core_pattern
ulimit -c unlimited
````

重新执行 `gnome-shell  --replace`

拿到 coredump 后，祭出 `gdb $(whitch gnome-shell) --core /corefile/core.gnome-shell`

在 gdb 中执行 bt 看到如下输出

```
(gdb) bt
#0  g_logv (log_domain=0x7f9a14874120 "mutter", log_level=G_LOG_LEVEL_ERROR, format=<optimized out>, args=args@entry=0x7fff99631e98)
    at /tmp/portage/dev-libs/glib-2.42.2/work/glib-2.42.2/glib/gmessages.c:1046
#1  0x00007f9a12f026da in g_log (log_domain=log_domain@entry=0x7f9a14874120 "mutter", log_level=log_level@entry=G_LOG_LEVEL_ERROR,
    format=format@entry=0x7f9a1486c838 "Unable to initialize Clutter.\n")
    at /tmp/portage/dev-libs/glib-2.42.2/work/glib-2.42.2/glib/gmessages.c:1079
#2  0x00007f9a147f53be in meta_clutter_init () at backends/meta-backend.c:469
#3  0x00007f9a1482362a in meta_init () at core/main.c:358
#4  0x0000000000401d5e in main (argc=1, argv=0x7fff99632328) at main.c:429

```

可以发现问题出现在 meta_clutter_init() at backends/meta-backend.c:469

定位到这个函数

```

/**
 * meta_clutter_init: (skip)
 */
void
meta_clutter_init (void)
{
  ClutterSettings *clutter_settings;
  GSource *source;

  meta_create_backend ();

  if (clutter_init (NULL, NULL) != CLUTTER_INIT_SUCCESS)
    g_error (“Unable to initialize Clutter.\n”);

  /*
   * XXX: We cannot handle high dpi scaling yet, so fix the scale to 1
   * for now.
   */
  clutter_settings = clutter_settings_get_default ();
  g_object_set (clutter_settings, “window-scaling-factor”, 1, NULL);

  source = g_source_new (&event_funcs, sizeof (GSource));
  g_source_attach (source, NULL);
  g_source_unref (source);

  meta_backend_post_init (_backend);
}

```

看来错误提示是由 13 行输出的了, `clutter_init()` 这个函数执行失败了

COGL 的目标是提供给开发者高性能访问现代图形硬件能力，提供统一的 API 在 Linux、Windows、OSX、Android 甚至是 Web Brower 上

## 看看 clutter 是个什么东西

![clutter][CLUTTER]


> Clutter is an open source (LGPL 2.1) software library for creating fast, compelling, portable, and dynamic graphical user interfaces. It is a core part of Gnome3, it is used by the GnomeShell, and is supported by the open source community.

> Clutter uses OpenGL for rendering (and optionally OpenGL|ES for use on mobile and embedded platforms), but wraps an easy to use, efficient, flexible API around GL's complexity.

> Clutter enforces no particular user interface style, but provides a rich, generic foundation for higher-level toolkits tailored to specific needs.

clutter 是一个使用 opengl 进行渲染的绘图工具包，提供易用、高效、灵活的 API，



export LD_PRELOAD=/usr/lib64/opengl/ati/lib/libGL.so

设置下 LDPRELOAD 环境变量，Gnome 就可以正常启动了



搜了下系统中 libGL 的实现发现有两个

```
cd /usr
find . -name libGL.so
./usr/lib64/libGL.so
./usr/lib64/opengl/xorg-x11/lib/libGL.so
./usr/lib64/opengl/ati/lib/libGL.so
./usr/lib32/libGL.so
./usr/lib32/opengl/xorg-x11/lib/libGL.so
./usr/lib32/opengl/ati/lib/libGL.so
```

其中 `/usr/lib64/libGL.so` 是一个软链接，指向另外两个之一

使用 `eselect opengl list` 可以看到有两个选项，设置选用不同的项就是更改上面这个软链接的引用



[CLUTTER]: https://wiki.gnome.org/Projects/Clutter?action=AttachFile&do=get&target=clutter-logo-simple.png
[COGL]:http://www.cogl3d.org/hello.html
[CLUTTERHOME]: https://wiki.gnome.org/Projects/Clutter
