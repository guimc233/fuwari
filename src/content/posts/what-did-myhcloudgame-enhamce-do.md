---
title: 一些魔改米家云游戏的笔记
published: 2025-12-23
description: ''
image: ''
tags: ["Mihoyo", "逆向", "开发"]
category: '开发'
draft: false 
lang: ''
---
## 自动复制URL
尝试搜索方法名 `setUrl`, 可以看到有如下实现  

![Image](https://speedtest.lightningnur.win/i/2025/12/23/z9zi7g.jpg)  

打开长得最像 Webview 组件的类 `ContentWebViewHolder`, 开始对它的 `setUrl` 方法动刀    
首先把它的动态补丁干掉    

```smali title="ContentWebViewHolder.smsli" del={6, 15} ins={7, 16}
.method public setUrl(Ljava/lang/String;)V
    .registers 9

    sget-object v0, Lcom/miHoYo/sdk/webview/framework/ContentWebViewHolder;->m__m:Lcom/mihoyo/hotfix/runtime/patch/RuntimeDirector;

    if-eqz v0, :cond_0
    goto :cond_0

    const/16 v1, 0xa

    invoke-interface {v0, v1}, Lcom/mihoyo/hotfix/runtime/patch/RuntimeDirector;->isRedirect(I)Z

    move-result v2

    if-eqz v2, :cond_0
    goto :cond_0

    const/4 v2, 0x1

    ...
```

接下来往上找, 可以看到一个 Activity    

```smali title="ContentWebViewHolder.smsli"
.field public activity:Landroid/app/Activity;
```

我们可以用它来复制内容到剪切板和弹出 Toast    
回到 `setUrl` 方法, 插入一些代码来执行上面的操作    

```smali title="ContentWebViewHolder.smsli" ins={6-48}
    ...

    :cond_0
    iput-object p1, p0, Lcom/miHoYo/sdk/webview/framework/ContentWebViewHolder;->url:Ljava/lang/String;

    iget-object v0, p0, Lcom/miHoYo/sdk/webview/framework/ContentWebViewHolder;->activity:Landroid/app/Activity;

    if-nez v0, :cond_1

    const-string v1, "clipboard"

    invoke-virtual {v0, v1}, Landroid/content/Context;->getSystemService(Ljava/lang/String;)Ljava/lang/Object;

    move-result-object v1

    check-cast v1, Landroid/content/ClipboardManager;

    const-string v2, "URL"

    invoke-static {v2, p1}, Landroid/content/ClipData;->newPlainText(Ljava/lang/CharSequence;Ljava/lang/CharSequence;)Landroid/content/ClipData;

    move-result-object v2

    invoke-virtual {v1, v2}, Landroid/content/ClipboardManager;->setPrimaryClip(Landroid/content/ClipData;)V

    new-instance v1, Ljava/lang/StringBuilder;

    invoke-direct {v1}, Ljava/lang/StringBuilder;-><init>()V

    const-string v2, "url copied: "

    invoke-virtual {v1, v2}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v1, p1}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    invoke-virtual {v1}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v1

    const/4 v2, 0x1

    invoke-static {v0, v1, v2}, Landroid/widget/Toast;->makeText(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;

    move-result-object v0

    invoke-virtual {v0}, Landroid/widget/Toast;->show()V

    :cond_1
    const-string v0, "no_joypad_close"

    invoke-direct {p0, p1, v0}, Lcom/miHoYo/sdk/webview/framework/ContentWebViewHolder;->getUrlQuery(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;

    move-result-object p1

    const-string v0, "1"

    ...
```

这样就完成啦~!    

## 显示具体排队队列
依旧使用 MT文件管理器, 搜索 `.source "EnqueueDialog.kt"`, 可以看到类似这样的搜索结果

![Image2](https://speedtest.lightningnur.win/i/2025/12/23/zkaxbu.jpg)

打开第一个搜索结果, 转换为 Java, 可以在方法 `K0` \(*不同版本重命名表可能不同*\) 发现

```java title="a.java" {8} {12} {16} {20} {24}
public final void K0(DispatchQueueInfo dispatchQueueInfo, boolean z) {
    RuntimeDirector runtimeDirector = m__m;
    if (runtimeDirector != null && runtimeDirector.isRedirect("-5ef8a324", 3)) {
        runtimeDirector.invocationDispatch("-5ef8a324", 3, this, new Object[]{dispatchQueueInfo, Boolean.valueOf(z)});
        return;
    }
    p0 waitingTimeStr = dispatchQueueInfo.getWaitingTimeStr();
    String queued_progress_display = dispatchQueueInfo.getQueued_progress_display();
    if (queued_progress_display == null) {
        queued_progress_display = "";
    }
    if (l0.g(queued_progress_display, QueuedProgressDisplayType.POSITION_TIME.getValue())) {
        // 略
        return;
    }
    if (l0.g(queued_progress_display, QueuedProgressDisplayType.LENGTH_TIME.getValue())) {
        // 略
        return;
    }
    if (l0.g(queued_progress_display, QueuedProgressDisplayType.TIME.getValue())) {
        // 略
        return;
    }
    if (l0.g(queued_progress_display, QueuedProgressDisplayType.POSITION_LENGTH_TIME.getValue())) {
        // 略
    }
    // 略
}
```

我们不难发现, `DispatchQueueInfo.getQueued_progress_display()` 决定了排队信息的显示    
并且可以大胆猜测, `QueuedProgressDisplayType.POSITION_LENGTH_TIME` 显示的是最详细的排队信息    

转到 `DispatchQueueInfo.getQueued_progress_display()`, 可以注意到实现如下

```java title="DispatchQueueInfo.java"
public final String getQueued_progress_display() {
    RuntimeDirector runtimeDirector = m__m;
    if (runtimeDirector != null && runtimeDirector.isRedirect("6fc95999", 19)) {
        return (String) runtimeDirector.invocationDispatch("6fc95999", 19, this, a.a);
    }
    return this.queued_progress_display;
}
```

那么问题就很简单了. 我们只需要让这个函数返回 `QueuedProgressDisplayType.POSITION_LENGTH_TIME.getValue()` 就好了
直接将 `getQueued_progress_display` 的实现替换如下:

```smali title="DispatchQueueInfo.smali"
.method public final getQueued_progress_display()Ljava/lang/String;
    .registers 1
    .annotation build Lkq/e;
    .end annotation

    sget-object v0, Lcom/mihoyo/cloudgame/commonlib/bean/QueuedProgressDisplayType;->POSITION_LENGTH_TIME:Lcom/mihoyo/cloudgame/commonlib/bean/QueuedProgressDisplayType;

    invoke-virtual {v0}, Lcom/mihoyo/cloudgame/commonlib/bean/QueuedProgressDisplayType;->getValue()Ljava/lang/String;

    move-result-object v0

    return-object v0
.end method
```

这样就完成啦~!