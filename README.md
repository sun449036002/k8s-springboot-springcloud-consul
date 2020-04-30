## 前提条件：

1.已经安装K8S，参考：[https://kuboard.cn/install/install-k8s.html#%E7%A7%BB%E9%99%A4worker%E8%8A%82%E7%82%B9%E5%B9%B6%E9%87%8D%E8%AF%95](https://kuboard.cn/install/install-k8s.html#%E7%A7%BB%E9%99%A4worker%E8%8A%82%E7%82%B9%E5%B9%B6%E9%87%8D%E8%AF%95)

2.安装了NFS服务器：[https://kuboard.cn/learning/k8s-intermediate/persistent/nfs.html#%E5%9C%A8kuboard%E4%B8%AD%E5%88%9B%E5%BB%BA-nfs-%E5%AD%98%E5%82%A8%E7%B1%BB](https://kuboard.cn/learning/k8s-intermediate/persistent/nfs.html#%E5%9C%A8kuboard%E4%B8%AD%E5%88%9B%E5%BB%BA-nfs-%E5%AD%98%E5%82%A8%E7%B1%BB)
3. 已配置好docker本地私有仓库,假设当前的docker私仓地址为192.168.10.240:8050
```yml
# 可直接通过docker-compose -f registry.yaml up 创建一个本地私有仓库
# 将此文件命名为 registry.yaml
version: '2.1'
services:
  registry:
    image: registry
    container_name: my_registry
    volumes:
      - ./registry:/var/lib/registry
      - ./auth:/auth
    environment:
      - REGISTRY_AUTH=htpasswd
      - REGISTRY_AUTH_HTPASSWD_REALM=Registry_Realm
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/passwd
    restart: always
    privileged: true
    ports:
      - "8050:5000"
```
4. 下载demo应用程序JAR包 链接：https://pan.baidu.com/s/1eG1qXjPhOYqK80k65LX8Aw 
提取码：kmcb 
5. 下载相关配置文件，https://github.com/sun449036002/k8s-springboot-springcloud-consul.git，并把之前下载的两个jar包，复制到此项目中的根目录(k8s-springboot-springcloud-consul目录)

* * *
### 打包镜像并上传到本地私有仓库
1. 先登录本地仓库，按提示输入账号和密码
```sh 
docker login 192.168.10.240:8050 
```
2. 在根目录，根据Dockerfile build 镜像，打上标签并push到私有仓库, *注意build时末尾的点号*
```sh
docker build -t shop-api .
docker tag shop-api 192.168.10.240:8050/sym/shop-api
docker push 192.168.10.240:8050/sym/shop-api

docker build -t shop-user-center -f Dockerfile4UserCenter .
docker tag shop-user-center 192.168.10.240:8050/sym/shop-user-center
docker push 192.168.10.240:8050/sym/shop-user-center
```
3. 创建k8s中docker pull 时需要使用到的认证名称,账号密码换成自己的搭建时设置的值
```sh
kubectl create secret docker-registry regsecret --docker-server=192.168.10.240:8050 --docker-username=docker --docker-password=123456 --docker-email=5566@qq.com
```
### k8s.yaml配置文件描述：
1. user api 配置说明：![image.png](https://upload-images.jianshu.io/upload_images/3512735-77cc7a6d672d1fba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. mysql、redis 配置说明
mysql的配置文件通过将数据映射到挂载在本地的NFS对应的目录中，
mysql的数据内容直接映射到主机的本地硬盘中![image.png](https://upload-images.jianshu.io/upload_images/3512735-737af1710faf27f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
程序中通过K8S配置的Service名称来连接mysql，故这里的服务名称不能变，同理，redis类似。
![image.png](https://upload-images.jianshu.io/upload_images/3512735-42a174aaa62db6e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3512735-ead8c68e8ccc398b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. consul 配置说明![image.png](https://upload-images.jianshu.io/upload_images/3512735-d95ba4d5e9bdda81.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

进入到 k8s-springboot-springcloud-consul目录，执行启动各服务
```sh
kubectl apply -f k8s.yaml
```
执行后，查看运行状态，
```sh
kubectl get svc,pod -o wide
```
![image.png](https://upload-images.jianshu.io/upload_images/3512735-7fc7a321791d0307.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过上面的端口加本机IP访问 consul后台UI界面
![image.png](https://upload-images.jianshu.io/upload_images/3512735-8f89e412a3f6c888.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


若都是Running说明启动成功， 若有没有在Running状态的，可执行以下命令查看错误原因及日志
```sh
kubectl describe pod/user-api-55f444446c-mmlf6
```
![image.png](https://upload-images.jianshu.io/upload_images/3512735-e997db6b4dfacefb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```sh
# 或者查看 pod中的日志，看是否有具体的报错信息
kubectl logs pod/user-api-55f444446c-mmlf6
```
![image.png](https://upload-images.jianshu.io/upload_images/3512735-b69eafaf3da982c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当查看 pod/user-center-xxxxx的log日志时，发现shop数据库没有创建时，可通过mysql目录连接到mysql，导入sql.sql中的数据，
```sh
# 连接数据的信息
host: 127.0.0.1
port: 31233
user:root
password:123456
```
并重启pod即可，通过kubuctl delete
```sh
#删除容器，使之重启一个
kubectl delete pod/user-center-xxxxxx
```
***
## 后续数据准备，并验证接口
1. 先通过映射出来的端口，连接redis 并往redis插入一条数据token对应的openid oIBgo45C6N1kW8u2ZF1,假如指定token为abvdefg
```sh
# 连接redis信息
# host: 127.0.0.1
# port: 31238
# password:123456
SET user:openid:abvdefg "oIBgo45C6N1kW8u2ZF1"
```

2. postman post 请求updateuserinfo接口存入redis,或者直接命令行中curl请求
```sh
curl --location --request POST 'http://127.0.0.1:31234/updateInfo?token=abvdefg' \
--form 'nickName=abc' \
--form 'avatarUrl=abc123456789' \
--form 'city=上海' \
--form 'gener=1' \
--form 'country=中国' \
--form 'province=上海' \
--form 'language=1'
```
![image.png](https://upload-images.jianshu.io/upload_images/3512735-adabb987f19412a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.最后访问接口查看是否正常获取到用户信息
http://127.0.0.1:31234/user/detail?token=xxxxxxxx

### 本人学习时网上搜罗记录以此， 欢迎骚扰、留言、and 喷我
### 外加效果图：
![image.png](https://upload-images.jianshu.io/upload_images/3512735-9ce7263e6a4909d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



