# 1. 后端代码规范

main.go:  config.Init() 的第二个参数要写上前缀，如 “APISERVER” ，与d.yaml的环境变量相匹配

用viper.GetString("db.addr") 即可获取环境变量（对应 APISERVER_DB_ARRD）

```go
func InitDB()  {
	DB = openDB(viper.GetString("db.username"), viper.GetString("db.password"), viper.GetString("db.addr"), viper.GetString("db.name"))
}

func openDB(username, password, addr, name string) *gorm.DB {
	config := fmt.Sprintf("%s:%s@tcp(%s)/%s?charset=utf8mb4&parseTime=%t&loc=%s",
		username,
		password,
		addr,
		name,
		true,
		"Local")

	db, err := gorm.Open("mysql", config)
	if err != nil {
		fmt.Errorf("Open database failed, %s\n", err.Error())
		panic(err)
	}

	return db
}
```

# 2. 前端代码规范



<center>诶嘿</center>

# 3. 生成镜像

### 后端 Dockerfile

```dockerfile
FROM golang:1.17-alpine
RUN go env -w GO111MODULE=on
RUN go env -w GOPROXY="https://goproxy.cn,direct"
RUN mkdir /app 
ADD . /app
WORKDIR /app
RUN go mod tidy
RUN go build -o /app main.go
CMD ["/app/main"]
```

### 前端 Dockerfile

```dockerfile
FROM node:14.18.1
# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY . /usr/src/app
# Build server file
RUN yarn config set registry https://registry.npm.taobao.org/
RUN yarn install
# Bundle app source
EXPOSE 3000
CMD [ "yarn", "preview" ]
```



**下面是用ci+gitea实现自动化生成镜像，也可以选择docker build&push 来手动生成镜像**



## 阿里云

创建镜像仓库 注意命名空间

## gitea:

前端所需配置文件 d.yaml  s.yaml  .drone.yml文件中的secret需要在ci.muxixyz.com中配置，其中docker_name,docker_password在阿里云上，AK和SK在七牛云上，注意namespace,.drone.yml修改镜像，repo和registry需要同源。

### .drone.yml 

AK SK在七牛云

```yaml
kind: pipeline
type: docker
name: arbor_day_fe

trigger:
  ref:
  - refs/tags/release-*

clone:
  depth: 1
 
steps:
  - name: build-assets
    image: node:14.18.1
    environment:
      REACT_APP_VERSION: ${DRONE_TAG}
    commands:
    - yarn config set registry https://registry.npm.taobao.org
    - yarn install
    - yarn run build
  - name: upload-assets
    image: node
    environment:
      AK:
        from_secret: AK
      SK:
        from_secret: SK
    commands:
    - yarn global add qiniu-upload-tool --registry=https://registry.npm.taobao.org
    - qiniu-upload push -z huad -p ./dist -b ccnustatic -f arbor_day -r yes
    - rm -rf ./src
    - rm -rf ./node_modules
  - name: build-and-push-image # 构建 user 服务镜像
    image: plugins/docker 
    volumes:
      - name: dockersock # 挂载下面定义的Volumn
        path: /var/run/docker.sock # 与宿主机用同一docker
    settings: # plugins/docker用到的相关配置
      username:
        from_secret: docker_user # alicloud指定的docker hub的用户名(前面配置)
      password:
        from_secret: docker_password # alicloud指定的docker hub的密码(前面配置)
      repo: registry-vpc.cn-hangzhou.aliyuncs.com/muxi/arbor_day_fe  #要推送docker地址
      registry: registry-vpc.cn-hangzhou.aliyuncs.com # 使用的docker hub地址
      tags: ${DRONE_TAG} 
volumes:
- name: dockersock
  host: 
    path: /var/run/docker.sock

```

登录 gitea.muxixyz.com root账号

分别创建项目仓库，注意设置为public，并设置自己账号为协作者，并把代码push到gitea仓库地址，在[ci](http://ci.muxixyz.com/)也要创建仓库



代码开发完成后，打tag（命名规范 从0.0.1开始）ci会自动生成镜像并自动上传阿里云镜像仓库



# 4. k8s:

### d.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ?-backend
  namespace: ?
  labels:
    run: ?-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      run: ?-backend
  template:
    metadata:
      labels:
        run: ?-backend
    spec:
      containers:
      - name: ?-backend
        image: registry.cn-hangzhou.aliyuncs.com/muxi_site/elite_hunter_be:release-0.0.8
        env:
        - name: APISERVER_DB_NAME
          value: ?
        - name: APISERVER_DB_ADDR
          value: rm-bp1pkcz9y269h30fm5o.mysql.rds.aliyuncs.com
        - name: APISERVER_DB_USERNAME
          value: ?
        - name: APISERVER_DB_PASSWORD
          value: ...
        ports:
        - containerPort: 8080 # 前端换3000
```

### s.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: ?
  labels:
    run: ?-backend
  name: ?-backend
spec:
  ports:
  - port: 8080
    targetPort: 8080 # 后端 run 的 port，前端都换3000
    protocol: TCP
    name: http
  selector:
    run: ?-backend # select pod of deploy
```

### i.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ?
  namespace: ?
  annotations:
    kubernetes.io/ingress.class: traefik
    cert-manager.io/issuer-name: letsencrypt-prod # 证书管理器
spec:
  rules:
  - host: ?.test.muxixyz.com # 在阿里云注册域名
    http:
      paths:
      - path: /api
        backend:
          serviceName: ?-backend
          servicePort: 8080
      - path: /
        backend:
          serviceName: ?-frontend
          servicePort: 3000
          tls:
  - hosts:
    - ?.test.muxixyz.com
      secretName: ?-secret
```

### certificate.yaml  (https)  --可选项

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: ?
  namespace: ?
spec:
  secretName: ?-secret
  dnsNames:
  - ?.test.muxixyz.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

### kubectl apply -f  （顺序很重要）/replace 

