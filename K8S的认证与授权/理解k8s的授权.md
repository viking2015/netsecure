## k8s的授权    
一个角色就是一组权限的集合，
比如市长对交通局的局长有一定的权限，如果市长标签绑定到一个人，他就有了市长这个权限
k8s世界里，也用角色来管理权限， 
角色绑定到一个实体对象， 这个对象就有了权限，
所以把角色绑定到某个对象，就是对某个对象授权。
这个实体对象可以是user, 也可以是serviceaccount. 
为叙述方便，serviceaccount简称sa
user是给人用的，sa是给pod用的， 
pod 都用有默认的sa，如果需要特定的sa可以在pod定义文件yaml文件指定
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
3. serviceaccount与secrets 
```
[master@212-node ~]$ k get secrets
NAME                                 TYPE                                  DATA   AGE
default-token-24p4w                  kubernetes.io/service-account-token   3      29h
heketi-service-account-token-h9rzt   kubernetes.io/service-account-token   3      28h
```
