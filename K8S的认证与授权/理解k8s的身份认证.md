[TOC]
## k8s的认证
认证英文名是authetification,  
认证就是对身份的认证，身份是签名证书，所以认证就是对签名证书的验证。
签名证书放在secrets或token里，secrets或token都放在SeviceAccout里。

### 1. ServiceAcount
K8s世界里的所有东西，都是资源，都要用yaml文件描述，
描述里必须有三个描述项：
```gotemplate
kind:
apiversion:
metadata:
描述资源的类型，api版本号，元数据， 
```
因为所有的资源都是通过api来操作，比如create， get等所有的CRUD操作，所以必须有api版本号，
通过api, 可以远程操控资源， 
即restful api范畴里的东西，资源必须有统一标识，k8s里的资源用名字和标签来标识。
因为所有的资源都要有名字和标签，这些都放在元数据里。 
如下是一个自定义的serviceaccount. 
```gotemplate

apiVersion: v1
kind: ServiceAccount
metadata:
  name: heketi-service-account

```
查看当前所有ServiceAccout： 
```
[master@212-node ~]$ k get sa
NAME                     SECRETS   AGE
default                  1         29h
heketi-service-account   1         28h
```

查看名字为default的系统默认Serviceaccount的详细描述，
每个pod都有这样一个默认的Serviceaccount
```
[master@212-node ~]$ kd sa default  
（自定义了kd, 是kubectl describe的别名，方便敲命令）
Name:                default
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-24p4w
Tokens:              default-token-24p4w
Events:              <none>
```

ServieAccount是完成身份认证用的，包含了多个身份
里面的secrets是一个身份，tokens也是一个身份
上面的serviceaccount有两个身份, 一个是Mountable secrets， 一个是Tokens
这两个身份的签名是相同, 因为名字都为default-token-24p4w

每个pod都有自己的servicecount,
pod所有的身份都放在这个servieAccount集中管理
pod通过拥有ServiceAccount， 就拥有了这个ServiceAccount包含的所有身份
如果没有指定用哪个serviceaccount, 系统就将系统默认的serviceaccount给这个pod。
这个默认的serviceaccount的name为defaut. 

pod在调用apiserver,挂载数据卷时，都需要pod的serviceaccount提供自己的身份，
pod做不同的事情时，需要用不同的身份:
pod调用apiserver,需要自己的身份一， 
挂载数据卷时，需要身份二.

### 2. pod和apiserver之间的双向认证
Mountable secrets和Tokens这两个身份的签名都是default-token-24p4w
这个签名有两个证书，来完成pod和apiserver的双向验证
看下这个签名的具体内容：
```gotemplate

[master@212-node ~]$ kd secrets default-token-24p4w
Name:         default-token-24p4w
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: 04b711c6-8288-4d69-ba18-ffc486bbe55b

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IkJGTTcyYmN1QklqZm1HS3JxZVZIaHFvQnF6NUxCWldnMUhGRjFvM3RnUUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tMjRwNHciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjA0YjcxMWM2LTgyODgtNGQ2OS1iYTE4LWZmYzQ4NmJiZTU1YiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.Ir889LHIkOClCdUkEKKIsdxNELiIeGQi8_GUrEI7F7r9NzOqgG1fjxNYBwXQCgKnTHuzhXtUXC_BtBL1BHFR-_hTIWrbVvGmYgaII6VSCiGctdgUyrUHKMdFbJ6gqml-17zjktpp94ZTSyZjTfcb5C6IAwvhfgNP6hT-B_9MAhT9_rKTyg4VXmcvgxryxxArBOwaExO-yRyE7OD0a9jATwx9-ICdvM4khlfzzQz8lbnExvzEmamAcL_BbwgVeEC-AviH1_jZkfHyKoUSwGfZ8qu8iP-HT3fhHRiwO15T4lKnPjMjN1Dyd-wTsMk-gZwrhZX1lsgrMj3HBToeiG6Fjgß
```

每个pod包含的serviceaccount信息都放在pod的
/run/secrets/kubernetes.io/serviceaccount/目录下，
该目录下有三个文件：ca.crt, token, namespace. 

进到pod里面看一下这三个文件：
```
[master@212-node ~]$ k exec -it glusterfs-jhxhp bash
[root@217-node /]# ls /run/secrets/kubernetes.io/serviceaccount/ca.crt -l
lrwxrwxrwx. 1 root root 13 Jan 13 15:09 /run/secrets/kubernetes.io/serviceaccount/ca.crt -> ..data/ca.crt
[root@217-node /]# cat /run/secrets/kubernetes.io/serviceaccount/ca.crt
-----BEGIN CERTIFICATE-----
MIICyDCCAbCgAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
cm5ldGVzMB4XDTIwMDExMTA0MTcxMVoXDTMwMDEwODA0MTcxMVowFTETMBEGA1UE
AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAPQi
v5lz0Qq5Tl4ovDj1JddqZ2aKgv/4WS4dDqeDG6XhVCvAJQQlg6wi4A3Lpq9lbXa5
g0T+7HJy7DjBcsVi/SqTcg8K68X1U1u6If/Fj8BQh3HyOnP4esYW1WCyO4udHb3x
Aj143zKsRtiLD40aM0QpNOrJKLpc1/wDnGSA78DMgU58n1otxgUjhFb6NC9h6q8N
PWMDL0BkoLdSVIMwR4l8GswJJ6bvcm+rKlsEd9E/61F8Xj3ZNg7RAZ+kWNnVHDj3
MJTtSuUZWw7TBKlQUZ13C1d6+VnG9ouB4IOaQLKzOrQK0xLQSLAX2mzX5Hv9EZR9
qBaT9SAkG6KPieMCyjUCAwEAAaMjMCEwDgYDVR0PAQH/BAQDAgKkMA8GA1UdEwEB
/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEBACO+LMXTbfwneo63utcqv1nlUCrG
Ks0o1DYxqVWY4jpMzJmw19TvmHHKiQRfRj8A1Y65PbnGbZ0tAhtpMUJcBn60ubDr
/5xtv+uzJVnuubhkVZYV314D6efj0AgGLb1SykOb/03NCm2zVOJsfFIuczOQoAyc
iTKiaqvMswo4nm0j3JPmz91tkO/j6/gCixgAGVMenNfnmvW6ZgfXVbvw0brhIWvT
WqZNduFTPRYnY/R7UV1bU4oAgaU5twwcRIvldgr++5BEaVcHsR3x0GB4OWt2/oEr
o62ulQf5BjQnQvsPN33v9F0sgTjSq2QOJZ66j3t99p8GZxO+aIYHuk3Pgvc=
-----END CERTIFICATE-----  
```
ca.crt这个证书文件是用来验证apiserver发来的签名证书的，pod以此确定apiserver的身份是对的   
再看token:
```
[root@217-node /]# cat /run/secrets/kubernetes.io/serviceaccount/token
eyJhbGciOiJSUzI1NiIsImtpZCI6IkJGTTcyYmN1QklqZm1HS3JxZVZIaHFvQnF6NUxCWldnMUhGRjFvM3RnUUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tMjRwNHciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjA0YjcxMWM2LTgyODgtNGQ2OS1iYTE4LWZmYzQ4NmJiZTU1YiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.Ir889LHIkOClCdUkEKKIsdxNELiIeGQi8_GUrEI7F7r9NzOqgG1fjxNYBwXQCgKnTHuzhXtUXC_BtBL1BHFR-_hTIWrbVvGmYgaII6VSCiGctdgUyrUHKMdFbJ6gqml-17zjktpp94ZTSyZjTfcb5C6IAwvhfgNP6hT-B_9MAhT9_rKTyg4VXmcvgxryxxArBOwaExO-yRyE7OD0a9jATwx9-ICdvM4khlfzzQz8lbnExvzEmamAcL_BbwgVeEC-AviH1_jZkfHyKoUSwGfZ8qu8iP-HT3fhHRiwO15T4lKnPjMjN1Dyd-wTsMk-gZwrhZX1lsgrMj3HBToeiG6Fjg   

```
这个token就是pod的签名证书，只不过是JWT格式的，
pod的这个签名证书发给apiserver，apiserver以此确定pod的身份是对的。 
这样就完成了pod与apiserver的双向身份验证，这是k8s的高安全性的基础。
    
serviceAccount只能在指定的namespace下有效，
有专门的namespace文件描述该serviceaccount所在的namespace
```
[root@217-node /]# cat /run/secrets/kubernetes.io/serviceaccount/namespace
default
```

## k8s的授权    
一个角色就是一组权限的集合，
比如市长对交通局的局长有一定的权限，如果市长标签绑定到一个人，他就有了市长这个权限
k8s世界里，也用角色来管理权限， 
角色绑定到一个实体对象， 这个对象就有了权限，
所以把角色绑定到某个对象， 就是对某个对象授权。
这个实体对象可以是user, 也可以是serviceaccount. 
为叙述方便 ， serviceaccount简称sa
user是给人用的， sa是给pod用的， pod 都用有默认的sa如果需要特定的sa可以在pod定义文件yaml文件指定
pod拥有了sa, 就拥有了绑定到这个sa上的所有角色的权限。 
    角色有个体角色和群集角色，角色也是k8s的资源，可以创建查看：
查看所有个体角色
```gotemplate

[master@212-node ~]$ k get role
No resources found in default namespace.
```

查看所有集群角色：
```
[master@212-node ~]$ k get clusterrole
NAME                                                                   AGE
admin                                                                  29h
calico-kube-controllers                                                29h
calico-node                                                            29h
cluster-admin                                                          29h
edit                                                                   29h
flannel                                                                29h
foo                                                                    115m
system:aggregate-to-admin                                              29h
system:aggregate-to-edit                                               29h
system:aggregate-to-view                                               29h
system:auth-delegator                                                  29h
system:basic-user                                                      29h
system:certificates.k8s.io:certificatesigningrequests:nodeclient       29h
system:certificates.k8s.io:certificatesigningrequests:selfnodeclient   29h
system:controller:attachdetach-controller                              29h
system:controller:certificate-controller                               29h
system:controller:clusterrole-aggregation-controller                   29h
system:controller:cronjob-controller                                   29h
system:controller:daemon-set-controller                                29h
system:controller:deployment-controller                                29h
system:controller:disruption-controller                                29h
system:controller:endpoint-controller                                  29h
system:controller:expand-controller                                    29h
system:controller:generic-garbage-collector                            29h
system:controller:horizontal-pod-autoscaler                            29h
system:controller:job-controller                                       29h
system:controller:namespace-controller                                 29h
system:controller:node-controller                                      29h
system:controller:persistent-volume-binder                             29h
system:controller:pod-garbage-collector                                29h
system:controller:pv-protection-controller                             29h
system:controller:pvc-protection-controller                            29h
system:controller:replicaset-controller                                29h
system:controller:replication-controller                               29h
system:controller:resourcequota-controller                             29h
system:controller:route-controller                                     29h
system:controller:service-account-controller                           29h
system:controller:service-controller                                   29h
system:controller:statefulset-controller                               29h
system:controller:ttl-controller                                       29h
system:coredns                                                         29h
system:discovery                                                       29h
system:heapster                                                        29h
system:kube-aggregator                                                 29h
system:kube-controller-manager                                         29h
system:kube-dns                                                        29h
system:kube-scheduler                                                  29h
system:kubelet-api-admin                                               29h
system:node                                                            29h
system:node-bootstrapper                                               29h
system:node-problem-detector                                           29h
system:node-proxier                                                    29h
system:persistent-volume-provisioner                                   29h
system:public-info-viewer                                              29h
system:volume-scheduler                                                29h
view                                                                   29h
```

查看某一个集群角色的详细描述：
```
[master@212-node ~]$ kd clusterrole foo
Name:         foo
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources    Non-Resource URLs  Resource Names  Verbs
  ---------    -----------------  --------------  -----
  pods/exec    []                 []              [get list watch create]
  pods/status  []                 []              [get list watch create]
  pods         []                 []              [get list watch create]
```

创建一个集群角色绑定，即将集群角色foo绑定到指定的serviceaccount,  servicecount 只存在与一个namespace, 不能像角色一样全局存在，所以要带上namespace名字，构成全名，
这里的指定的serviceaccount的全名就default:heketi-service-account 
```
[master@212-node ~]$ k create clusterrolebinding my-ca-view --clusterrole=foo --serviceaccount=default:heketi-service-account --namespace=default
clusterrolebinding.rbac.authorization.k8s.io/my-ca-view created

```
### serviceaccount与secrets 
```
[master@212-node ~]$ k get secrets
NAME                                 TYPE                                  DATA   AGE
default-token-24p4w                  kubernetes.io/service-account-token   3      29h
heketi-service-account-token-h9rzt   kubernetes.io/service-account-token   3      28h
```
