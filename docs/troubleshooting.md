| # | 문제 | 원인 요약 |
|---|------|----------|
| 1 | dnf 저장소 접속 실패 | CentOS 8 EOL로 저장소 폐쇄 |
| 2 | ISO 업로드 실패 | Proxmox local 저장소 91% |
| 3 | Packstack Glance 설치 실패 | (확인 중) |



## 1. dnf 저장소 접속 실패

**증상**
Rocky 8에 centos-release-openstack-yoga 설치 후, dnf 실행 시 아래 에러 발생함
`Curl error (6): Could not resolve host: mirrorlist.centos.org`

**원인**
- EL8에서 설치 가능한 OpenStack 패키지(RDO)의 최신 버전이 Yoga였음
- Yoga 저장소 패키지가 CentOS 8용 의존 저장소 함께 추가함
- CentOS 8 지원 종료로 해당 저장소 서버가 폐쇄되어 dnf가 메타데이터를 받지 못함

**해결**
Rocky8,9,10 중에 9로 재구축하고 centos-release-openstack-dalmatian 설치함

## 2. ISO 업로드 실패 (Proxmox local 저장소 용량 부족)

**증상**
- Proxmox 웹 UI에서 Ubuntu ISO 업로드 시 공간 부족 에러
- local 저장소 사용량 91% (100.86GB 중 91GB)
- 이후 ISO 없이 만든 VM을 부팅하니 `Boot failed: not a bootable disk` 발생

**원인**
- 웹 업로드는 파일을 호스트의 `/var/tmp`에 먼저 받은 뒤 저장소로 복사함
  → 업로드 중에는 ISO 크기의 약 2배 공간이 필요했음
- `/var/tmp`와 local 저장소가 모두 호스트 OS 디스크에 있어 공간이 부족했음


**확인 과정**
- 호스트 쉘은 일반 사용자 계정이라 확인 못함
- 웹 UI의 local 저장소 Content 탭으로 점유 항목 확인
  - VM 디스크 4개 (각 68GB 이상 표시, 씬 프로비저닝이라 실제 사용량은 더 작음)
  - ISO 이미지 약 12GB

**해결**
- 호스트 관리자가 새 디스크(이름 : sdc)를 저장소로 추가
- 웹 업로드 대신 **Download from URL**로 Rocky 9 minimal ISO를 sdc에 직접 다운로드 -> 웹 업로드 잘 안되면 URL 통해서 받는 경우 흔한 것 같음
  (`/var/tmp`를 거치지 않음)


**알게 된 것**
- Proxmox local 저장소는 호스트 OS와 디스크를 공유하므로 VM 디스크는 별도 저장소에 둘 것
- 화면에 표시되는 VM 디스크 크기는 할당 크기이며 실제 사용량과 다를 수 있음
- 저장소 사용률 80% 이상이면 정리 필요 (100% 도달 시 호스트 서비스 장애 위험)



