# 编码规范

[Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)


## Interface 合理性验证

```
type Handler struct {
  // ...
}
// 用于触发编译期的接口的合理性检查机制
// 如果Handler没有实现http.Handler,会在编译期报错
var _ http.Handler = (*Handler)(nil)
func (h *Handler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  // ...
}
```

## 零值 Mutex 是有效的

零值 sync.Mutex 和 sync.RWMutex 是有效的。所以指向 mutex 的指针基本是不必要的。

```
Bad Ver
    mu := new(sync.Mutex)
    mu.Lock()

Good Ver
    var mu sync.Mutex
    mu.Lock()
```