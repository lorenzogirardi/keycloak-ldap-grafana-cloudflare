# keycloak-ldap-grafana-cloudflare

![graph](https://res.cloudinary.com/ethzero/image/upload/v1735489103/misc/cloudflare-grafana-keycloak-activedirectory.png)


# Intro  

So... keycloak gave me the opportunity to play with multiple components in my infrastructure,  
the following will not just keycloak installation but how this component will work having   
an origin lockdown with cloudflare tunnel that work L7 , plus the kbernetes cluster where keycloak, pgsql and cloudflared are installed,  
plus the usage of federation with MS active directory.  


# Setup  

### MS active directory   
--> installed as a virtual machine in vmware  
![msad](https://res.cloudinary.com/ethzero/image/upload/c_thumb,w_400,g_face/v1735492319/misc/Screenshot_2024-12-29_at_16.43.25.png)     

### Cloudflare   
--> used as WAF and tunnel to avoid the exposure of public ip with cloudlared tunnel    
![cloudflare](https://res.cloudinary.com/ethzero/image/upload/c_thumb,w_400,g_face/v1735492834/misc/Screenshot_2024-12-29_at_18.17.21.png)      

### Grafana   
--> as a virtual machine with prometheus,influxdb and graphite   
![grafana](https://res.cloudinary.com/ethzero/image/upload/c_thumb,w_400,g_face/v1735492317/misc/Screenshot_2024-12-29_at_16.33.10.png) 

### Keycloak   
--> installed as a container in a single namespace with pgsql as well   
![keycloak1](https://res.cloudinary.com/ethzero/image/upload/c_thumb,w_400,g_face/v1735493326/misc/Screenshot_2024-12-29_at_18.28.18.png)   
![keycloak2](https://res.cloudinary.com/ethzero/image/upload/c_thumb,w_400,g_face/v1735492322/misc/Screenshot_2024-12-29_at_16.40.31.png)   



# Configuration   

### Cloudflare   


 ```
    - hostname: auth.k8s.it
      service: http://auth.192.168.1.14.nip.io
      originRequest:
        httpHostHeader: "auth.k8s.it"
        noTLSVerify: true
        http2Origin: true
``` 

Since cloudflared (aka tunnel) is acting also as a ingress/reverseproxy to connect an internal endpoint with Cloudflare edges,  
fist of all we have to expose the new domain that will be used by keycloak.  
Service itself inside kubernetes can be used directly or with the ingress,  
in this scenario is used as an ingress since in the future i'd like to add internal and external domains for the installation.    


### Grafana

```
[auth]
oauth_allow_insecure_email_lookup=true
[auth.generic_oauth]
skip_org_role_sync = true
enabled = true
name = Keycloak-OAuth
allow_sign_up = true
client_id = grafana-oauth
client_secret = CLIENT_ID_SECRET_GENERATED_BY_KEYCLOAK
scopes = openid profile profile roles
email_attribute_path = email
login_attribute_path = username
name_attribute_path = full_name
auth_url = https://auth.k8s.it/realms/h4x0r3d/protocol/openid-connect/auth
token_url = https://auth.k8s.it/realms/h4x0r3d/protocol/openid-connect/token
api_url = https://auth.k8s.it/realms/h4x0r3d/protocol/openid-connect/userinfo
```


### MS active directory     

Here u just need a service account (an user in ad) able to have the access
```CN=ldapbind,OU=ServiceAccount,DC=h4x0r3d,DC=lan ```   


### Keycloak    

#### Kubernetes    


##### Namespace
```
apiVersion: v1
kind: Namespace
metadata:
  name: keycloak
```

##### Secrets
```
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
  namespace: keycloak
type: Opaque
stringData:
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: YOUR_DB_PWD
  POSTGRES_DB: keycloak
---
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-secrets
  namespace: keycloak
type: Opaque
stringData:
  KEYCLOAK_ADMIN: admin
  KEYCLOAK_ADMIN_PASSWORD: ADMIN_PWD
``` 

##### pv pvc for pgsqk
```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: postgres-pv
  namespace: keycloak
  labels:
    type: local
    app: postgres
spec:
  storageClassName: "microk8s-hostpath"
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: keycloak
  labels:
    app: postgres
spec:
  storageClassName: "microk8s-hostpath"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

##### deploy pgsql
```
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: keycloak
  labels:
    app: postgres
spec:
  ports:
  - port: 5432
    name: postgres
  selector:
    app: postgres
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: keycloak
spec:
  selector:
    matchLabels:
      app: postgres
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - image: postgres:latest
        name: postgres
        envFrom:
          - secretRef:
              name: postgres-credentials
        ports:
        - containerPort: 5432
          name: postgres
        securityContext:
          privileged: false
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
            limits:
              memory: 512Mi
              cpu: "1"
            requests:
              memory: 256Mi
              cpu: "0.2"
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

##### deploy keycloak
```
apiVersion: v1
kind: Service
metadata:
  name: keycloak
  namespace: keycloak
  labels:
    app: keycloak
spec:
  ports:
  - name: https
    port: 443
    targetPort: 8080
  selector:
    app: keycloak
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak
  labels:
    app: keycloak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:24.0
        args: ["start-dev"]
        env:
        - name: KEYCLOAK_ADMIN
          valueFrom:
            secretKeyRef:
              key: KEYCLOAK_ADMIN
              name: keycloak-secrets
        - name: KEYCLOAK_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              key: KEYCLOAK_ADMIN_PASSWORD
              name: keycloak-secrets
        - name: KC_PROXY
          value: "edge"
        - name: KC_HTTP_ENABLED
          value: "true"
        - name: KC_HOSTNAME_STRICT
          value: "true"
        - name: KC_HOSTNAME_URL
          value: "https://auth.k8s.it/"
        - name: KC_HOSTNAME_ADMIN_URL
          value: "https://auth.k8s.it/"
        - name: KC_HOSTNAME_STRICT_HTTPS
          value: "true"
        - name: KC_HOSTNAME_STRICT_BACKCHANNEL
          value: "false"
        - name: KC_HEALTH_ENABLED
          value: "true"
        - name: KC_METRICS_ENABLED
          value: "true"
        - name: KC_LOG_LEVEL
          value: INFO
        - name: KC_DB
          value: postgres
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_DB
        - name: KC_DB_URL
          value: jdbc:postgresql://postgres/$(POSTGRES_DB)
        - name: KC_DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_USER
        - name: KC_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-credentials
              key: POSTGRES_PASSWORD
        ports:
        - name: http
          containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 250
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 500
          periodSeconds: 30
        resources:
            limits:
              memory: 512Mi
              cpu: "1"
            requests:
              memory: 256Mi
              cpu: "0.2"
```

##### ingress
```
apiVersion: v1
items:
- apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
    name: keycloak-ext
    namespace: keycloak
  spec:
    rules:
    - host: auth.k8s.it
      http:
        paths:
        - backend:
            service:
              name: keycloak
              port:
                number: 8080
          path: /
          pathType: Prefix
```


#### Configurations    







