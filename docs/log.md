
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



