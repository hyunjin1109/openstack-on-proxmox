
## 2026-10-08
termius에서 ssh 접속을 위해 할당받은 ip 확인하려고 한다.


`ip a`
로 ip 확인해보니 vm ip 정보가 안보였다. 참고로 scope global 이 있는 inet 부분이 외부와 통신할 수 있는 주소이다.

NetworkManager가 관리하는 디바이스 정보
`nmcli device status`
로 네트워크 상태 확인하니 `NetworkManager is not running` 이 떴다. 

생각해보니 openstack 구축하면서 잠깐 networkManager 를 disable, stop 해놨던 기억이 났다.

```bash
systemctl enable NetworkManager
systemctl start NetworkManager
```
로 네트워크 서비스 실행해서 자동 ip 할당되게 했다.

참고로 위 두 명령어를 
`systemctl enable --now NetworkManager`
로 대체 가능하다.

다시 ip 확인해보니 할당이 잘 된 것을 확인할 수 있다.



openstack-glance 설치에서 문제가 있었던 거라

`dnf -y install openstack-glance`

해당 패키지만 다시 설치 시도했다. 실패한 것만 이렇게 따로 설치해보면 설치 실패 이유가 뜬다.

실패한 이유는, 해당 라이브러리를 설치하려면 python3-pyxattr 이 필요한데 이 패키지는 rocky 9의 crb 라는 저장소에 있다.
이 저장소는 꺼져 있는 게 기본이라 켜서 설치해야 하는 것 같다. 
(참고로 openstack-glance → python3-glance → python3-pyxattr 순서로 의존함)

crb 저장소 키는 명령어는 아래와 같다

`dnf config-manager --set-enabled crb`

- dnf config-manager: 저장소 설정을 바꾸는 도구
- --set-enabled crb: crb 저장소를 켬

켜져 있는 저장소 목록 확인 명령어는 아래와 같다
`dnf repolist`
확인해보니 crb가 있어 다시 glance를 설치했다.

<img width="470" height="129" alt="image" src="https://github.com/user-attachments/assets/69d3df0b-0adc-4c86-8be6-88d65cc84853" />

설치 완료!

아래는 패키지 버전 확인 명령어이다.

`rpm -q openstack-glance`


## 2026-10-10
glance 설치 실패로 중단되었던 packstack 다음 과정을 다시 실행 해보려고 한다.
그 전에 해당 rocky9가 메모리 2gb, swap 2gb였어서 설치 도중 문제가 생길 것 같아서 8gb로 변경했다.
아직 proxmox 전체 메모리에서 20%만 사용 중이라 문제 없다고 판단하여 변경했고, 
vm에 반영하려면 재부팅으로는 안되고 terminate 후 재실행해야 한다. 

과거 packstack 설정 파일 만들면서 어떻게 들어간 건지 기억이 안나는데,
현재 vm ip는 201로 끝나는데 answer.txt 파일에 202로 ip가 들어가 있는 것을 알 수 있었다. 

해당 파일은 /etc/systemd/system에 있었는데 홈 디렉토리로 옮긴 후 백업했다. 백업 후에 202로 되어 있는 ip를 모두 201로 변경했다.

그리고 해당 파일이 아래 명령어로 확인해보니 형식이 data로 되어 있었다. (무슨 형식인지 알 수 없다는 뜻)

`file answer.txt`

answer.txt에 NUL 문자 하나가 섞여 있어서 그랬던 것 같다. 생성 시점부터 있었던 것 같은데 해당 부분은 주석 부분이라 설치에는 영향이 없어보이지만 혹시 모를 오작동을 위해 해당 줄 삭제했다. 


answer.txt에 적힌 설정대로 Openstack 자동 설치하라는 명령어를 다시 실행했다. 
```
packstack --answer-file=/root/answer.txt
```

한 30분? 이상 소요된 것 같다. 



또 막혔다. 확인해보니 answer.txt는 모두 201로 변경했는데 db 안의 Nova 셀 정보는 첫 설치 때 저장된 설정이 그대로 되어 있어 문제였다. 


<img width="1903" height="92" alt="image" src="https://github.com/user-attachments/assets/da86dc37-edda-4700-9bfe-ddb0a9e13d04" />



이런 문제가 계속 발생할 것 같아 ip를 하나로 고정을 하긴 해야겠다 싶었고, 서버 주인의 허락을 받고 102로 Ip를 고정 할당했다. answer.txt도 다 다시 102로 변경하고 db 안의 Nova 셀 주소를 다 .102로 수정했다. 
그리고 Packstack을 재실행했다. 

처음 .102를 사용하는 곳이 없는지 확인(해당 명령어는 proxmox console 에서 하는 걸 추천)

`ping -c 2 192.168.0.102`

출력 : 0 received 로 사용 문제 없는 것 확인

아래는 .102 고정 후 ip 확인, ping 까지 한번에 하는 명령어이다.

```
cp /etc/NetworkManager/system-connections/ens18.nmconnection /root/ens18.nmconnection.bak
nmcli connection modify ens18 ipv4.method manual ipv4.addresses 192.168.0.102/24 ipv4.gateway 192.168.0.1 ipv4.dns "168.126.63.1 168.126.63.2"
nmcli connection up ens18
ip -br a
ping -c 2 google.com
```

answer.txt를 .102로 다시 변경하는 명령어
```
sed -i 's/192\.168\.0\.201/192.168.0.102/g' /root/answer.txt
grep -c "192.168.0.201" /root/answer.txt
grep -c "192.168.0.102" /root/answer.txt
```

DB 안의 Nova 셀 주소 고치기
```
mysqldump nova_api cell_mappings > /root/cell_mappings.bak.sql
mysql -e "UPDATE nova_api.cell_mappings SET database_connection=REPLACE(REPLACE(database_connection,'192.168.0.202','192.168.0.102'),'192.168.0.201','192.168.0.102'), transport_url=REPLACE(REPLACE(transport_url,'192.168.0.202','192.168.0.102'),'192.168.0.201','192.168.0.102');"
mysql -e "SELECT name, database_connection, transport_url FROM nova_api.cell_mappings\G" | sed 's/:[^:@]*@/:****@/g'
```




packstack 재실행(새로 tmux 세션 만들어서 진행함)
```
tmux new -s packstack2
packstack --answer-file=/root/answer.txt
```
packstack이 192.168.0.102_controller.pp(설치 설명서)를 서버에 보내고, 서버에서 Puppet이 그 설명대로 작업을 시작한다. packstack은 작업이 끝났는지를 주기적으로 물어보는 과정이다.
201이 남아 있으면 puppet이 고쳐준다. 아래 이미지가 그 작업을 실행 중인 화면이다.

<img width="629" height="49" alt="image" src="https://github.com/user-attachments/assets/07452e93-181f-49ec-9e05-c4ebb204e1bf" />


또 에러가 났다. "Packstack이 ssh-keyscan 출력의 # 안내 줄을 걸러내지 않아 잘못된 SSH 키 데이터가 생성되면서 compute 설치 실패" 했다는 내용이다. 
<img width="844" height="67" alt="image" src="https://github.com/user-attachments/assets/7aa853b3-0d89-4577-acad-c57649e948ef" />

이런 문제가 발생한 원인은 2가지로 보인다. 
1. Packstack이 ssh-keyscan 출력의 안내 줄을 걸러내지 않았다
2. 텍스트 출력을 파싱하는 방식이 OS, 도구 버전 변화에 취약하다
해당 내용을 깊게 파지는 않았다. 

openstack 설치는 성공했다. 
아래 빨간색 글씨는 Neutron 네트워크 엔진으로 OVN을 선택했기 때문에 VPN 기능(VPNaaS)은 쓸 수 없고, 프로젝트 네트워크는 Geneve 방식으로 만들어진다는 내용이다. 해당 내용에 나오는 OVN이나 Geneve, 그리고 각 vm에 vpn 프로그램을 직접 설치하는 방법은 다음에 다루겠다. 

<img width="1201" height="187" alt="image" src="https://github.com/user-attachments/assets/1643e3ed-48d7-415e-ae59-1827fb1779bd" />


IP를 .202 → .201 → .102로 바꾸면서 설치했기 때문에 이전 IP가 남아 있는지 확인했다.
```
grep -rlE "192\.168\.0\.20[12]" /etc 2>/dev/null
mysqldump --all-databases 2>/dev/null | grep -cE "192\.168\.0\.20[12]"
```
- 첫 줄: 설정 파일 중 .201, .202가 들어간 파일 이름 찾기
- 둘째 줄: DB 전체에서 .201, .202가 들어간 줄 개수 (0이면 깨끗함)

Packstack(Puppet)은 새 설정은 추가하지만 이전 설정은 지우지 않아 아래 두 곳에 이전 IP가 남아 있었다.
해당 vm에는 iptables(방화벽)과 swift ring에 남아 있어 제거했고, 자세한 내용은 아래 적었다.
(보안상 관계 없는 설정은 모두 제거하는 게 좋을 것 같다)

**1. 방화벽(iptables)**
이전 IP에서 오는 DB, 메신저 접속을 허용하는 규칙이 남아 있었다.
나중에 그 IP를 다른 기기가 받으면 그 기기에 DB 포트가 열릴 수 있어 삭제했다.

```bash
iptables-save > /root/iptables.bak
iptables -S | grep -E "192\.168\.0\.20[12]" | sed 's/^-A/-D/' | while read -r rule; do eval iptables $rule; done
iptables-save > /etc/sysconfig/iptables
```

**2. Swift 링**
저장 장치 목록에 .201 장치가 남아 있어서 데이터의 절반이 없는 서버로 가게 되어 있었다.

```bash
cd /etc/swift
for r in object account container; do
  swift-ring-builder $r.builder remove 192.168.0.201
  swift-ring-builder $r.builder rebalance
done
```

정리 후 `iptables.save`, `swift/backups/`에만 이전 IP가 남는데 둘 다 자동으로 만들어진 예전 백업본이라 실제 동작에는 영향이 없다.


### Openstack 잘 동작하는지 확인
```
# 관리자 접속 정보 불러오기
source ~/keystonerc_admin
```

```
# Nova 서비스 상태. State가 전부 up이면 정상
openstack compute service list
```

```
# 네트워크(OVN) 에이전트 상태. Alive가 :-) 면 정상
openstack network agent list
```

```
# VM 돌릴 하이퍼바이저. 하나 보이고 State가 up이면 정상
openstack hypervisor list
```
