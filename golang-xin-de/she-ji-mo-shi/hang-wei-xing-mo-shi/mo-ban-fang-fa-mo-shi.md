# 模板方法模式

**定义一个操作中的算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以不改变一个算法的结构即可重定义该算法的某些步骤。**

## 实现

以下载文件为例，分为两个步骤：下载和保存。算法步骤就是先下载，然后保存。不同方式的下载器遵循相同的步骤，但是每个步骤的具体操作不同，例如HTTP下载器和ftp下载器在下载时采用的连接方式等不同。

首先，定义下载器接口：

```text
type Downloader interface {
    Download(uri string)
}
```

定义模板方法及需要实现的接口：

```text
// 将下载和保存两个步骤抽象出来
type implement interface {
    download()
    save()
}

type template struct {
    implement
    uri string
}

func newTemplate(impl implement) *template {
    return &template{
        implement: impl,
    }
}

func (t *template) Download(uri string) {
    t.uri = uri
    fmt.Print("prepare downloading\n")
    t.implement.download()
    t.implement.save()
    fmt.Print("finish downloading\n")
}

func (t *template) save() {
    fmt.Print("default save\n")
}
```

实现具体的下载器，无需修改`template.Download`方法，只需要实现`implement`接口（也可以不实现，则使用默认的）：

```text
type HTTPDownloader struct {
    *template
}

func NewHTTPDownloader() Downloader {
    downloader := &HTTPDownloader{}
    template := newTemplate(downloader)
    downloader.template = template
    return downloader
}

func (d *HTTPDownloader) download() {
    fmt.Printf("download %s via http\n", d.uri)
}

func (*HTTPDownloader) save() {
    fmt.Printf("http save\n")
}

type FTPDownloader struct {
    *template
}

func NewFTPDownloader() Downloader {
    downloader := &FTPDownloader{}
    template := newTemplate(downloader)
    downloader.template = template
    return downloader
}

func (d *FTPDownloader) download() {
    fmt.Printf("download %s via ftp\n", d.uri)
}
```

客户端调用：

```text
var downloader Downloader = NewHTTPDownloader()
downloader.Download("http://example.com/abc.zip")
```

模板方法模式通过把不变行为搬移到超类，去除子类中的重复代码。

当不变和可变的行为在方法的子类实现中混合在一起时，不变的行为就会在子类中重复出现，通过模板方法模式把这些行为搬移到单一的地方，这样就帮助子类摆脱重复的不变行为的纠缠。

