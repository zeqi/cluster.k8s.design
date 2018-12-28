The design project for kubernetes cluster
### Download
```sh
$ git clone https://github.com/zeqi/cluster.k8s.design.git
```

# 私有云k8s集群搭建记录
## 一、docker
### docker常用操作
>docker容器的日常维护、管理等

* 容器和镜像独立操作
```sh
# 创建容器
```

* 容器和镜像的批量操作
```sh
# 停止所有的container，这样才能够删除其中的images：
$ docker stop $(docker ps -a -q)
# 如果想要删除所有container的话再加一个指令：
$ docker rm $(docker ps -a -q)
# 查看当前有些什么images
$ docker images
# 删除images，通过image的id来指定删除谁
$ docker rmi <image id>
# 想要删除untagged images，也就是那些id为<None>的image的话可以用
$ docker rmi $(docker images | grep "^<none>" | awk "{print $3}")
# 要删除全部image的话
$ docker rmi $(docker images -q)
```

## 二、rancher安装
### 三台私有云服务器
>三台服务器的静态ip:`10.140.10.50`、`10.140.10.59`、`10.140.10.61`,系统版本统一为:`CentOS Linux release 7.4.1708`,jdk版本为:`1.8.0_151`,python版本为:`Python 3.6.4 | Python 2.7.5`,golang版本:`go1.8.3`,确定`10.140.10.50`为主机,安装mysql.

* 切换到root下:
```sh
# 所有主机执行
$ sudo su -
```

* 分别在三台服务器的hosts中添加:
```sh
$ vim /etc/hosts
10.140.10.50     IDC-SM-RD-bigdata-k8s-mysql
10.140.10.50     IDC-SM-RD-bigdata-k8s-1
10.140.10.59     IDC-SM-RD-bigdata-k8s-2
10.140.10.61     IDC-SM-RD-bigdata-k8s-3
```

* 分别设置三台主机的名称:
```sh
# 在10.140.10.50
$ vim /etc/sysconfig/network
    HOSTNAME=IDC-SM-RD-bigdata-k8s-1
# 在10.140.10.59
$ vim /etc/sysconfig/network
    HOSTNAME=IDC-SM-RD-bigdata-k8s-2
# 在10.140.10.61
$ vim /etc/sysconfig/network
    HOSTNAME=IDC-SM-RD-bigdata-k8s-3
# 所有主机执行
$ service network restart
```

* 安装docker 17.03.2.ce-1:
```sh
# 所有主机执行
$ yum remove docker docker-common container-selinux docker-selinux docker-engine
$ yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
$ yum-config-manager --enable docker-ce-edge
$ yum install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
$ yum -y install docker-ce-17.03.2.ce-1.el7.centos
$ service docker start
$ systemctl enable docker
```

* 解决无法启动docker服务的问题:
```sh
# 所有主机执行
$ cd /var/lib/docker/
$ rm -rf *
```

* 配置mysql:
```sh
# 所有主机执行
> CREATE DATABASE IF NOT EXISTS cattle COLLATE = 'utf8_general_ci' CHARACTER SET = 'utf8';
> GRANT ALL ON cattle.* TO 'cattle'@'%' IDENTIFIED BY 'cattle';
> GRANT ALL ON cattle.* TO 'cattle'@'localhost' IDENTIFIED BY 'cattle';
> flush privileges;
```

* 部署ruancher:
```sh
# 主节点上执行,之后浏览器访问10.140.10.50:8080进行操作
$ docker run -d --restart=unless-stopped -p 8080:8080 -p 9345:9345 rancher/server \
      --db-host 10.140.10.50 --db-port 3306 --db-user cattle --db-pass cattle --db-name cattle \
      --advertise-address 10.140.10.50
```

## 三、独立部署k8s
### 三台私有云服务器
>三台服务器的静态ip:`10.140.40.22`、`10.140.40.13`、`10.140.40.23`,系统版本统一为:`CentOS Linux release 7.4.1708`,jdk版本为:`1.8.0_151`,python版本为:`Python 3.6.4 | Python 2.7.5`,golang版本:`go1.8.3`,确定`10.140.10.50`为主机.

* 安装docker 17.03.2.ce-1:
```sh
# 所有主机执行
$ yum remove docker docker-common container-selinux docker-selinux docker-engine
$ yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
$ yum-config-manager --enable docker-ce-edge
$ yum install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm
$ yum -y install docker-ce-17.03.2.ce-1.el7.centos
$ service docker start
$ systemctl enable docker
```

* 安装k8s
```sh
# 如果已经系统中已经有kubeadm、kubectl、
$ ./init --node-type master --master-address 10.140.40.22
```

* kubeadm安装
```sh
# 详情见文章:https://segmentfault.com/a/1190000012755243
# https://www.kubernetes.org.cn/docs
$ yum install -y ebtables
$ rpm -ivh socat-1.7.3.2-2.el7.x86_64.rpm
$ rpm -ivh kubernetes-cni-0.6.0-0.x86_64.rpm  kubelet-1.9.9-9.x86_64.rpm
$ rpm -ivh kubectl-1.9.0-0.x86_64.rpm
$ rpm -ivh kubeadm-1.9.0-0.x86_64.rpm
# 确认第一台master三大组件都成功启动
$ kubectl get componentstatuses
$ sed -i "s,ExecStart=$,Environment=\"KUBELET_EXTRA_ARGS=--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1\"\nExecStart=,g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
$ sed -i 's/cgroup-driver=systemd/cgroup-driver=cgroupfs/g' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
$ kubeadm join --token 863f67.19babbff7bfe8543 --discovery-token-unsafe-skip-ca-verification 10.140.40.22:6443
# 查看集群结点状态
$ kubectl get nodes
# 查看详细结点信息
$ kubectl describe nodes
# 查看集群服务状态
$ kubectl get pods --all-namespaces
$ kubectl get pods --namespace="kube-system"
# 确定dns
$ kubectl get svc --namespace=kube-system
$ kubectl exec -it busybox -- /bin/sh
# 查看集群运行在那些ip上
$ kubectl cluster-info
# To further debug and diagnose cluster problems
$ kubectl cluster-info dump
$ kubectl describe pods/kubernetes-dashboard-77bd6c79b-sc5wb --namespace="kube-system" 
$ kubectl get depolyment
$ kubectl get clusterrole/cluster-admin -o yaml
# The connection to the server 10.140.40.22:6443 was refused - did you specify the right host or port? 解决方式
$ service kubelet restart
$ systemctl daemon-reload
$ kubectl create clusterrolebinding login-on-dashboard-with-cluster-admin --clusterrole=cluster-admin --user=admin
# 获取token
$ kubectl get secret -n kube-system
$ kubectl describe secret kubernetes-dashboard-token-j9dpf -n kube-system
$ kubectl create secret generic kubernetes-dashboard-certs --from-file=certs -n kube-system
# kubectl proxy方式
# $ kubectl proxy --address='0.0.0.0' --port=30099 --accept-hosts='^*$'
# dashboard安装https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
# registry.cn-shenzhen.aliyuncs.com/rancher_cn/
# registry.cn-hangzhou.aliyuncs.com/google_containers/
# registry.cn-hangzhou.aliyuncs.com/spacexnice/
# 安装Heapster:https://github.com/kubernetes/heapster/releases
# https://www.cnblogs.com/iiiiher/p/8176769.html
# $ docker pull lanny/k8s.gcr.io_heapster-influxdb-amd64:v1.3.3
# $ docker pull lanny/gcr.io_google_containers_heapster-amd64:v1.5.0
$ docker pull lanny/k8s.gcr.io_heapster-grafana-amd64:v4.4.3
# $ docker pull registry.cn-shenzhen.aliyuncs.com/rancher_cn/heapster-amd64:v1.4.2
$ docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/heapster-influxdb-amd64:v1.3.3
$ docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/heapster-grafana-amd64:v4.4.3
$ kubectl create -f kube-config/influxdb/
$ kubectl create -f kube-config/rbac/heapster-rbac.yaml
```

## 四、spark集群部署
### kubernetes上部署spark
>利用kubespark来部署整套spark计算集群

* 环境配置
```sh
# https://github.com/apache-spark-on-k8s/spark/releases
# 可参考:https://my.oschina.net/blueyuquan/blog/1593060/
# https://www.kubernetes.org.cn/doc-37
# https://zhuanlan.zhihu.com/p/29349351
# https://registry-1.docker.io/v2/
# https://apache-spark-on-k8s.github.io/userdocs/running-on-kubernetes.html
# https://hub.docker.com/u/kubespark/
# https://zhuanlan.zhihu.com/p/29349351
# https://github.com/rootsongjc/kubernetes-handbook
# https://github.com/rootsongjc/kubernetes-handbook/tree/master/manifests/spark-with-kubernetes-native-scheduler
$ docker pull kubespark/spark-driver:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-resource-staging-server:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-init:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-shuffle:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-executor:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-driver-py:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-executor-py:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-driver-r:v2.2.0-kubernetes-0.5.0
    docker pull kubespark/spark-executor-r:v2.2.0-kubernetes-0.5.0
# 下载兼容的spark包,详情见:https://github.com/apache-spark-on-k8s/spark/releases
$ wget https://github.com/apache-spark-on-k8s/spark/releases/download/v2.2.0-kubernetes-0.5.0/spark-2.2.0-k8s-0.5.0-bin-with-hadoop-2.7.3.tgz
$ tar xvf spark-2.2.0-k8s-0.5.0-bin-with-hadoop-2.7.3.tgz
# https://www.kubernetes.org.cn/3238.html
$ kubectl describe clusterrole cluster-admin -n kube-system
# http://blog.csdn.net/chenhaifeng2016/article/details/78813786
# https://github.com/kubernetes/dashboard/wiki/Certificate-management
```

* 配置cluster-spark命名空间
```yaml
# 新建namespace-spark-cluster.yaml文件
# 详细可见yaml/namespace-spark-cluster.yaml
$ vim namespace-spark-cluster.yaml
# 添加如下内容到namespace-spark-cluster.yaml文件中
    apiVersion: v1 
    kind: Namespace
    metadata:
      name: "cluster-spark"
      labels:
        name: "cluster-spark"
$ kubectl create -f namespace-spark-cluster.yaml
# 需要为 spark 集群创建一个 serviceaccount 和 clusterrolebinding：
# 同时新建一个spark用户
$ kubectl create serviceaccount spark --namespace cluster-spark
# $ kubectl create rolebinding spark-edit --clusterrole=edit --serviceaccount=cluster-spark:spark --namespace=cluster-spark
$ kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=cluster-spark:spark --namespace=cluster-spark
```

* 提交spark任务
```sh
# 亲测这个成功了
$ bin/spark-submit \
  --deploy-mode cluster \
  --class org.apache.spark.examples.SparkTC \
  --master k8s://https://10.140.40.22:6443 \
  --kubernetes-namespace cluster-spark \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-tc \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver:v2.2.0-kubernetes-0.5.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor:v2.2.0-kubernetes-0.5.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.5.0 \
  --conf spark.kubernetes.resourceStagingServer.uri=http://10.140.40.22:31000 \
  examples/jars/spark-examples_2.11-2.2.0-k8s-0.5.0.jar
# 执行成功
$ bin/spark-submit \
  --deploy-mode cluster \
  --master k8s://https://10.140.40.22:6443 \
  --kubernetes-namespace cluster-spark \
  --conf spark.kubernetes.authenticate.driver.serviceAccountName=spark \
  --conf spark.executor.instances=5 \
  --conf spark.app.name=spark-pi \
  --conf spark.kubernetes.driver.docker.image=kubespark/spark-driver-py:v2.2.0-kubernetes-0.5.0 \
  --conf spark.kubernetes.executor.docker.image=kubespark/spark-executor-py:v2.2.0-kubernetes-0.5.0 \
  --conf spark.kubernetes.initcontainer.docker.image=kubespark/spark-init:v2.2.0-kubernetes-0.5.0 \
  --conf spark.kubernetes.resourceStagingServer.uri=http://10.140.40.22:31000 \
  --jars examples/jars/spark-examples_2.11-2.2.0-k8s-0.5.0.jar \
  examples/src/main/python/pi.py 10
```

## 四、storm集群部署
### kubernetes上部署storm
>利用storm容器镜像来部署整套storm计算集群

* 环境配置
```sh
# https://github.com/kubernetes/examples/tree/master/staging
# https://github.com/kubernetes/examples/tree/master/staging/cluster-dns
# https://github.com/kubernetes/examples/tree/master/staging/storm
```

* 服务配置
```sh
# zookeeper
# storm
```

* CI/CD
```sh
# https://helpcdn.aliyun.com/document_detail/56336.html
# https://gitlab.com/gitlab-org/kubernetes-gitlab-demo
```