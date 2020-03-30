# Helm Hook钩子

- ### Hooks

Helm提供了Hook的机制，允许Chart开发人员在Release的生命周期中的某些节点来进行干预，比如我们可以利用Hooks来做下面的这些事情：

- 在加载Chart的其它资源之前，先加载ConfigMap或Secret。
- 在安装新Chart之前执行作业以备份数据库，然后在升级后执行第二个作业以恢复数据。
- 在删除Release之前运行作业，以便在删除Release之前优雅地停止服务。

Hooks和普通模板一样工作，但是它们具有特殊的注释，可以使Helm以不同的方式使用它们。

Hooks在资源清单中的metadata部分用annotations注解的方式进行声明：

    apiVersion: batch/v1
    kind: Job
    metadata:
      annotations:
        "helm.sh/hook": post-install
        "helm.sh/hook-weight": "-5"
        "helm.sh/hook-delete-policy": hook-succeeded
    ......

- ### Hooks类型

Helm中定义了如下一些可供我们使用的Hooks：

| 注解值 | 说明 |
| :------| :------ |
| 预安装pre-install | 在渲染模板之后，但在Kubernetes中创建任何资源之前执行 |
| 安装后post-install | 在所有Kubernetes资源安装到集群后执行 |
| 预删除pre-delete | 在从Kubernetes删除任何资源之前执行删除请求 |
| 删除后post-delete | 删除所有Release的资源后执行 |
| 升级前pre-upgrade | 在渲染模板之后，但在任何资源升级之前执行 |
| 升级后post-upgrade | 在所有资源升级后执行 |
| 预回滚pre-rollback | 在模板渲染后，在任何资源回滚之前执行 |
| 回滚后post-rollback | 在修改所有资源后执行回滚请求 |
| 测试test | 在调用Helm test子命令时执行 |

请注意，为了支持Helm 3中的crds目录，已删除了crd-install钩子。

- ### Hooks和Release生命周期

Hooks允许开发人员在Release的生命周期中的一些关键节点执行一些钩子，我们正常安装一个Chart包的时候的生命周期如下所示：

1. 用户运行helm install foo
1. Helm客户端调用Helm library库的安装API
1. 经过一些验证，Helm库渲染foo模板
1. Helm库将产生的资源加载到Kubernetes中去
1. Helm库将Release对象（和其它数据）返回给Helm客户端
1. Helm客户端退出

如果开发人员在install的生命周期中定义了两个hook：pre-install和post-install，那么我们安装一个Chart包的生命周期就会多一些步骤了：

1. 用户运行helm install foo
1. Helm客户端调用Helm library库的安装API
1. 安装crds目录下的CRDs
1. 经过一些验证，Helm库渲染foo模板
1. Helm库准备执行pre-install hooks（将hook资源加载到kubernetes中）
1. Helm库根据权重对hooks进行排序（默认分配权重0），权重相同时按照名称升序排序
1. Helm库然后加载最低权重的hook（从负到正）
1. Helm库等待，直到hook就绪Ready（除了CRDs）
1. Helm库将产生的资源加载到Kubernetes中去，注意：如果设置了--wait参数，Helm库将会等待直到所有资源为就绪状态，在所有资源就绪前post-install hook不会执行
1. Helm库执行post-install hook（将hook资源加载到kubernetes中）
1. Helm库等待，直到hook就绪Ready
1. Helm库将Release对象（和其它数据）返回给Helm客户端
1. Helm客户端退出

等待直到hook就绪Ready意味着什么？这是一个阻塞的操作，如果 hook 中声明的是一个 Job 资源，那么 Tiller 将等待 Job 成功完成，如果失败，则发布失败，在这个期间，Helm 客户端是处于暂停状态的。

对于所有其他类型，只要 kubernetes 将资源标记为加载（添加或更新），资源就被视为就绪状态，当一个 hook 声明了很多资源是，这些资源是被串行执行的。

另外需要注意的是 hook 创建的资源不会作为 release 的一部分进行跟踪和管理，一旦 Tiller Server 验证了 hook 已经达到了就绪状态，它就不会去管它了。

所以，如果我们在 hook 中创建了资源，那么不能依赖helm delete去删除资源，因为 hook 创建的资源已经不受控制了，要销毁这些资源，需要在pre-delete或者post-delete这两个 hook 函数中去执行相关操作，或者将helm.sh/hook-delete-policy这个 annotation 添加到 hook 模板文件中。

等待直到hook就绪Ready意味着什么？这取决于钩子中声明的资源。如果资源是Job或Pod类型，Helm将等待直到成功运行完成completion为止。如果钩子失败，Release就会失败。

对于所有其它资源类型，只要Kubernetes将资源标记为已加载loaded（添加added或更新updated），该资源即被视为就绪Ready。

- ### 写一个Hook

Hooks也是Kubernetes清单文件，只不过是在metadata中带有特殊的注解annotation。因为Hooks也是模板文件，所以可以使用所有常规模板功能，包括读取.Values，.Release和.Template。

例如，现在我们来创建一个Hook，在前面的示例templates/目录中添加一个post-install-job.yaml的文件，表示安装后执行的一个hook：

    apiVersion: batch/v1
    kind: Job
    metadata:
      name: "{{ .Release.Name }}"
      labels:
        app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
        app.kubernetes.io/instance: {{ .Release.Name | quote }}
        app.kubernetes.io/version: {{ .Chart.AppVersion }}
        helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
      annotations:
        # This is what defines this resource as a hook. Without this line, the
        # job is considered part of the release.
        "helm.sh/hook": post-install
        "helm.sh/hook-weight": "-5"
        "helm.sh/hook-delete-policy": hook-succeeded
    spec:
      template:
        metadata:
          name: "{{ .Release.Name }}"
          labels:
            app.kubernetes.io/managed-by: {{ .Release.Service | quote }}
            app.kubernetes.io/instance: {{ .Release.Name | quote }}
            helm.sh/chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
        spec:
          restartPolicy: Never
          containers:
          - name: post-install-job
            image: "alpine:3.3"
            command: ["/bin/sleep","{{ default "10" .Values.sleepyTime }}"]

使这个模板成为钩子的原因是以下的注解：

      annotations:
        "helm.sh/hook": post-install

一种资源也可以实现多个挂钩，如下该资源可以在安装后和升级后执行。

    annotations:
      "helm.sh/hook": post-install,post-upgrade

同样，一种Hook也可以有多个资源支持。例如，可以将Secret和ConfigMap都声明为预安装钩子。

当subchart声明钩子时，也会对其进行评估。顶级Chart图表无法禁用子Chart所声明的钩子。

可以为钩子定义权重，这将有助于建立确定性的执行顺序。权重使用以下注释定义：

    annotations:
      "helm.sh/hook-weight": "5"

- ### 钩子删除策略

钩子删除策略定义了何时删除相应钩子资源的策略。挂钩删除策略使用以下注释定义：

    annotations:
      "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded


| 注解值 | 说明 |
| :------| :------ |
| before-hook-creation | 在启动新的挂钩之前删除先前的资源（默认） |
| hook-succeeded | 挂钩成功执行后删除资源 |
| hook-failed | 如果挂钩在执行期间失败，则删除资源 |
