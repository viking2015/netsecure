[TOC]
## poster主要做的事情
poster 做windowpost 和出块两件事情， 
### 一， 对sector做window post证明， 
P4消息发到链上后， 
从浏览器看到provencommit的success，  表示这个消息所对应的sector的信息已经上链了, 简称sector上链
同时sectors表能看到proving, 

sector上链之后， poster就可以对这个sector做windowpost证明， 不是立即做， 要等一会. 而且以后要每隔固定时间， 都要做windowpost证明 

从poster.log看到：
```
2020-06-19T10:32:56.116+0800	INFO	storageminer	generating windowPost	{"sectors": 4}
2020-06-19T10:32:56.117+0800	INFO	storageminer	submitting window PoSt	{"elapsed": 0.000443517}
2020-06-19T10:36:08.098+0800	INFO	storageminer	running windowPost	{"chain-random": "5ayz5dvY4RxqH1gBB1LkirBOoXwWmcGgfd2Nf4C6Zic=", "deadline": {"CurrentEpoch":43710,"PeriodStart":40830,"Index":30,"Open":43710,"Close":43806,"Challenge":43690,"FaultCutoff":43690}, "height": "43710", "skipped": 0}
202
```
poster对sector做完证明后， 会把证明完成的消息发到链上， 这个消息的method叫SubmitWindowedPoSt, 网页用中文显示，可以看到提交时空证明。 P4提交的证明叫做数据提交证明。 

poster做的这个证明叫postwindow证明， 也叫时空证明。  
时空证明可以简单的这样理解：比如用户32G数据保存在我这里，我要每过一段时间，证明数据在我这里， 没有丢失， 这个叫时空证明， 

从 proving deadlines 可以看到 window post的证明情况
```
[fil@yangzhou010010019017 ~]$ ./lotus-storage-miner proving deadlines
deadline  sectors  partitions  proven
0         0        0           0
1         36       18          0
2         36       18          0
3         36       18          0
4         36       18          0
5         36       18          0
```
现在是每隔40分钟证明一次，一次证明一个partition, 一个partion包括两个sector. proven是证明的partiton的个数 

poster完成一次证明， 会向链上发一条SubmitWindowedPoSt 消息， 可以在浏览器消息里method列看到这个消息。 

### 二， 出块
出块，是本节点获得出块权， 将链上消息打包。 

一个高度对应一个tipset,  一个tipst可以由多个块组成， 目前规定一个tipset最多有5个块，  从浏览器可以看到每个高度的tipset由几个块组成

./lotus chain list 每一行就是一个高度：
包括高度值， 高度上每个快的的hash值，和出该块的矿工号
```
[fil@yangzhou010010019017 ~]$ ./lotus chain list
43750: (Jun 19 10:37:13) [ bafy2bzacednb2uww5i57guren4muxmmujir7i3ixrxzgf6njtr3f7libadivm: t01003, ]
```

矿工在出块前要对此块与前一个块之间的所有消息进行打包，  

每个节点向链上发消息， 都要为消息处理付一定的费用

此块与前一个块的消息的所有缴纳的费用全部都转入此次消息打包的矿工账号 


#### poster读取的文件
poster 需要读取的证明文件； /mnt/nfs/10.10.4.23/caches
## poster问题排查
### 正常的 miner info
应该没有faulty, 即没有算力惩罚
```
[fil@yangzhou010010011031 ~]$ ./lotus-storage-miner info
Mode: poster
Miner: t01005
Sector Size: 512 MiB
Byte Power:   850 GiB / 1.157 TiB (71.7275%)
Actual Power: 850 Gi / 1.13 Ti (73.6957%)
	Committed: 850 GiB
	Proving: 850 GiB
Expected block win rate: 3456.0000/day (every 25s)

Miner Balance: 3620.283906987152187052
	PreCommit:   0
	Locked:      3615.242626826665635249
	Available:   5.041280160486551803
Worker Balance: 950.78347599741957621
Market (Escrow):  0
Market (Locked):  0
```

### 判断poster是否正常
#### 1. 查看算力
```
[fil@yangzhou010010019017 ~]$ ./lotus-storage-miner info
Mode: poster
Miner: t01003
Sector Size: 2 KiB
Byte Power:   588 KiB / 600 KiB (98.0000%)
Actual Power: 588 Ki / 690 Ki (85.2173%)
	Committed: 2.69 MiB
	Proving: 588 KiB (2.12 MiB Faulty, 78.66%)
Expected block win rate: 43200.0000/day (every 2s)

Miner Balance: 52812.59398882125816547
	PreCommit:   0
	Locked:      52211.965497554934544959
	Available:   600.628491266323620511
Worker Balance: 945.389642209971088202
Market (Escrow):  0
Market (Locked):  0
```
上面的信息详细解释：
```
[fil@yangzhou010010019017 ~]$ ./lotus-storage-miner info
Mode: poster
表示是一个poster

Miner: t01003

Sector Size: 2 KiB
sector大小为2Kb， 目前真实网络的sector是32G

Byte Power:   588 KiB / 600 KiB (98.0000%)。
原值算力, 600KiB 是全网的总算力， 即全网所有sector的总大小。 
算力是sector大小的倍数， 算力也可以是百分比， 即占全网的算力的百分数
 
Actual Power: 588 Ki / 690 Ki (85.2173%)。
有效算力，有写入，会被奖励， 会被乘10.  这里没有写入，和原值算力相等

	Committed: 2.69 MiB  
   已经上链的sector总大小有2.6M， 上链是指sector的消息，sector的位置，大小等所有     信息都在这个消息里

	Proving: 588 KiB (2.12 MiB Faulty, 78.66%)
   虽然有很多sector已经上链，但其中很多sector没有被poster做时空证明，即poster没有证明这些sector还存在。
   这里完成证明的sector只有588K， 没有被证明的sector有2.12M， 即78%没有被证明， sector上链了， 但没有证明， 算力会被惩罚，所以算力到了588K， 就没有增长。
	
Expected block win rate: 43200.0000/day (every 2s)

Miner Balance: 52812.59398882125816547
	PreCommit:   0
	Locked:      52211.965497554934544959
	Available:   600.628491266323620511
Worker Balance: 945.389642209971088202
Market (Escrow):  0
Market (Locked):  0
```
所以commited 很多， 而proving很少时， 就说明 poster不正常了。 
    

#### 2. 查看proven
查看proven， 即查看poster的证明情况
```
[fil@yangzhou010010019017 ~]$ ./lotus-storage-miner proving deadlines
deadline  sectors  partitions  proven
0         0        0           0
1         36       18          0
2         36       18          0
3         36       18          0
4         36       18          0
5         36       18          0
6         36       18          0
7         36       18          0
8         36       18          0
9         36       18          0
10        36       18          0
```

poster每隔40分钟， 做一次任务， 36个sector需要证明， 但是proven为0， 表示没有被证明， 


#### 3. 查看poster.log
证明的任务由poster完成， 查poster.log: 
```
[fil@yangzhou010010019017 ~]$ cat poster.log | grep -ai error
2020-06-19T10:23:20.163+0800	ERROR	storageminer	when running wdpost for partition 466, got err could not read from path="/mnt/218/cache/s-t01003-2218/p_aux"
    No such file or directory (os error 2)
```
poster会把证明文件存放到一个路径下 ，这个路径拼接而成. 

poster会调用lotus-server获取， lotus-server读数据库获取， 返回给poster

lotus-server读取的表为storage-nodes表：
```
1	10.10.4.23	1	10.10.4.23	nfs
```
最后拼接出的路径为:
```
/mnt/nfs/10.10.4.23/caches
```

据此判断， poster启动时没有带上server-api,  制定参数的地方
```
[fil@yangzhou010010019017 ~]$ ps -ef | grep lotus
fil       7050     1  1 Jun18 ?        00:18:18 ./lotus-storage-miner run --mode=poster --dist-path=/mnt --nosync
fil       9383 44996  0 11:39 pts/2    00:00:00 grep --color=auto lotus
root     28231     1  1 Jun18 ?        00:19:54 ./lotus-storage-miner run --mode=remote-sealer --server-api=http://10.10.19.17:3456 --dist-path=/mnt --nosync --groups=1
fil      33248     1  0 Jun18 ?        00:06:58 ./lotus-server
fil      45161     1 13 Jun18 ?        02:40:14 ./lotus daemon --server-api=http://10.10.19.17:3456 --bootstrap=false --api 11234
[fil@yangzhou0100
```

poster server-api参数没有指定， 
重启poseter, 带上server-API， 

数据库都是被server-api读取， 然后poster再从server-api读取， 拼接成存储证明文件的地址为
1	10.10.4.23	1	10.10.4.23	nf
/mnt/nfs/10.10.4.23/caches/xxx



### miner info  大量的证明失败

poster.log里搜索WinningPoSt
```
2020-06-24T20:16:30.005+0800    ^[[34mINFO^[[0m storageminer    storage/miner.go:201    Computing WinningPoSt ;[{SealProof:2 SectorNumber:166 SealedCID:bafk4ehzayheivjmctu6iztzieuwankmnfa3tdymzzpcwhapvbvckh5fqcaiq}]; [239 110 152 230 198 206 75 251 23 202 61 8 249 200 122 50 114 38 154 247 102 211 94 127 251 229 191 43 179 249 157 103]
2020-06-24T20:16:30.005+0800    ^[[34mINFO^[[0m localprover     local/prover.go:57      run GenerateWinningPoSt ...
2020-06-24T20:16:30.009+0800    ^[[31mERROR^[[0m        miner   miner/miner.go:165      mining block failed: failed to compute winning post proof:
    github.com/filecoin-project/lotus/miner.(*Miner).mineOne
        /builds/ForceMining/lotus-force/miner/miner.go:340
  - could not read from path="/mnt/nfs/10.10.13.22/cache/s-t01004-166/p_aux"

    Caused by:
        No such file or directory (os error 2)
    github.com/filecoin-project/filecoin-ffi.GenerateWinningPoSt
        /builds/ForceMining/lotus-force/extern/filecoin-ffi/proofs.go:545
    github.com/filecoin-project/lotus/force/fstorage/fsealmgr/fprover/local.(*ForceProver).GenerateWinningPoSt
        /builds/ForceMining/lotus-force/force/fstorage/fsealmgr/fprover/local/prover.go:65
    github.com/filecoin-project/lotus/storage.(*StorageWpp).ComputeProof
        /builds/ForceMining/lotus-force/storage/miner.go:204
    github.com/filecoin-project/lotus/miner.(*Miner).mineOne
        /builds/ForceMining/lotus-force/miner/miner.go:338
    github.com/filecoin-project/lotus/miner.(*Miner).mine
        /builds/ForceMining/lotus-force/miner/miner.go:163
    runtime.goexit
        /usr/local/go/src/runtime/asm_amd64.s:1373
2020-06-24T20:16:31.011+0800    ^[[33mWARN^[[0m miner   miner/miner.go:157      BestMiningCandidate from the previous round: [bafy2bzaced4gh6irlxu7motqrqsocrr4tn33l5rldplhxzqf6pitxku655sj2] (nulls:0)
```
发现证明参数cache文件不在了， 这些文件被worker错误的删除了


```
[fil@yangzhou010010011031 ~]$ ./lotus-storage-miner proving deadlines
deadline  sectors  partitions  proven
0         0        0           0  (current)
1         3        2           0
2         2        1           0
3         2        1           0
4         2        1           0
5         2        1           0
6         2        1           0
7         2        1           0
8         2        1           0
9         2        1           0
10        2        1           0
11        2        1           0
12        2        1           0
13        2        1           0
14        2        1           0
15        2        1           0
16        2        1           0
17        2        1           0
18        2        1           0
```


####  poster.log报错infoKey not exist

```
2020-07-13T11:00:42.875+0800  ERROR  fminer  fminer/baseinfo.go:124  infoKey not exist ...
2020-07-13T11:00:42.875+0800  ERROR  fminer  fminer/baseinfo.go:144  infoKey not exist ...   这个error
```
原因：
base 和 baseinfo 之前对应， 有个key
baseinfo   会做computeproof, 
base 在 25 秒到的时候， 会删除key
但baseinfo 还在做， 回头去找这个key,  这个key已经被base 删了， 所以报error了

