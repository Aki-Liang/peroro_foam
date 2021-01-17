# job

描述完成后退出的离线业务的API对象

```
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4   //重试次数为 4
```

Job Controller直接控制 Pod对象

Job Controller 在控制循环中进行的调谐（Reconcile）操作，是根据实际在 Running 状态 Pod 的数目、已经成功退出的 Pod 的数目，以及 parallelism、completions 参数的值共同计算出在这个周期里，应该创建或者删除的 Pod 数目，然后调用 Kubernetes API 来执行

### 重试
当离线作业失败时，如果定义了 restartPolicy=Never。，Controller 就会不断地尝试创建一个新 Pod，spec.backoffLimit 字段里定义了重试次数为 4（默认是6）
Job Controller 重新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10 s、20 s、40 s …后

如果定义的 restartPolicy=OnFailure，离线作业失败后，Job Controller 就不会去尝试创建新的 Pod。而会不断地尝试重启 Pod 里的容器

### 超时

在 Job 的 API 对象里，有一个 spec.activeDeadlineSeconds 字段可以设置最长运行时间
可以在 Pod 的状态里看到终止的原因是 reason: DeadlineExceeded

### 并行作业

在 Job 对象中，负责并行控制的参数有两个：

1. spec.parallelism，它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行
2. spec.completions，它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数

### 常见应用场景

1. 外部管理器 +Job 模板。
2. 拥有固定任务数目的并行 Job。
3. 指定并行度（parallelism），但不设置固定的 completions 的值。

## CronJob

```
例
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。可以通过 spec.concurrencyPolicy 字段来定义具体的处理策略

1. concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在；
2. concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过；
3. concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job。
   
如果某一次 Job 创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss 的数目达到 100 时， CronJob 会停止再创建这个 Job
这个时间窗口，可以由 spec.startingDeadlineSeconds 字段指定。比如 startingDeadlineSeconds=200，意味着在过去 200 s 里，如果 miss 的数目达到了 100 次，这个 Job 就不会被创建执行了