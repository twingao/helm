# Helm 一些常见的疑难用法

### 用法1

    helm install myrelease mychart \
      --set favorites[0].drink=coffee \
      --set favorites[0].food=pizza \
      --set favorites[1].drink=tea \
      --set favorites[1].food=hotpot

等效于

    favorites:
      - drink: coffee
        food: pizza
      - drink: tea
        food: hotpot

### 用法2

    prometheus:
      ingress:
        hosts: []
        #  - prometheus.example.com

    helm install myrelease mychart \
      --set prometheus.ingress.hosts={prometheus.example.com}

等效于

    prometheus:
      ingress:
        hosts: []
          - prometheus.example.com

### 用法3

    prometheus:
      ingress:
        tls: []
          # - secretName: prometheus-general-tls
          #   hosts:
          #     - prometheus.example.com
    
    helm install myrelease mychart \
      --set prometheus.ingress.tls[0].secretName=prometheus-general-tls \
      --set prometheus.ingress.tls[0].hosts[0]=prometheus.example.com

等效于

    prometheus:
      ingress:
        tls: []
          - secretName: prometheus-general-tls
            hosts:
              - prometheus.example.com

### 用法4

    prometheus:
      service: {}
      #  type: ClusterIP   
    	#  targetPort: 9090
      #  nodePort: 30090
    
    helm install myrelease mychart \
      --set prometheus.service.type=ClusterIP \
      --set prometheus.service.targetPort=9090 \
      --set prometheus.service.nodePort=30090

等效于

    prometheus:
      service:
        type: ClusterIP   
    	  targetPort: 9090
        nodePort: 30090

### 用法5

    prometheus:
      ingress:
        annotations: {}
        #  kubernetes.io/ingress.class: nginx
        #  kubernetes.io/tls-acme: 

    helm install myrelease mychart \
      --set prometheus.ingress.annotations."kubernetes\.io/ingress\.class"=nginx \
      --set prometheus.ingress.annotations."kubernetes\.io/tls-acme"="true"

等效于

    prometheus:
      ingress:
        annotations: {}
          kubernetes.io/ingress.class: nginx
          kubernetes.io/tls-acme: 





















