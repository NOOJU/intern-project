# Ansible

## Ansible을 써야하는 이유
서버를 운용하다보면 예를들면 패키지 설치, 응용프로그램 설치 등 셋팅이 필요한 경우가 있음
또한 설치가 끝나면 해당 프로그램에 맞게 셋팅도 해줘야 함

apache라면 '/var/www/html/'에 있어야됨.
nginx라면 '/usr/share/nginx/html' 에 있어야됨.

 

### 상황 가정
 ```bash
클러스터를 구축한다고 가정해보겠습니다.
즉, 동일한 기능을 하는 웹 서버를 5대 셋팅한다는 뜻입니다.
5대라면 충분히 서버 하나하나에 접속해서 셋팅할 수 있습니다.
그럼 100대, 200대 라고 생각해봅시다.
또한 위 서버들의 셋팅은 한 그룹은 웹서버, 다른 그룹은 DB, 다른 그룹은 다른 웹서버 등 전부 다 다릅니다.
거의 하루 이틀 날잡아서 될 양이 아닙니다.
특히 요즘 Micromicro service architecture 시대에서 이런 서버양은 가설이 아니라 사실입니다.
 ```
이것들을 자동화 시켜 사용자의 편의성을 제공해주는 소프트웨어가 존재함<br/>
Ansible, Chef, Puppet, SaltStack 이 대표적



## Ansible 소개
Ansible은 Python으로 만들어진 오픈소스 프로젝트이며, 현재는 RedHat에서 인수하여 관리하고 있음<br/>

Ansible은 Playbook이라는 정의된 스크립트나 Ad-Hoc이라는 커맨드 명령을 통해 원격지 노드들을 관리함<br/>
- **Note** ad-hoc 명령이란?<br/>
  사전에 작성된 플레이북을 사용하지 않고, 단일 명령이나 작업을 즉시 실행할 때 사용합니다. 이 방식은 간단한 작업을 빠르게 수행하고 싶을 때 유용합니다.<br/> 예시: ansible all -i 호스트파일 -m command -a "uptime" <br/>
또한 RedHat에서 관리하고 있다는 측면에서 큰 메리트가 있는데, 버그 패치같은 것들을 신경써서 해준다는 점이 있음

 ## Ansible 핵심 특징
Ansible은 SSH를 이용한 원격 통신 및 권한접근임
Chef나 Puppet같은 경우 원격 노드들에게 접근하기 위해서 클라이언트 에이전트를 깔아줘야 하기에, 불편함이 있음<br/>
Ansible은 SSH통신으로 각 노드들에게 접근할 수 있어서 에이전트같은 프로그램 설치가 불필요함

## Ansible 구성 요소
1. 제어 노드(컨트롤 머신)
- Ansible을 설치하고 실행하는 노드
- 원격으로 관리 노드 제어
- 프로젝트 파일의 사본 보관
- Python v2.6 이상 필요

2. 매니지드 노드(관리 노드)
- Ansible을 이용해 관리하고자 하는 서버
- 매니지드 노드에는 Ansible이 설치되지 않음
- 인벤토리에 나열된 대상
- 단독, 그룹으로 관리 가능
- Python v2.4 이상 및 ssh 필요
 
3. Inventory
- Ansible에서는 Inventory라고 하는 파일에 자동화를 위한 타겟을 정의하게 됨
- 또한 이런 호스트들을 그룹으로 묶어서 관리할 수도 있음
- 그외 전역변수를 통해서 변수 설정도 가능
- 해당 Inventory를 가져와서 Ansible이 해당 호스트 정보를 토대로 명령어 실행을 처리

4.  Playbook
- 쉘 스크립트 같은 파일이라고 생각하면 됨
- Inventory에서의 정보를 가져와 원하는 호스트들에게 원격으로 명령을 실행할 수 있도록 정의
- 명령어 실행 순서는 위에서 아래로 진행
- 또한 모듈이라고 하는 명령어들이 정의된 하나의 모듈을 읽어와 실행해주기도 함
- Playbook은 task라고 하는 명령어를 기반으로 사용자가 정의한 명령어가 실행됨
- yml 파일로 저장

6. Module
- Playbook에서 사용할 수 있는 특정 작업을 수행하기 위한 단위로, 파일 복사, 패키지 설치, 네트워크 장치 구성 등의 작업을 수행할 수 있음
- 즉, 일련의 명령어를 실행하는 기능의 집합체(ex: copy, yum, apt 등)

7. 태스크(Task)
- Ansible의 작업 단위

8. role
- 다양한 role과 컬렉션이 있는 public repository는 ansible-galaxy가 있음(기본적으로 ansible을 설치할 때 같이 설치되어서 ansible-galaxy에 있는 다양한 role과 컬렉션을 끌어올 수 있음)

10. facts
- playbook 실행시 노드에 대한 정보를 수집하는 모듈(gather_facts:no 옵션 명시하면 수집안함)
- 이렇게 수집한 정보를 변수화해서 동적인 playbook을 만들 수 있음
- 일반적으로 /etc/ansible/facts.d/ 밑에 파일이름.fact 라는 이름으로 저장됨
-  ansible 호스트명 -m setup을 ad-hoc 방식으로 확인해보면, facts가 수집하는 정보들을 확인할 수 있음
<br/>
 

Ansible의 간략한 구성도<br/>
![image (5)](https://github.com/NOOJU/intern-project/assets/88716899/09fee8fe-f5f4-469d-a1b7-8ac9a60f298e)

<br/><br/>
## IaC로써 ansible과 terraform의 차이점
Ansible과 Terraform 차이점
Terraform은 원하는 상태를 정의하여 IT 인프라의 구성을 유지하려고 하는 선언적 프로그래밍이라는 방식을 사용. Ansible은 원하는 상태에 도달하는 단계를 정의하여 IT 인프라의 구성을 유지하려고 하는 절차적(또는 명령적) 프로그래밍 방식을 사용

주로 Terraform에서 인프라를 구성하고 Ansible에서 인프라 구성 관리를 하는 방식으로 씀
