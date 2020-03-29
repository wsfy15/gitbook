# 单例模式

**保证一个类仅有一个实例，并提供一个访问它的全局访问点。**

## 实现

在golang中，单例模式很容易实现。

```text
type Singleton struct {}

var singleton *Singleton
var once sync.Once // 只执行一次

func GetInstance() *Singleton {
    once.Do(func() {
        singleton = &Singleton{}
    })
    return singleton
}
```

