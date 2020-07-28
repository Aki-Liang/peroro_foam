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

