![下载管理器](download_01.png)
/// caption
下载管理器
///

## DownlaodManager

`DownlaodManager` 逻辑主要有两个部分组成：`TaskPool` 和 `DownloadCounter`，前者负责管理多个下载代理并处理并发下载，而后者则是跟踪所有下载操作的进度和状态。从它的主循环中可以看到：

``` cs title="DownlaodManager.Update"
internal override void Update(float elapseSeconds, float realElapseSeconds)
{
    m_TaskPool.Update(elapseSeconds, realElapseSeconds);
    m_DownloadCounter.Update(elapseSeconds, realElapseSeconds);
}
```

另外提供了四个下载事件供外部注册：

- DownloadStart
- DownloadUpdate
- DownloadSuccess
- DownloadFailure

它们由任一 `DownloadAgent` 触发。

## DownloadAgent

一个下载代理包含三个主要对象：

- **DownloadTask**: 提供下载信息，比如下载地址、超时时间等
- **IDownloadAgentHelper**: 虽然是下载辅助器，但实际的下载行为被它执行，上层需要派生实现具体的网络操作以下载数据
- **FileStream**: 文件流，通过它写入下载数据并按一定的缓冲区大小保存到磁盘

### 1. 开始下载任务

``` cs title="开始下载任务"
public StartTaskStatus Start(DownloadTask task)
{
    if (task == null)
    {
        throw new GameFrameworkException("Task is invalid.");
    }

    m_Task = task;

    m_Task.Status = DownloadTaskStatus.Doing;
    string downloadFile = Utility.Text.Format("{0}.download", m_Task.DownloadPath);

    try
    {
        // 检查路径并初始化下载大小
        if (File.Exists(downloadFile))
        {
            m_FileStream = File.OpenWrite(downloadFile);
            m_FileStream.Seek(0L, SeekOrigin.End);
            m_StartLength = m_SavedLength = m_FileStream.Length;
            m_DownloadedLength = 0L;
        }
        else
        {
            string directory = Path.GetDirectoryName(m_Task.DownloadPath);
            if (!Directory.Exists(directory))
            {
                Directory.CreateDirectory(directory);
            }

            m_FileStream = new FileStream(downloadFile, FileMode.Create, FileAccess.Write);
            m_StartLength = m_SavedLength = m_DownloadedLength = 0L;
        }

        // 通知外部开始
        if (DownloadAgentStart != null)
        {
            DownloadAgentStart(this);
        }

        // 开始执行下载
        if (m_StartLength > 0L)
        {
            m_Helper.Download(m_Task.DownloadUri, m_StartLength, m_Task.UserData);
        }
        else
        {
            m_Helper.Download(m_Task.DownloadUri, m_Task.UserData);
        }

        // 固定返回CanResume，每帧推动直到下载完成
        return StartTaskStatus.CanResume;
    }
    catch (Exception exception)
    {
        // 下载失败的处理
        DownloadAgentHelperErrorEventArgs downloadAgentHelperErrorEventArgs = DownloadAgentHelperErrorEventArgs.Create(false, exception.ToString());
        OnDownloadAgentHelperError(this, downloadAgentHelperErrorEventArgs);
        ReferencePool.Release(downloadAgentHelperErrorEventArgs);
        return StartTaskStatus.UnknownError;
    }
}
```

### 2. 辅助器承担下载回调

``` cs title="辅助器的职责"
/// <summary>
/// 初始化下载代理。
/// </summary>
public void Initialize()
{
    m_Helper.DownloadAgentHelperUpdateBytes += OnDownloadAgentHelperUpdateBytes;
    m_Helper.DownloadAgentHelperUpdateLength += OnDownloadAgentHelperUpdateLength;
    m_Helper.DownloadAgentHelperComplete += OnDownloadAgentHelperComplete;
    m_Helper.DownloadAgentHelperError += OnDownloadAgentHelperError;
}
```

业务需要实现一个具体的下载辅助器，通过回调触发推动底层的运作

### 3. 下载回调

``` cs title="更新字节数据"
private void OnDownloadAgentHelperUpdateBytes(object sender, DownloadAgentHelperUpdateBytesEventArgs e)
{
    m_WaitTime = 0f;
    try
    {
        m_FileStream.Write(e.GetBytes(), e.Offset, e.Length);
        m_WaitFlushSize += e.Length;
        m_SavedLength += e.Length;

        if (m_WaitFlushSize >= m_Task.FlushSize)
        {
            m_FileStream.Flush();
            m_WaitFlushSize = 0;
        }
    }
    catch (Exception exception)
    {
        DownloadAgentHelperErrorEventArgs downloadAgentHelperErrorEventArgs = DownloadAgentHelperErrorEventArgs.Create(false, exception.ToString());
        OnDownloadAgentHelperError(this, downloadAgentHelperErrorEventArgs);
        ReferencePool.Release(downloadAgentHelperErrorEventArgs);
    }
}
```

下载的字节数据通过字节流写入文件中，当达到缓冲区上限后，保存至磁盘中。

``` cs title="更新下载大小"
private void OnDownloadAgentHelperUpdateLength(object sender, DownloadAgentHelperUpdateLengthEventArgs e)
{
    m_WaitTime = 0f;
    m_DownloadedLength += e.DeltaLength;
    if (DownloadAgentUpdate != null)
    {
        DownloadAgentUpdate(this, e.DeltaLength);
    }
}
```

下载大小从这里回调到了 `DownloadManager`，是 `DownloadCounter` 的数据来源

``` cs title="下载完成"
private void OnDownloadAgentHelperComplete(object sender, DownloadAgentHelperCompleteEventArgs e)
{
    m_WaitTime = 0f;
    m_DownloadedLength = e.Length;
    if (m_SavedLength != CurrentLength)
    {
        throw new GameFrameworkException("Internal download error.");
    }

    m_Helper.Reset();
    m_FileStream.Close();
    m_FileStream = null;

    if (File.Exists(m_Task.DownloadPath))
    {
        File.Delete(m_Task.DownloadPath);
    }

    File.Move(Utility.Text.Format("{0}.download", m_Task.DownloadPath), m_Task.DownloadPath);

    m_Task.Status = DownloadTaskStatus.Done;

    if (DownloadAgentSuccess != null)
    {
        DownloadAgentSuccess(this, e.Length);
    }

    m_Task.Done = true;
}
```

下载完成，关闭字节流，转换最终文件并通知下载完成

``` cs title="下载错误"
private void OnDownloadAgentHelperError(object sender, DownloadAgentHelperErrorEventArgs e)
{
    m_Helper.Reset();
    if (m_FileStream != null)
    {
        m_FileStream.Close();
        m_FileStream = null;
    }

    if (e.DeleteDownloading)
    {
        File.Delete(Utility.Text.Format("{0}.download", m_Task.DownloadPath));
    }

    m_Task.Status = DownloadTaskStatus.Error;

    if (DownloadAgentFailure != null)
    {
        DownloadAgentFailure(this, e.ErrorMessage);
    }

    m_Task.Done = true;
}
```

## IDownloadAgentHelper

辅助器接口的定义很简单，上层需要实现下载函数，并提供数据更新事件、大小更新事件、完成事件和错误事件

``` cs title="IDownloadAgentHelper"
public interface IDownloadAgentHelper
{
    event EventHandler<DownloadAgentHelperUpdateBytesEventArgs>
    DownloadAgentHelperUpdateBytes; // 下载代理辅助器更新数据流事件
    event EventHandler<DownloadAgentHelperUpdateLengthEventArgs>
    DownloadAgentHelperUpdateLength; // 下载代理辅助器更新数据大小事件
    event EventHandler<DownloadAgentHelperCompleteEventArgs>
    DownloadAgentHelperComplete; // 下载代理辅助器完成事件
    event EventHandler<DownloadAgentHelperErrorEventArgs>
    DownloadAgentHelperError; // 下载代理辅助器错误事件
    void Download(string downloadUri, object userData); // 通过下载代理辅助器下载指定地址的数据
    void Download(string downloadUri, long fromPosition, object userData); // 通过下载代理辅助器下载指定地址的数据
    void Download(string downloadUri, long fromPosition, long toPosition, object userData); // 通过下载代理辅助器下载指定地址的数据
    void Reset(); // 重置下载代理辅助器
}
```

UGF 实现了基于 WWW 和 UnityWebRequest 两种方案的辅助器，感兴趣可以自己看看，也可以针对自己的项目实现自己的下载逻辑，这里仅分析 GF 模块架构不赘述

## 总结

下载管理器是游戏框架的核心模块，它提供强大的功能，可将文件从远程源下载到本地存储。它管理下载任务、跟踪下载进度、处理错误，并提供可配置的并发下载系统，支持优先级设置。