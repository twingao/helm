# Helm Chart开发入门

本文我们展示如何开发一个简单的Chart，其中包含了一个Deployment和Service简单的Template，最后安装该Chart。整个Chart的代码已经放到[https://github.com/twingao/httpbin](https://github.com/twingao/httpbin)。

先使用helm create命令创建一个Chart。

    helm create httpbin
    Creating httpbin

查看httpbin的目录结构。

    tree httpbin -a
    httpbin
    ├── charts
    ├── Chart.yaml
    ├── .helmignore
    ├── templates
    │   ├── deployment.yaml
    │   ├── _helpers.tpl
    │   ├── ingress.yaml
    │   ├── NOTES.txt
    │   ├── serviceaccount.yaml
    │   ├── service.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml
    
    3 directories, 10 files

Helm规范了Chart的目录和文件结构，这些目录或者文件都有确定的用途。

- charts/，包含其它Chart，称之为Sub Chart，或者依赖Chart。

- Chart.yaml，包含Chart的说明，可在从模板中访问Chart定义的值。

- .helmignore，定义了在helm package时哪些文件不会打包到Chart包tgz中。

- ci/，缺省没有该目录，持续集成的一些脚本。

- templates/，用于放置模板文件，主要定义提交给Kubernetes的资源yaml文件。安装Chart时，Helm会根据chart.yaml、values.yam以及命令行提供的值对Templates进行渲染，最后会将渲染的资源提交给Kubernetes。

- _helpers.tpl，定义了一些可重用的模板片断，此文件中的定义在任何资源定义模板中可用。

- NOTES.txt，提供了安装后的使用说明，在Chart安装和升级等操作后，

- tests/，包含了测试用例。测试用例是pod资源，指定一个的命令来运行容器。容器应该成功退出（exit 0），测试被认为是成功的。该pod定义必须包含helm测试hook注释之一：helm.sh/hook: test-success或helm.sh/hook: test-failure。

- values.yaml，values文件对模板很重要，该文件包含Chart默认值。Helm渲染template时使用这些值。

为了部署httpbin服务，我们对Chart的默认配置做一些相应的修改。

查看values.yaml文件，需要做一些配置修改：

- 将镜像改为image.repository=docker.io/kennethreitz/httpbin。
- 不创建serviceAccount，serviceAccount.create=false
- 为了Kubernetes集群外能访问Service，将type改为NodePort。并增加一个参数，为nodePort配置一个固定端口。

values.yaml修改如下：

    cd httpbin/

    vi values.yaml
    # Default values for httpbin.
    # This is a YAML-formatted file.
    # Declare variables to be passed into your templates.
    
    replicaCount: 1
    
    image:
      repository: docker.io/kennethreitz/httpbin
      tag: latest
      pullPolicy: IfNotPresent
    
    imagePullSecrets: []
    nameOverride: ""
    fullnameOverride: ""
    
    serviceAccount:
      # Specifies whether a service account should be created
      create: false
      # The name of the service account to use.
      # If not set and create is true, a name is generated using the fullname template
      name:
    
    podSecurityContext: {}
      # fsGroup: 2000
    
    securityContext: {}
      # capabilities:
      #   drop:
      #   - ALL
      # readOnlyRootFilesystem: true
      # runAsNonRoot: true
      # runAsUser: 1000
    
    service:
      type: NodePort
      port: 80
      nodePort: 30080
    
    ingress:
      enabled: false
      annotations: {}
        # kubernetes.io/ingress.class: nginx
        # kubernetes.io/tls-acme: "true"
      hosts:
        - host: chart-example.local
          paths: []
      tls: []
      #  - secretName: chart-example-tls
      #    hosts:
      #      - chart-example.local
    
    resources: {}
      # We usually recommend not to specify default resources and to leave this as a conscious
      # choice for the user. This also increases chances charts run on environments with little
      # resources, such as Minikube. If you do want to specify resources, uncomment the following
      # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
      # limits:
      #   cpu: 100m
      #   memory: 128Mi
      # requests:
      #   cpu: 100m
      #   memory: 128Mi
    
    nodeSelector: {}
    
    tolerations: []
    
    affinity: {}

由于配置了固定的nodePort，所以在service.yaml中增加该参数，并引用了对应的value值。

    vi templates/service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: {{ include "httpbin.fullname" . }}
      labels:
        {{- include "httpbin.labels" . | nindent 4 }}
    spec:
      type: {{ .Values.service.type }}
      ports:
        - port: {{ .Values.service.port }}
          targetPort: http
          protocol: TCP
          name: http
          nodePort: {{ .Values.service.nodePort }}
      selector:
        {{- include "httpbin.selectorLabels" . | nindent 4 }}

查看deployment.yaml，可以看出镜像的tag使用变量{{ .Chart.AppVersion }}，该参数定义在Chart.yaml文件中。

    vi templates/deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: {{ include "httpbin.fullname" . }}
      labels:
        {{- include "httpbin.labels" . | nindent 4 }}
    spec:
      replicas: {{ .Values.replicaCount }}
      selector:
        matchLabels:
          {{- include "httpbin.selectorLabels" . | nindent 6 }}
      template:
        metadata:
          labels:
            {{- include "httpbin.selectorLabels" . | nindent 8 }}
        spec:
        {{- with .Values.imagePullSecrets }}
          imagePullSecrets:
            {{- toYaml . | nindent 8 }}
        {{- end }}
          serviceAccountName: {{ include "httpbin.serviceAccountName" . }}
          securityContext:
            {{- toYaml .Values.podSecurityContext | nindent 8 }}
          containers:
            - name: {{ .Chart.Name }}
              securityContext:
                {{- toYaml .Values.securityContext | nindent 12 }}
              image: "{{ .Values.image.repository }}:{{ .Chart.AppVersion }}"
              imagePullPolicy: {{ .Values.image.pullPolicy }}
              ports:
                - name: http
                  containerPort: 80
                  protocol: TCP
              livenessProbe:
                httpGet:
                  path: /
                  port: http
              readinessProbe:
                httpGet:
                  path: /
                  port: http
              resources:
                {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.nodeSelector }}
          nodeSelector:
            {{- toYaml . | nindent 8 }}
          {{- end }}
        {{- with .Values.affinity }}
          affinity:
            {{- toYaml . | nindent 8 }}
        {{- end }}
        {{- with .Values.tolerations }}
          tolerations:
            {{- toYaml . | nindent 8 }}
        {{- end }}

将Chart.yaml文件中的appVersion改为latest。appVersion一般配置为镜像的版本号。

    vi Chart.yaml
    apiVersion: v2
    name: httpbin
    description: A Helm chart for Kubernetes
    
    # A chart can be either an 'application' or a 'library' chart.
    #
    # Application charts are a collection of templates that can be packaged into versioned archives
    # to be deployed.
    #
    # Library charts provide useful utilities or functions for the chart developer. They're included as
    # a dependency of application charts to inject those utilities and functions into the rendering
    # pipeline. Library charts do not define any templates and therefore cannot be deployed.
    type: application
    
    # This is the chart version. This version number should be incremented each time you make changes
    # to the chart and its templates, including the app version.
    version: 0.1.0
    
    # This is the version number of the application being deployed. This version number should be
    # incremented each time you make changes to the application.
    appVersion: latest

现在已经开发完成了httpbin的Chart。我们看一下模板的渲染后的效果。

    helm template my-release httpbin
    ---
    # Source: httpbin/templates/service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-release-httpbin
      labels:
        helm.sh/chart: httpbin-0.1.0
        app.kubernetes.io/name: httpbin
        app.kubernetes.io/instance: my-release
        app.kubernetes.io/version: "latest"
        app.kubernetes.io/managed-by: Helm
    spec:
      type: NodePort
      ports:
        - port: 80
          targetPort: http
          protocol: TCP
          name: http
          nodePort: 30080
      selector:
        app.kubernetes.io/name: httpbin
        app.kubernetes.io/instance: my-release
    ---
    # Source: httpbin/templates/deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-release-httpbin
      labels:
        helm.sh/chart: httpbin-0.1.0
        app.kubernetes.io/name: httpbin
        app.kubernetes.io/instance: my-release
        app.kubernetes.io/version: "latest"
        app.kubernetes.io/managed-by: Helm
    spec:
      replicas: 1
      selector:
        matchLabels:
          app.kubernetes.io/name: httpbin
          app.kubernetes.io/instance: my-release
      template:
        metadata:
          labels:
            app.kubernetes.io/name: httpbin
            app.kubernetes.io/instance: my-release
        spec:
          serviceAccountName: default
          securityContext:
            {}
          containers:
            - name: httpbin
              securityContext:
                {}
              image: "docker.io/kennethreitz/httpbin:latest"
              imagePullPolicy: IfNotPresent
              ports:
                - name: http
                  containerPort: 80
                  protocol: TCP
              livenessProbe:
                httpGet:
                  path: /
                  port: http
              readinessProbe:
                httpGet:
                  path: /
                  port: http
              resources:
                {}
    ---
    # Source: httpbin/templates/tests/test-connection.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: "my-release-httpbin-test-connection"
      labels:
    
        helm.sh/chart: httpbin-0.1.0
        app.kubernetes.io/name: httpbin
        app.kubernetes.io/instance: my-release
        app.kubernetes.io/version: "latest"
        app.kubernetes.io/managed-by: Helm
      annotations:
        "helm.sh/hook": test-success
    spec:
      containers:
        - name: wget
          image: busybox
          command: ['wget']
          args:  ['my-release-httpbin:80']
      restartPolicy: Never

然后我们安装Chart。

    cd ..

    helm install my-release httpbin
    NAME: my-release
    LAST DEPLOYED: Sun Mar 29 12:43:29 2020
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    NOTES:
    1. Get the application URL by running these commands:
      export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-release-httpbin)
      export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
      echo http://$NODE_IP:$NODE_PORT

查看Kubernetes的资源。

    kubectl get all
    NAME                                      READY   STATUS    RESTARTS   AGE
    pod/my-release-httpbin-74956dc94f-mgwcp   1/1     Running   0          104s
    
    NAME                         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
    service/kubernetes           ClusterIP   10.1.0.1      <none>        443/TCP        105d
    service/my-release-httpbin   NodePort    10.1.170.94   <none>        80:30080/TCP   104s
    
    NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/my-release-httpbin   1/1     1            1           104s
    
    NAME                                            DESIRED   CURRENT   READY   AGE
    replicaset.apps/my-release-httpbin-74956dc94f   1         1         1       104s

根据安装说明Notes，可以执行这些命令给出如何访问该服务。

    export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-release-httpbin)
    export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT
    http://192.168.1.55:30080

访问httpbin服务。说明该Chart开发完成，并能够正确安装。

    curl -i http://192.168.1.55:30080/status/200
    HTTP/1.1 200 OK
    Server: gunicorn/19.9.0
    Date: Sun, 29 Mar 2020 04:47:41 GMT
    Connection: keep-alive
    Content-Type: text/html; charset=utf-8
    Access-Control-Allow-Origin: *
    Access-Control-Allow-Credentials: true
    Content-Length: 0

根据上面渲染的Kubernetes资源，httpbin/templates/tests/test-connection.yaml文件定义了测试用例，在Pod中通过wget my-release-httpbin:80来测试该Chart是否安装成功。

    helm test my-release
    Pod my-release-httpbin-test-connection pending
    Pod my-release-httpbin-test-connection pending
    Pod my-release-httpbin-test-connection succeeded
    NAME: my-release
    LAST DEPLOYED: Sun Mar 29 12:43:29 2020
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1
    TEST SUITE:     my-release-httpbin-test-connection
    Last Started:   Sun Mar 29 12:49:13 2020
    Last Completed: Sun Mar 29 12:49:31 2020
    Phase:          Succeeded
    NOTES:
    1. Get the application URL by running these commands:
      export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-release-httpbin)
      export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
      echo http://$NODE_IP:$NODE_PORT

.helmignore定义了在helm package时哪些文件不会打包到Chart包tgz中。可以预览一下Helm预定义忽略了哪些文件。也可以自行增加需要忽略的文件。

    vi httpbin/.helmignore
    # Patterns to ignore when building packages.
    # This supports shell glob matching, relative path matching, and
    # negation (prefixed with !). Only one pattern per line.
    .DS_Store
    # Common VCS dirs
    .git/
    .gitignore
    .bzr/
    .bzrignore
    .hg/
    .hgignore
    .svn/
    # Common backup files
    *.swp
    *.bak
    *.tmp
    *~
    # Various IDEs
    .project
    .idea/
    *.tmproj
    .vscode/

将Chart打包，tgz包文件中会自动添加Chart.yaml的version参数。

    helm package httpbin
    Successfully packaged chart and saved it to: /root/httpbin-0.1.0.tgz
    
    ls
    httpbin  httpbin-0.1.0.tgz
