---
description: Kubernetes cheatsheet, snippets, troubleshooting, tips, hints
---

## Volumes
```yaml
spec:
  template:
    spec:
      volumes:
      - name: plugindir
        emptyDir: {}
        
      containers:
      - name: mongos
        image: mongo:4.0.2
        volumeMounts:
        - name: mongo-socket
          mountPath: /tmp
```

## Resources
```yaml
        resources:
          limits:
            cpu: 1
            memory: 2G
          requests:
            cpu: 1
            memory: 2G
```

## PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: grafana
    spec:
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: standard
          # emptyDir: {}
```

## Affinity
### Pod 只能放在符合以下條件的 Node
```yaml
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: cloud.google.com/gke-preemptible
                operator: NotIn
                values:
                - "true"
              - key: sysctl/vm.max_map_count
                operator: In
                values:
                - "262144"
```

### 同一個 Node 不放一樣的 Pod
```yaml
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - topologyKey: "kubernetes.io/hostname"
              labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - mongo-sh0
```


## env

### metadata.annotations
```yaml
      template:
        metadata:
          annotations:
            version: abc4444
            
        spec:
          containers:
          - name: cronjob
            env:
            - name: MY_VERSION
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['version']
```

### metadata.name metadata.namespace
ref: https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api
```yaml
          env:
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
```

### Secret

Encode/Decode
```bash
echo -n 'mykey' | base64
echo -n 'bXlrZXk=' | base64 -D
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  mykey: bXlrZXk=
```

```yaml
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
    - name: SECRET_USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: mykey
```

### envFrom
```yaml
spec:
  containers:
  - name: http
    image: asia.gcr.io/swag-2c052/swag:42ff66b1
    envFrom:
    - configMapRef:
      name: myconfigmap
    - secretRef:
      name: mysecret
```

### volumeMounts from configmap
```yaml tab="Pod"
    spec:
      volumes:
      - name: config
        configMap:
          name: nginx-config

      containers:
      - name: echoserver
        image: gcr.io/google_containers/echoserver:1.10
        ports:
        - containerPort: 8080

        # nginx.conf override
        volumeMounts:
        - name: config
          subPath: nginx.conf
          mountPath: /etc/nginx/nginx.conf
          # mountPath: /usr/local/openresty/nginx/conf/nginx.conf
          readOnly: true
```

```yaml tab="ConfigMap"
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: ingress-nginx
data:
  nginx.conf: |-
    events {
      worker_connections 1024;
    }
```

## Container Injection

### /etc/hosts and /etc/resolv.conf

ref:

- https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/#adding-additional-entries-with-hostaliases
- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-config

This will append hostname and overwrite /etc/resolv.conf

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: dnsmasq
  labels:
    name: dnsmasq
spec:
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: dnsmasq
    spec:
      hostAliases:
      - ip: "35.227.233.133"
        hostnames:
        - "swag.live"
      dnsPolicy: "None"
      dnsConfig:
        nameservers:
          - 1.1.1.1
          - 8.8.8.8
      containers:
      - image: rammusxu/dnsmasq
        name: dnsmasq
        resources:
          requests:
            cpu: "100m"
            memory: "100M"   
        ports:
        - containerPort:  53
          name: dnsmasq
        - containerPort:  53
          protocol: UDP
          name: dnsmasq-udp
        imagePullPolicy: Always
```

### Container with commands
```yaml
      - name: imagemagick
        image: swaglive/imagemagick:lastest
        command: ["/bin/sh","-c"]
        args: 
        - |
          magick montage -quality 90 -resize 211x292^ -gravity center -crop 211x292+0+0 -geometry +4+4 -tile 7x4 -background none +repage $(cat /tmp/covers.txt) /tmp/montage.jpg
        volumeMounts:
        - name: tmp
          mountPath: /tmp
```

## Frequent Commands
### Gracefully rolling restart deployment.
```bash
kubectl rollout restart deployment/my-sites --namespace=default
```

## Reference
- [feiskyer Handbook](https://kubernetes.feisky.xyz/)
- [feiskyer Examples](https://github.com/feiskyer/kubernetes-handbook/tree/master/examples)