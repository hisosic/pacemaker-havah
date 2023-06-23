# pacemaker-havah
havah-chain-node HA
# 파일 위치

- `/etc/init.d/`
  - havah_active
  - havah_backup
  - havah_status
- `/app/havah-chain-node/`
  - docker-compose.yml



# Pacemaker Install

- Pacemaker 설치 되는 노드 간 양방향 Port Open 필요 합니다. (TCP 22, TCP 2224, TCP 9000, UDP 5404 ~ 5412)


## (ALL) Package Install 및 Pacemaker 활성 (CentOS 및 RHEL 기반)

```
[root@validator01 ~]$ yum install pacemaker pcs resource-agents
[root@validator01 ~]$ systemctl enable pcsd.service
```

:warning: AWS EC2 활용 시 amazon linux kenel 5버전 이하 ami으로 사용 권장 합니다. (kenel 6 epel 패키지 지원 안함)


## (ALL) Package Install (Ubuntu)

```
[root@validator01 ~]$ apt install pacemaker corosync fence-agents pcs
[root@validator01 ~]$ systemctl enable pcsd.service
```


## (ALL) hacluster 계정 패스워드 설정 (P@ssw0rd)

```
[root@validator01 ~]$ passwd hacluster
Changing password for user hacluster.
New password:
BAD PASSWORD: The password contains the user name in some form
Retype new password:
passwd: all authentication tokens updated successfully.
[root@validator01 ~]$
```






# Pacemaker Configure


## (ONE) Cluster 노드 등록

```
[root@validator01 ~]$ pcs cluster auth cluster01 cluster02 <> /dev/tty
Username: hacluster
Password: P@ssw0rd (패스워드 입력)
cluster02: Authorized
cluster01: Authorized
```


## (ONE) Cluster 설정

```
[root@validator01 ~]$ pcs cluster setup --name cluster cluster01 cluster02 --transport udpu
```


## (ONE) Start Cluster and Check Status

```
[root@validator01 ~]$ pcs cluster start --all --wait=60

[root@validator01 ~]$ systemctl enable corosync pacemaker pcsd

[root@validator01 ~]$ pcs status
Cluster name: cluster

WARNINGS:
No stonith devices and stonith-enabled is not false

Stack: corosync
Current DC: cluster02 (version 1.1.23-1.amzn2.1-9acf116022) - partition with quorum
Last updated: Fri Jun  9 01:48:40 2023
Last change: Fri Jun  9 01:47:43 2023 by hacluster via crmd on cluster02

2 nodes configured
0 resource instances configured

Online: [ cluster01 cluster02 ]

No resources


Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```


## (ONE) Configure Cluster CIB (Cluster Information Base)

```
[root@validator01 ~]$ pcs cluster cib tmp-cib.xml

[root@validator01 ~]$ pcs -f tmp-cib.xml property set stonith-enabled=false
```

:warning: After the initial setup, STONITH is enabled by default, but since STONITH (Fencing) is not configured, an error occurs.

It is safe to operate without using the STONITH (Fencing) feature. If you want to use it, please refer to the link below.

[Chapter 10. Configuring fencing in a Red Hat High Availability cluster](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_fencing_in_a_red_hat_high_availability_cluster/index)



# Resource 등록

```
# Active Node resource (*havah_active*) 등록
[root@validator01 ~]$ pcs -f tmp-cib.xml resource create havah_active lsb:havah_active \
  op force-reload interval=0s timeout=60s monitor interval=30s timeout=60s \
  restart interval=0s timeout=60s start interval=0s timeout=60s stop \
  interval=0s
  
# Active Node Status resource (*havah_status*) 등록
[root@validator01 ~]$ pcs -f tmp-cib.xml resource create havah_status lsb:havah_status \
  op force-reload interval=0s timeout=60s monitor interval=30s timeout=60s \
  restart interval=0s start interval=0s stop interval=0s
  
# Backup Node resource (*havah_backup*) 등록
[root@validator01 ~]$ pcs -f tmp-cib.xml resource create havah_backup lsb:havah_backup \
  op force-reload interval=0s timeout=60s monitor interval=30s timeout=60s \
  restart interval=0s timeout=60s start interval=0s timeout=60s stop \
  interval=0s
  
# 만약 개별 리소스에 대해 설정이 필요하다면 op 옵션과 함께 지정해 준다.

# 글로벌 Default 옵션이 필요하다면 pcs resource op defaults timeout=XX 형식으로 지정해 줄 수 있다.

# 글로벌 Default 옵션에 대한 확인은 pcs resource op defaults 로 확인 할 수 있다. (운영환경에서는 권고하지않음)

 

# Group 생성

# Active 그룹을 만들고 Active 관련 (havah_active, havah_status) Resource 를 추가
[root@validator01 ~]$ pcs -f tmp-cib.xml resource group add Active havah_active havah_status

# Backup 그룹을 만들고 Backup 관련 (havah_backup) Resource 를 추가
[root@validator01 ~]$ pcs -f tmp-cib.xml resource group add Backup havah_backup

# Resource Group 설정 ( Active 와 Backup 그룹이 같은 노드에 실행 되지 않도록 )
```
 

 
## Constraint Order(실행순서 규칙) Config 적용
```
[root@validator01 ~]$ pcs -f tmp-cib.xml constraint order havah_active then havah_status
```

## Resource-stickiness (노드 우선사용 규칙) 적용
```
[root@validator01 ~]$ pcs -f tmp-cib.xml resource defaults resource-stickiness=3000
Warning: Defaults do not apply to resources which override them with their own defined values
```

## Config 적용
```
[root@validator01 ~]$ pcs cluster cib-push tmp-cib.xml
```

## Pacemaker Status Check
초기 기본 상태 확인
```
[root@validator01 ~]$ pcs status
Cluster name: cluster
Stack: corosync
Current DC: cluster02 (version 1.1.23-1.amzn2.1-9acf116022) - partition with quorum
Last updated: Fri Jun  9 16:31:14 2023
Last change: Fri Jun  9 16:23:17 2023 by root via cibadmin on cluster02

2 nodes configured
3 resource instances configured (1 DISABLED)

Online: [ cluster01 cluster02 ]

Full list of resources:

 Resource Group: Active
     havah_active	(lsb:havah_active):	Started cluster01
     havah_status	(lsb:havah_status):	Started cluster01
 Resource Group: Backup
     havah_backup	(lsb:havah_backup):	Stopped (disabled)

Failed Resource Actions:

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```


## Cluster Status

- Active Group의 havah_active, havah_status Resource 가 cluster01 서버에서 실행 중 :white_check_mark:
- Backup Group의 havah_backup Resource 가 Stopped(disabled) 되어 있음. :white_check_mark:


## havah_backup Resource 기동

```
[root@validator01 ~]$ pcs resource enable havah_backup
```
- pcs status 로 확인하면 cluster02 노드에서 havah_backup resource 기동 되는게 확인 됨. :white_check_mark:


## Failover 테스트 및 Restore 절차

- cluster01 노드 장애로 인해 Failover 상태 확인.

```
[root@validator01 ~]$ pcs status
```

```
2 nodes configured
3 resource instances configured (1 DISABLED)    ## 활성화 및 비활성화 Resource 개수 표시

Online: [ cluster02 ]
OFFLINE: [ cluster01 ]   ## OFFLINE 상태 인 클러스터 노드 표시

Full list of resources:

 Resource Group: Active
     havah_active	(lsb:havah_active):	Started cluster02   ## cluster02 노드로 Active 전환
     havah_status	(lsb:havah_status):	Started cluster02   ## cluster02 노드로 Active 전환
 Resource Group: Backup
     havah_backup	(lsb:havah_backup):	Stopped (disabled)  ## havah_backup disable

Failed Resource Actions:
* havah_status_monitor_60000 on cluster02 'unknown error' (1): call=73, status=Timed Out, exitreason='',
    last-rc-change='Mon Jun 12 18:28:51 2023', queued=0ms, exec=60000ms
* havah_active_monitor_60000 on cluster02 'not running' (7): call=69, status=complete, exitreason='',
    last-rc-change='Mon Jun 12 18:28:51 2023', queued=0ms, exec=0ms

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

:warning: Pacemaker는 기본적으로 A노드에서 Failed가 발생되어 B노드로 재배치 되었다고 했을 때 A노드가 정상복구된 상태에서 B노드에서 Failed가 발생되면 A노드로 FailOver를 하지 못한다.

그 이유는 FailCount 설정 때문이다. 이를 해결할 수 있는 방법은

- 첫째, 수동으로 FailCount Reset 해준다.
```

[root@validator01 ~]$ pcs resource failcount show <resource-name>

[root@validator01 ~]$ pcs resource failcount reset <resource-name>
```


## Cluster 상태 복구 절차 (Active Resource  cluster02 → cluster01)
:info:  모든 노드가 Online 상태인지 확인.

```
[root@validator01 ~]$ pcs status
...
2 nodes configured
3 resource instances configured (1 DISABLED)

Online: [ cluster01 cluster02 ]  ## cluster02 노드 Online 상태 확인.

Full list of resources:

 Resource Group: Active
     havah_active	(lsb:havah_active):	Started cluster02
     havah_status	(lsb:havah_status):	Started cluster02
 Resource Group: Backup
     havah_backup	(lsb:havah_backup):	Stopped (disabled)
...
```

:info:  havah_backup  Resource 활성 및 상태 확인.
```
[root@validator01 ~]$ pcs resource enable havah_backup

[root@validator01 ~]$ pcs status
...
2 nodes configured
3 resource instances configured

Online: [ cluster01 cluster02 ]

Full list of resources:

 Resource Group: Active
     havah_active	(lsb:havah_active):	Started cluster02
     havah_status	(lsb:havah_status):	Started cluster02
 Resource Group: Backup
     havah_backup	(lsb:havah_backup):	Started cluster01  ## cluster01 노드에서 활성 확인.

...
```

:info:  cluster01 노드 BlockSync 완료 후 Resource  노드 위치 원복. (cluster02 → cluster01)

```
[root@validator01 ~]$ pcs resource move Active cluster01

...
Online: [ cluster01 cluster02 ]

Full list of resources:

 Resource Group: Active
     havah_active	(lsb:havah_active):     Started cluster01
     havah_status	(lsb:havah_status):     Started cluster01
 Resource Group: Backup
     havah_backup	(lsb:havah_backup):     Stopped (disabled)
...
```
:check_mark:  havah_active 기동 시 스크립트 상 havah_bakcup disable 시키는 로직이 있음. (active 기동 간섭 방지용)

 

:info:  havah_backup Resource 다시 활성화
```
[root@validator01 ~]$ pcs resource enable havah_backup

[root@validator01 ~]$ pcs status

...
Online: [ cluster01 cluster02 ]

Full list of resources:

 Resource Group: Active
     havah_active	(lsb:havah_active):	Started cluster01
     havah_status	(lsb:havah_status):	Started cluster01
 Resource Group: Backup
     havah_backup	(lsb:havah_backup):	Started cluster02
...
```

:info:  pcs location Fixed 설정 제거 하기
```
[root@validator01 ~]$ pcs constraint location --full
Location Constraints:
  Resource: Active
    Enabled on: cluster02 (score:INFINITY) (role: Started) (id:cli-prefer-Active)
  Resource: havah_active
    Enabled on: cluster01 (score:INFINITY) (role: Started) (id:cli-prefer-havah_active)
    
[root@validator01 ~]$ pcs constraint location remove cli-prefer-Active   ## 위 명령 결과 값 id 입력
[root@validator01 ~]$ pcs constraint location remove cli-prefer-havah_active  ## 위 명령 결과 값 id 입력
:check_mark: 해당 location 설정이 되어 있다면, Resource 기동 시  설정 되어 있는 노드 우선 적으로 실행 하게 됨.
```
 

 

# Pacemaker 기본 운영 명령어.
 

:info: Create cluster 
```
[root@validator01 ~]$ pcs cluster setup --name cluster cluster01 cluster02 --transport udpu
```

:info: cluster 시작 및 중지

시작
```
[root@validator01 ~]$ pcs cluster start  # Start on One node
[root@validator01 ~]$ pcs cluster start -all  # Start on all cluster members
```

중지
```
[root@validator01 ~]$ pcs cluster stop  # Stop One node
[root@validator01 ~]$ pcs cluster stop -all  # Stop all cluster members
```
:warning:  서버 재기동 시 Offline 으로 되어 있다면 위 start 명령어로 기동 시킴.

 

:info: pacemaker 상태 체크
```
[root@validator01 ~]$ pcs status
```

:info: havah_active Resource 를 활성 및 비활성
```
[root@validator01 ~]$ pcs resource enable havah_active
[root@validator01 ~]$ pcs resource disable havah_active
```

:info: Resource 노드 간 이동
```
[root@validator01 ~]$ pcs resource move havah_active cluster02
[root@validator01 ~]$ pcs resource move Active cluster02
```

:info: havah_active 의 Fail Count 확인 및 Reset

```
[root@validator01 ~]$ pcs resource failcount show
[root@validator01 ~]$ pcs resource failcount reset havah_active
```
:warning:  pcs status 화면에서 Failed Resource Actions 부분에 에러 정보가 확인 됨. 
       Fail Count가 많아 지면 Resource 동작이 Blocked 되기 때문에, 장애 원인 해결 후 Count Reset 을 해줘야 함.

 

:info: pacemaker 구성 초기화
```
[root@validator01 ~]$ pcs cluster destroy
```

:info: pcs 기본 명령 구성정보
```
- pcs cluster: 클러스터 노드 관련 작업
- pcs property: 클러스터 속성 설정
- pcs resource: 리소스 관련 작업
- pcs constraint: 제약 조건 관련 작업
- pcs stonith: STONITH 관련 작업
- pcs status: 클러스터 상태 확인
- pcs config: 클러스터 구성 파일 생성 및 관리
```

 

### 참고 사이트 : Chapter 14. Colocating cluster resources Red Hat Enterprise Linux 8 | Red Hat Customer Portal 
