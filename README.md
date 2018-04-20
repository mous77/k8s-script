# k8s-script

## 配置 hosts 集群ip 信息(双主且都是 Node 测试环境)

* 创建版本 - v20180418
* 当前 K8s 版本 v1.10.1

#### Vagrant vm 机器列表(如果真实机器也是如此)
| hosts | ip            | delopy       |
|:------|:--------------|:-------------|
| node1 | 192.168.33.41 | virtual ware |
| node2 | 192.168.33.42 | virtual ware |
| node3 | 192.168.33.43 | virtual ware |


## Vagrant+Virtualbox

[√] 下载 Vagrantfile
> curl https://raw.githubusercontent.com/laik/k8s-script/master/Vagrantfile > Vagrantfile 

[√] 开始下载准备好的镜像(`tmp/k8s-dev.sh`需要修改里面的用户密码[自己去阿里云搞个用户])
> mkdir tmp &&
> curl https://raw.githubusercontent.com/laik/k8s-script/master/k8s-dev.sh > tmp/k8s-dev.sh

[√] 启动 Vagrant 
> vagrant up

[√] 需要手工初始化kubernetes

第一次初始化失败已经存在配置文件,需要加 `--ignore-preflight-errors=all`(参数跟飞机起飞)

```Shell
# 初始化获取当前版本
curl https://storage.googleapis.com/kubernetes-release/release/stable-1.10.txt

# 我们使用v1.10.1
kubeadm init --apiserver-advertise-address=192.168.33.41--kubernetes-version=v1.10.1 --pod-network-cidr=10.244.0.0/16

# 对于非root用户
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 对于root用户
$ export KUBECONFIG=/etc/kubernetes/admin.conf
# 也可以直接放到~/.bash_profile
$ echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bashrc

# 默认情况下，为了保证master的安全，master是不会被调度到app的。你可以取消这个限制通过输入：
> kubectl taint nodes --all node-role.kubernetes.io/master-

# addons 
接下来要注意，我们必须自己来安装一个network addon。network addon必须在任何app部署之前安装好。同样的，kube-dns也会在network addon安装好之后才启动 kubeadm只支持CNI-based networks（不支持kubenet）。比较常见的network addon有：Calico, Canal, Flannel, Kube-router, Romana, Weave Net等。这里我们使用Calico。

# 用 weave net 安装方法
export kubever=$(kubectl version | base64 | tr -d '\n')
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
systemctl restart docker && systemctl restart kubelet

# 用 Calico 作为网络传输层
kubectl apply -f https://docs.projectcalico.org/v3.0/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
systemctl restart docker && systemctl restart kubelet

# dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
kubectl proxy
```


[√] 清除当前集群信息

```
   kubeadm reset
```

Enjoy it!!


---

[√] FQA(关于在部署的过程中遇到一些问题记录)

> 1. cni 问题 Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
>> 删除 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 中的$ KUBELET_NETWORK_ARGS 参数之后执行
`systemctl daemon-reload && systemctl restart kubelet`


> 2. https 访问 dashboard 证书问题
>> 基于docker镜像部署方式
>>>  source url from github [https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above]
>>> 
>>> This way of accessing Dashboard is only recommended for development environments in a single node setup.
>>> 
>>> Edit kubernetes-dashboard service.
>>> ```
>>> $ kubectl -n kube-system edit service kubernetes-dashboard
>>> ```
>>> You should see yaml representation of the service. Change type: ClusterIP to type: NodePort and save file. If it's already changed go to next step.
>>> 
>>> Please edit the object below. Lines beginning with a '#' will be ignored,
>>> and an empty file will abort the edit. If an error occurs while saving this file will be
>>> #### reopened with the relevant failures.
>>> ```
>>> apiVersion: v1
>>> ...
>>>   name: kubernetes-dashboard
>>>   namespace: kube-system
>>>   resourceVersion: "343478"
>>>   selfLink: /api/v1/namespaces/kube-system/services/kubernetes-dashboard-head
>>>   uid: 8e48f478-993d-11e7-87e0-901b0e532516
>>> spec:
>>>   clusterIP: 10.100.124.90
>>>   externalTrafficPolicy: Cluster
>>>   ports:
>>>   - port: 443
>>>     protocol: TCP
>>>     targetPort: 8443
>>>   selector:
>>>     k8s-app: kubernetes-dashboard
>>>   sessionAffinity: None
>>>   type: ClusterIP
>>> status:
>>>   loadBalancer: {}
>>> ```
>>> Next we need to check port on which Dashboard was exposed.
>>> 
>>> ```
>>> $ kubectl -n kube-system get service kubernetes-dashboard
>>> ```
>>> ```
>>> NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
>>> kubernetes-dashboard   10.100.124.90   <nodes>       443:31707/TCP   21h
>>> Dashboard has been exposed on port 31707 (HTTPS). Now you can access it from your browser at: https://<master-ip>:31707. master-ip can be found by executing kubectl cluster-info. Usually it is either 127.0.0.1 or IP of your machine, assuming that your cluster is running directly on the machine, on which these commands are executed.
>>> ```
>>> 
>>> #### 当改为节点访问10.100.124.90:31707 时会出现"system:serviceaccount:kube-system:kubernetes-dashboard"
>>> 
>>> If you received an error like below, you need to grant access to Kubernetes dashboard to in your cluster.
>>> 
>>> configmaps is forbidden: User "system:serviceaccount:kube-system:kubernetes-dashboard" cannot list configmaps in the namespace "default"
>>> 
>>> If you are planning to access to Kubernetes Dashboard via proxy from remote machine, you will need to grant ClusterRole to allow access to dashboard.
>>> 
>>> Create new file and insert following details.
>>> ```
>>> vi kube-dashboard-access.yaml
>>> ```
>>> ```
>>> apiVersion: rbac.authorization.k8s.io/v1beta1
>>> kind: ClusterRoleBinding
>>> metadata:
>>>   name: kubernetes-dashboard
>>>   labels:
>>>     k8s-app: kubernetes-dashboard
>>> roleRef:
>>>   apiGroup: rbac.authorization.k8s.io
>>>   kind: ClusterRole
>>>   name: cluster-admin
>>> subjects:
>>> - kind: ServiceAccount
>>>   name: kubernetes-dashboard
>>>   namespace: kube-system
>>> ```
>>> Now we will apply changes to Kubernetes Cluster to grant access to dashboard.
>>> ```
>>> kubectl create -f kube-dashboard-access.yaml
>>> ```


>> #### 二进制部署的改造方法(没试过)
>>> ##### Kubernetes API Server新增了 –anonymous-auth 选项，允许匿名请求访问secure port。没有被其他authentication方法拒绝的请求即Anonymous requests， 这样的匿名请求的username为system:anonymous, 归属的组为system:unauthenticated。并且该选线是默认的。这样一来，当采用chrome浏览器访问dashboard UI时很可能无法弹出用户名、密码输入对话框，导致后续authorization失败。为了保证用户名、密码输入对话框的弹出，需要将 –anonymous-auth 设置为 false。
>>> 
>>> #### 解决方法1：
>>> 在api-server配置文件中添加 –anonymous-auth=false
>>> ```
>>> [root@master1 dashboard]# vim /etc/systemd/system/kube-apiserver.service
>>> ```
>>> ```
>>> [Unit]
>>> Description=Kubernetes API Server
>>> Documentation=https://github.com/GoogleCloudPlatform/kubernetes
>>> After=network.target
>>> After=etcd.service
>>> 
>>> [Service]
>>> ExecStart=/usr/local/bin/kube-apiserver \
>>>   --logtostderr=true \
>>>   --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
>>>   --advertise-address=192.168.161.161 \
>>>   --bind-address=192.168.161.161 \
>>>   --insecure-bind-address=127.0.0.1 \
>>>   --authorization-mode=Node,RBAC \
>>>   --anonymous-auth=false \
>>>   --basic-auth-file=/etc/kubernetes/basic_auth_file \
>>>   --runtime-config=rbac.authorization.k8s.io/v1alpha1 \
>>>   --kubelet-https=true \
>>>   --enable-bootstrap-token-auth \
>>>   --token-auth-file=/etc/kubernetes/token.csv \
>>>   --service-cluster-ip-range=10.254.0.0/16 \
>>>   --service-node-port-range=8400-10000 \
>>>   --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
>>>   --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
>>>   --client-ca-file=/etc/kubernetes/ssl/ca.pem \
>>>   --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
>>>   --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
>>>   --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
>>>   --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
>>>   --etcd-servers=https://192.168.161.161:2379,https://192.168.161.162:2379,https://192.168.161.163:2379 \
>>>   --enable-swagger-ui=true \
>>>   --allow-privileged=true \
>>>   --apiserver-count=3 \
>>>   --audit-log-maxage=30 \
>>>   --audit-log-maxbackup=3 \
>>>   --audit-log-maxsize=100 \
>>>   --audit-log-path=/var/lib/audit.log \
>>>   --event-ttl=1h \
>>>   --v=2
>>> Restart=on-failure
>>> RestartSec=5
>>> Type=notify
>>> LimitNOFILE=65536
>>> 
>>> [Install]
>>> WantedBy=multi-user.target
>>> ```
>>> #### Unauthorized问题
>>> 解决了上面那个问题之后，再度访问dashboard页面，发现还是有问题，出现下面这个问题：
>>> ```
>>> {
>>>   "kind": "Status",
>>>   "apiVersion": "v1",
>>>   "metadata": {
>>> 
>>>   },
>>>   "status": "Failure",
>>>   "message": "Unauthorized",
>>>   "reason": "Unauthorized",
>>>   "code": 401
>>> }
>>> ```
>>> 解决方法1：
>>> 新建 /etc/kubernetes/basic_auth_file 文件，并在其中添加：
>>> 
>>> admin123,admin,1002
>>> 文件内容格式：password,username,uid
>>> 
>>> 然后在api-server配置文件（即上面的配置文件）中添加：
>>> ```
>>> --basic-auth-file=/etc/kubernetes/basic_auth_file \
>>> ```
>>> #### 保存重启kube-apiserver：
>>> ```
>>> systemctl daemon-reload
>>> systemctl restart kube-apiserver
>>> systemctl status kube-apiserver
>>> 最后在kubernetes上执行下面这条命令：
>>> ```
>>> ```
>>> kubectl create clusterrolebinding login-dashboard-admin --clusterrole=cluster-admin --user=admin
>>> ```
>>> 将访问账号名admin与dashboard.yaml文件中指定的cluster-admin关联，获得访问权限。
>>> 
>>> 
>>> #### 解决方法2:
>>> ```
>>> nohup kubectl proxy --accept-hosts='^*$' > /tmp/proxy.log 2>&1 &
>>> ```


> #### 3. 关于"Failed to create summary reader for "/system.slice/auditd.service": none of the resources are being tracked."
>> ##### 总结: 以下那么长的解释就是为了在 /etc/systemd/system/kubelet.service.d/10-kubeadm.conf Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice" 加入后面 --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice 这一段
>> 
>> ##### ---------------------分隔线------------------------------------------
>> 
>> I've got the same errors on
>> 
>> CentOS 7.4.1708
>> docker 1.12.6
>> kubeadm v1.9.4
>> Mar 14 08:32:35 ksa-m1.blue kubelet[9322]: E0314 08:32:34.998853    9322 summary.go:92] Failed to get system container stats for "/system.slice/kubelet.service": failed to get cgroup stats for "/system.slice/kubelet.service": failed to get container info for "/system.slice/kubelet.service": unknown container "/system.slice/kubelet.service"
>> Mar 14 08:32:35 ksa-m1.blue kubelet[9322]: E0314 08:32:34.998879    9322 summary.go:92] Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats for "/system.slice/docker.service": failed to get container info for "/system.slice/docker.service": unknown container "/system.slice/docker.service"
>> And the above fix works for me.
>> On CentOS I can add these options in /etc/systemd/system/kubelet.service.d/10-kubeadm.conf:
>> ```
>> egrep KUBELET_CGROUP_ARGS= /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
>> ```
>> ```
>> Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"
>> ```
>> Should we have this fix added to the kubeadm RPM by default?
>> Environment:
>> 
>> cat /etc/redhat-release 
>> CentOS Linux release 7.4.1708 (Core) 
>> 
>> uname -a
>> Linux ksa-m1.blue 3.10.0-514.6.1.el7.x86_64 #1 SMP Wed Jan 18 13:06:36 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
>> 
>> kubectl version
>> Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.4", GitCommit:"bee2d1505c4fe820744d26d41ecd3fdd4a3d6546", GitTreeState:"clean", BuildDate:"2018-03-12T16:29:47Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
>> Server Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.4", GitCommit:"bee2d1505c4fe820744d26d41ecd3fdd4a3d6546", GitTreeState:"clean", BuildDate:"2018-03-12T16:21:35Z", GoVersion:"go1.9.3", Compiler:"gc", Platform:"linux/amd64"}
>> 
>> rpm -qa | egrep kube
>> kubernetes-cni-0.6.0-0.x86_64
>> kubelet-1.9.4-0.x86_64
>> kubectl-1.9.4-0.x86_64
>> kubeadm-1.9.4-0.x86_64
>> 
>> docker version 
>> Client:
>>  Version:         1.12.6
>>  API version:     1.24
>>  Package version: docker-1.12.6-71.git3e8e77d.el7.centos.1.x86_64
>>  Go version:      go1.8.3
>>  Git commit:      3e8e77d/1.12.6
>>  Built:           Tue Jan 30 09:17:00 2018
>>  OS/Arch:         linux/amd64
>> 
>> Server:
>>  Version:         1.12.6
>>  API version:     1.24
>>  Package version: docker-1.12.6-71.git3e8e77d.el7.centos.1.x86_64
>>  Go version:      go1.8.3
>>  Git commit:      3e8e77d/1.12.6
>>  Built:           Tue Jan 30 09:17:00 2018
>>  OS/Arch:         linux/amd64




---


## 常用命令

```Shell
kubectl get po -o wide --all-namespaces

# 查看 Master join Token
kubeadm token create --print-join-command

```

```
kubeadm join ${ipaddr}:${proxy} --token 6y08dg.q977edbxcjepnq68 --discovery-token-ca-cert-hash sha256:003ade97af781e60aba97817f0330f512a531336e604950b694ebfa3fcd0b6cd
```