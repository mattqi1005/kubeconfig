# kubeconfig
# **k8s的多集群切换**

2套集群升级，停掉1套集群（升级），使用另外1套集群做业务 蓝绿部署 不间断业务

**master之间的切换是通过配置`kubeconfig`来实现**

查看`kubeconfig`

```sh
[root@vms40 .kube]# kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.31.40:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    namespace: kube-system
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
[root@vms40 .kube]#
```

**kubeconfig的两种实现方式**

:one: 创建kubeconfig文件

- 申请证书

- 创建kubeconfig文件

- 验证kubeconfig文件

:two: 手动复制证书

- 集群环境

  | 序号 | IP                                         | 主机名                             | 角色                                |
  | ---- | ------------------------------------------ | ---------------------------------- | ----------------------------------- |
  | 1    | <font color="#00af50">192.168.31.40</font> | <font color="#00af50">vms40</font> | <font color="#00af50">master</font> |
  | 2    | <font color="#00af50">192.168.31.41</font> | <font color="#00af50">vms41</font> | <font color="#00af50">node1</font>  |
  | 3    | <font color="#00af50">192.168.31.42</font> | <font color="#00af50">vms42</font> | <font color="#00af50">node2</font>  |
  | 4    | <font color=#01b0f1>192.168.66.54</font>   | <font color=#01b0f1>vms54</font>   | <font color=#01b0f1>master</font>   |
  | 5    | <font color=#01b0f1>192.168.66.55</font>   | <font color=#01b0f1>vms55</font>   | <font color=#01b0f1>node1</font>    |
  | 6    | <font color=#01b0f1>192.168.66.56</font>   | <font color=#01b0f1>vms56</font>   | <font color=#01b0f1>node2</font>    |

- 组网架构图

<img src='https://i.loli.net/2021/09/05/oHP1mT9dBcVj265.jpg' align='left' style=' width:800px;height:120 px'/>


- 配置方法

  默认的kubeconfig路径  ~/.kube/config

  - <font color="#00af50">master节点上（vms40）</font>先备份~/.kube/config文件 

    ```sh
    [root@vms40 ~]# cd  ~/.kube/
    [root@vms40 .kube]# cp config config.bak
    ```

  - 编辑  ~/.kube/config

    ```sh
    [root@vms40 .kube]# vim ~/.kube/config
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data:
        server: https://192.168.31.40:6443
      name: kubernetes
    contexts:
    - context:
        cluster: kubernetes
        namespace: kube-system
        user: kubernetes-admin
      name: kubernetes-admin@kubernetes
    current-context: kubernetes-admin@kubernetes
    kind: Config
    preferences: {}
    users:
    - name: kubernetes-admin
      user:
        client-certificate-data:
        client-key-data:
    ```

    &#x1F449; 为了好编辑先把证书的内容删除，编辑好以后再粘贴回来

    `certificate-authority-data:`

    `client-certificate-data:`

    `client-key-data:`

  - 修改为如下

    ```sh
    apiVersion: v1
    clusters:
    - cluster:                                   
        certificate-authority-data:
        server: https://192.168.31.40:6443
      name: cluster1
    - cluster:
        certificate-authority-data:
        server: https://192.168.66.54:6443
      name: cluster2
    contexts:
    - context:
        cluster: cluster1
        namespace: kube-system
        user: admin1
      name: context1
    - context:
        cluster: cluster2
        namespace: kube-system
        user: admin2
      name: context2
    current-context: context1
    kind: Config
    preferences: {}
    users:
    - name: admin1
      user:
        client-certificate-data:
        client-key-data:
    - name: admin2
      user:
        client-certificate-data:
        client-key-data:
    ```

  - 通过 kubectl config get-contexts查看 context

    ```sh
    [root@vms40 .kube]# kubectl config get-contexts
    CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
    *         context1   cluster1   admin1     kube-system
              context2   cluster2   admin2     kube-system
    [root@vms40 .kube]#
    ```

  - 通过  kubectl config use-context context2 <font color=red>**切换**</font>到 context2

    ```sh
    [root@vms40 .kube]# kubectl config use-context context2
    Switched to context "context2".
    [root@vms40 .kube]# kubectl  get nodes
    NAME            STATUS   ROLES                  AGE   VERSION
    vms54.rhce.cc   Ready    control-plane,master   13d   v1.21.0
    vms55.rhce.cc   Ready    <none>                 13d   v1.21.0
    vms56.rhce.cc   Ready    <none>                 13d   v1.21.0
    [root@vms40 .kube]# kubectl config use-context context1
    Switched to context "context1".
    [root@vms40 .kube]# kubectl  get nodes
    NAME            STATUS   ROLES                  AGE   VERSION
    vms40.rhce.cc   Ready    control-plane,master   12d   v1.21.0
    vms41.rhce.cc   Ready    <none>                 12d   v1.21.0
    vms42.rhce.cc   Ready    <none>                 12d   v1.21.0
    [root@vms40 .kube]#
    ```

- 通过kubectx快速切换管理k8s的context

  - 安装方法

    ```sh
    [root@vms40 bin]# cd /usr/bin/
    [root@vms40 bin]# wget https://github.com/ahmetb/kubectx/releases/download/v0.9.4/kubectx
    [root@vms40 bin]# chmod +x kubectx 
    ```

  - 通过 kubectx查看 context

    ```sh
    [root@vms40 .kube]# kubectx
    context1
    context2
    ```

  - 通过  kubectx<font color=red>**切换**</font>到 context

    ```sh
    [root@vms40 .kube]# kubectx context1
    Switched to context "context1".
    [root@vms40 .kube]# kubectx context2
    Switched to context "context2".
    ```
