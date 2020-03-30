# Helm Chart模板说明

Helm最核心的就是模板，即模板化的Kubernetes清单文件，模板经过渲染后会被提交到Kubernetes中，本质上就是Go语言的template模板，模板文件位于template/目录中。

将Kubernetes清单文件中可能经常变动的字段，通过指定一个变量，在安装的过程中该变量将被值value动态替换掉，这个过程就是模板的渲染。

变量的值定义在values.yaml文件中，该文件中定义了变量的缺省值，但可以在helm install命令中配置新的值来覆盖缺省值。

- ### 内置对象

对象从模板引擎传递到模板中。如使用{{.Release.Name}}将Release的名称插入到模板中。Release是可以在模板中访问的顶级对象之一。

对象可以很简单，只有一个值。或者他们可以包含其他对象或函数。例如，Release对象包含多个对象（如Release.Name），并且Files对象具有一些函数。

Helm内置了多个顶级对象：

- Release：这个对象描述了Release本身，它里面有几个对象：
    - Release.Name：Release名称。
    - Release.Namespace：Release的Namespace。
    - Release.IsUpgrade：如果当前操作是升级或回滚，则将其设置为true。
    - Release.IsInstall：如果当前操作是安装，则设置为true。
    - Release.Revision：此Release 的修订版本号。
    - Release.Service：渲染此模板的服务，Helm中总是“Helm”。
- Values：从values.yaml文件和用户提供的文件传入模板的值。默认情况下，Values是空的。
- Chart：Chart.yaml文件的内容。
- Files：提供对Chart中所有非特殊文件的访问。虽然您不能使用它来访问模板，但是可以使用它来访问Chart图表中的其他文件。
    - Files.Get是用于按名称获取文件。
    - Files.GetBytes是用于以字节数组而不是字符串的形式获取文件内容的函数。 这对于诸如图像之类的东西很有用。
    - Files.Glob是一个函数，该函数返回名称与给定的Shell Glob模式匹配的文件列表。
    - Files.Lines是一项功能，可逐行读取文件。 这对于遍历文件中的每一行很有用。
    - Files.AsSecrets是一个函数，以Base 64编码的字符串形式返回文件主体。
    - Files.AsConfig是一个将文件正文作为YAML映射返回的函数。
- Capabilities：提供了有关Kubernetes集群支持哪些功能的信息。
    - Capabilities.APIVersions是一组版本。
    - Capabilities.APIVersions.Has：$version指示群集上是否有版本（例如，batch/v1）或资源（例如，apps/v1/Deployment）。
    - Capabilities.KubeVersion是Kubernetes版本。
    - Capabilities.KubeVersion.Version是Kubernetes版本。
    - Capabilities.KubeVersion.Major是Kubernetes的主要版本。
    - Capabilities.KubeVersion.Minor是Kubernetes的次要版本。
- Template：包含有关正在执行的当前模板的信息。
    - Name：当前模板的命名空间文件路径（例如mychart/templates/mytemplate.yaml）
    - BasePath：当前Chart的模板目录的命名空间路径（例如mychart/templates）。

这些值可用于任何顶级模板。内置值始终以大写字母开头，这符合Go的命名约定。

- ### Values文件

四个内置对象之一的Values，该对象提供对传入Chart的值的访问。其内容来自四个来源：

- Chart中的values.yaml文件。
- 如果这是一个子Chart，来自父Chart的values.yaml文件。
- values文件通过helm install或helm upgrade的-f标志传入文件（helm install -f myvals.yaml ./mychart）。
- 通过--set（helm install --set foo=bar ./mychart）。

上面的列表按照特定的顺序排列：values.yaml是默认级别，父级Chart的可以覆盖该默认级别，而该values.yaml又可以被用户提供的values文件覆盖，而该文件又可以被--set参数覆盖。

我们先创建一个Chart。

    helm create mychart
    Creating mychart

我们将默认的values.yaml文件内容删除，并设置一个参数：

    vi mychart/values.yaml
    favoriteDrink: coffee

为了简单起见，删除templates/目录中所有文件，并创建configmap.yaml模板文件。

模板指令放在{{和}}块之间，模板指令{{ .Release.Name }}表示将Release名称注入模板。可以将传递到模板中的值视为命名空间对象，其中句号"."用来分割每个命名空间元素，前导"."表示当前的作用域，此处表示顶级命名空间。{{ .Release.Name }}表示从顶级命名空间开始，找到Release对象，然后在其中查找名为Name的对象。{{ .Values.favoriteDrink }}表示从顶级命名空间开始，找到Values对象，然后在其中查找名为favoriteDrink的对象。

    rm -rf mychart/templates/*

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favoriteDrink }}

我们看看渲染的效果。

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: coffee

参数favoriteDrink在默认values.yaml文件中设置为coffee，但可以在helm template命令中通过加一个--set添标志来覆盖，因为--set的优先级更高。

    helm template myrelease mychart --set favoriteDrink=tea
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: tea

values文件也可以包含更多结构化内容。在values.yaml文件中可以创建favorite添加几个键：

    vi mychart/values.yaml
    favorite:
      drink: coffee
      food: pizza

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink }}
      food: {{ .Values.favorite.food }}
    
    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: coffee
      food: pizza

- ### 函数

在模板文件中有时不想直接引用对象，而是需要对对象做一些转换，比如将对象作为字符串引用，可以使用函数quote。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ quote .Values.favorite.drink }}
      food: {{ quote .Values.favorite.food }}
    
    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "pizza"

模板函数遵循语法functionName arg1 arg2....。在上面的代码段中，quote .Values.favorite.drink调用quote函数并将其传递给单个参数。

- ### 管道

模板语言的强大功能之一是其管道概念。像Linux中的管道一样，是将一系列模板命令链接在一起以紧凑表示一系列转换的工具。管道是按顺序完成多项工作的有效方式。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | quote }}
      food: {{ .Values.favorite.food | quote }}
    
    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "pizza"

管道符号"|"将前面的对象作为后面函数的参数，上面的例子我们将.Values.favorite.drink对象通过管道传递给quote函数。

管道方便的地方在可以顺序将对象在函数中传递。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | repeat 5 | quote }}
      food: {{ .Values.favorite.food | upper | quote }}
    
    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffeecoffeecoffeecoffeecoffee"
      food: "PIZZA"

上例将.Values.favorite.drink对象作为repeat函数的第二个参数，而第一个参数为5。

- ### 缺省值

模板中经常需要用到缺省值功能：default DEFAULT_VALUE GIVEN_VALUE，此功能允许在模板内部指定缺省值，没有定义该对象或者对象为空时有缺省值对应。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | default "tea" | quote }}
      food: {{ .Values.favorite.food | upper | quote }}
    
    helm template myrelease mychart --set favorite.drink=null
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "tea"
      food: "PIZZA"

- ### Helm流程控制-If/Else

Helm通过If/Else块有条件地控制模板中是否包含文本块。

    {{ if PIPELINE }}
      # Do something
    {{ else if OTHER PIPELINE }}
      # Do something else
    {{ else }}
      # Default case
    {{ end }}

注意if后面是管道，当值为以下情况时，管道视为false。

- a boolean false：逻辑值false
- a numeric zero：数字值0
- an empty string：空字符串
- a nil (empty or null)-值为空或者null
- an empty collection (map, slice, tuple, dict, array)：一个空的集合

以下模板根据条件显示mug: true。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | default "tea" | quote }}
      food: {{ .Values.favorite.food | upper | quote }}
      {{ if eq .Values.favorite.drink "coffee" }}mug: true{{ end }}

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
      mug: true

- ### Helm流程控制-缩进

以上的方式控制mug: true的显示不易阅读，尝试使用缩进。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{.Release.Name}}-configmap
    data:
      myvalue: "Hello World"
      drink: {{.Values.favorite.drink | default "tea" | quote}}
      food: {{.Values.favorite.food | upper | quote}}
      {{if eq .Values.favorite.drink "coffee"}}
        mug: true
      {{end}}

    helm template myrelease mychart
    Error: YAML parse error on mychart/templates/configmap.yaml: error converting YAML to JSON: yaml: line 9: did not find expected key

在渲染时报错，实际上模板被渲染为了如下清单文件，多缩进了两个空格。

    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
        mug: true

重新尝试缩进设置。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{.Release.Name}}-configmap
    data:
      myvalue: "Hello World"
      drink: {{.Values.favorite.drink | default "tea" | quote}}
      food: {{.Values.favorite.food | upper | quote}}
      {{if eq .Values.favorite.drink "coffee"}}
      mug: true
      {{end}}

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
    
      mug: true

结果发现缩进正确，但却多出了一个空行。原因是模板渲染时，渲染引擎会直接替换到{{ ... }}，然后原样保留之外的所有字符。所以{{if eq .Values.favorite.drink "coffee"}}后面的换行符被保留了，导致了一个空行。

可以使用特殊字符修改模板声明的大括号语法，以告诉模板引擎填充空白。"{{- "表示删除左边的所有空格，直到非空格字符，而" -}}"表示删除右边的所有空格。注意，换行符也是空格！当然还包括空格，TAB字符。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | default "tea" | quote }}
      food: {{ .Values.favorite.food | upper | quote }}
      {{- if eq .Values.favorite.drink "coffee" }}
      mug: true
      {{- end }}
    
    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
      mug: true

这次缩进完全控制正确了，上面由于空格不显示出来不容易理解，我们可以使用\*替换空格，使用\$替换换行符，这样看的更清楚一些。{{- ... }}表示将左边的\$和\*\*删除。{{ ... -}}表示将右边的\$和\*\*删除。这样就剩下了中间的\$和\*\*，换行符\$恰好就是food和mug两行之间的换行符。

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | default "tea" | quote }}
      food: {{ .Values.favorite.food | upper | quote }}$
    **{{- if eq .Values.favorite.drink "coffee" }}$
    **mug: true$
    **{{- end }}

当然控制缩进更推荐使用函数indent，更容易理解。

    vi mychart/templates/configmap.yaml
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | default "tea" | quote }}
      food: {{ .Values.favorite.food | upper | quote }}
      {{- if eq .Values.favorite.drink "coffee" }}
    {{ "mug: true" | indent 2 }}
      {{- end }}    

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
      mug: true

- ### Helm流程控制-with

with用来控制变量作用域。前面提过，前导"."是对当前作用域的引用。.Values告诉模板在当前范围中查找Values对象。

语法with类似于一个简单的if语句：

    {{ with PIPELINE }}
      # restricted scope
    {{ end }}

作用域是可以更改的，可以将当前作用域"."设置为特定对象。

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      {{- with .Values.favorite }}
      drink: {{ .drink | default "tea" | quote }}
      food: {{ .food | upper | quote }}
      {{- end }}

由于with将当前作用域指定为.Values.favorite，所以在with块中，我们直接使用.drink和.food。

注意：在with受限作用域中，是无法父作用域或者顶级作用域访问其他对象。

    {{- with .Values.favorite }}
    drink: {{ .drink | default "tea" | quote }}
    food: {{ .food | upper | quote }}
    release: {{ .Release.Name }}
    {{- end }}

在with块中不能引用{{ .Release.Name }}，因为当前作用域不是顶级作用域，是无法找到顶级对象Release的。

- ### Helm流程控制-range

在Helm的模板语言中，使用rang来迭代集合。我们在values.yaml中定义一个列表。

    vi mychart/values.yaml
    favorite:
      drink: coffee
      food: pizza
    pizzaToppings:
      - mushrooms
      - cheese
      - peppers
      - onions

我们将列表展示在configmap中。使用range来迭代列表的值，"."表示迭代的当前值，通过管道传递给title函数，将首字母大写，然后再通过管道传递给quote，转为字符串。

    vi mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      {{- with .Values.favorite }}
      drink: {{ .drink | default "tea" | quote }}
      food: {{ .food | upper | quote }}
      {{- end }}
      toppings: |-
        {{- range .Values.pizzaToppings }}
        - {{ . | title | quote }}
        {{- end }}

渲染模板之后可以看出确实迭代List。

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
      toppings: |-
        - "Mushrooms"
        - "Cheese"
        - "Peppers"
        - "Onions"

- ### 变量

在Helm模板中，变量是对另一个对象的命名引用。它遵循$name格式，变量使用特殊的赋值运算符进行赋值":="。

    vi mychart/values.yaml
    favorite:
      drink: coffee
      food: pizza

    vi mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    data:
      myvalue: "Hello World"
      {{- $relname := .Release.Name -}}
      {{- with .Values.favorite }}
      drink: {{ .drink | default "tea" | quote }}
      food: {{ .food | upper | quote }}
      release: {{ $relname }}
      {{- end }}

因为.Release.Name不能在with块中直接使用，所以在with之前将.Release.Name赋值给变量relname。

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
      release: myrelease

- ### 命名模板

Helm支持在一个文件中定义命名模板，然后在其他模板中使用命名模板。命名模板使用define声明，使用template使用模板。

我们可以在模板文件中使用define创建命名模板，语法如下：

    {{ define "MY.NAME" }}
      # body of template here
    {{ end }}

模板名称是全局的，如果声明两个具有相同名称的模板，则只会使用的是最后定义的模板。因为Sub Chart中的模板是与父Chart模板一起编译的，所以应谨慎命名命名模板。一种流行的命名约定是在每个命名模板前添加Chart名称{{define "mychart.labels"}}，可以避免由于两个不同的Chart定义了相同名称的模板而引起的任何冲突。

可以在configmap.yaml中直接定义命名模板，然后使用template使用该命名模板。

    vi mychart/templates/configmap.yaml
    {{- define "mychart.labels" }}
      labels:
        generator: helm
        date: {{ now | htmlDate }}
    {{- end }}
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
      {{- template "mychart.labels" }}
    data:
      myvalue: "Hello World"

渲染效果如下。

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
      labels:
        generator: helm
        date: 2020-03-29
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"

我们先看一下templates/目录下的文件命名约定：
- NOTES.txt是一个例外，改文件是安装后的说明文件，不是Kubernetes清单文件。
- 名称以下划线"_"开头的文件不是Kubernetes清单文件。_helpers.tpl就是缺省的命名模板定义文件。
- 除此之外都是Kubernetes清单文件，在安装时会自动渲染并加载到Kubernetes中。

我们将以上的命名模板移到_helpers.tpl文件中。

    vi mychart/templates/_helpers.tpl
    {{/* Generate basic labels */}}
    {{- define "mychart.labels" }}
      labels:
        generator: helm
        date: {{ now | htmlDate }}
    {{- end }}

删除configmap.yaml中的命名模板。

    vi mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
      {{- template "mychart.labels" }}
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | default "tea" | quote }}
      food: {{ .Values.favorite.food | upper | quote }}

有相同的渲染效果。

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
      labels:
        generator: helm
        date: 2020-03-29
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"

我们也可以使用include使用命名模板。

    vi mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ .Release.Name }}-configmap
    {{- include "mychart.labels" . }}
    data:
      myvalue: "Hello World"
      drink: {{ .Values.favorite.drink | default "tea" | quote }}
      food: {{ .Values.favorite.food | upper | quote }}

也有相同的渲染效果。

    helm template myrelease mychart
    ---
    # Source: mychart/templates/configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: myrelease-configmap
      labels:
        generator: helm
        date: 2020-03-29
    data:
      myvalue: "Hello World"
      drink: "coffee"
      food: "PIZZA"
