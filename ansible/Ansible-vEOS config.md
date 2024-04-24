# Ansible을 활용한 스위치 실습
<br>
서버에 ansible을 설치하고 스위치에서 사용가능하도록 설정합니다. 간단한 playbook 예제를 통해 스위치에 config를 해봅니다.<br>
ansible-galaxy를 활용한 arista EOS 모듈을 사용한 playbook으로 스위치 config 실습을 진행합니다.

### 실습 구성도
![스크린샷 2024-04-11 134531](https://github.com/NOOJU/intern-project/assets/127095828/d7b73c5b-5c0d-4f42-827d-067f56fdade2)


## 1. ansible 서버(관리 노드)에 ansible 설치 및 설정
* Hypervisor: Pnetlab
* OS : Ubuntu 22.04
* root 권한으로 진행
* 실습 환경 : ansible = core 2.16.5, python version = 3.10.12, jinja version = 3.0.3
<br>

1. 네트워크 설정
   - ansible과 필요 패키지들을 다운받기 위해 네트워크가 설정되어있어야함
   ```
   vi /etc/netplan/00-installer-config.yaml
   ```
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/5b9b1525-bf31-4f2c-93d0-7760fdd064c6)<br>
   - ens3 : 인터넷 통신을 하기 위한 인터페이스
   - ens4 : test1 스위치와 연결
   - ens5 : test2 스위치와 연결
   <br>

   ```
   netplan apply
   ```
   - 네트워크 인터페이스 변경 사항 적용
   <br>


3. ansible 설치를 위한 레포지토리 등록
   ```
   apt-add-repository --yes --update ppa:ansible/ansible
   ```
   - ansible 최신 버전을 다운받기 위함
   <br>
   

4. 서버에 ansible 설치
   ```
   apt install ansible -y
   ```

5. ansible 설치 확인
   ```
   ansible --version
   ```
   - 버전 확인 : ansible = core 2.16.5, python version = 3.10.12, jinja version = 3.0.3

   
<br>  

## 2. ansible 서버와 스위치 연동하기

1. 스위치 설정
     #### ID/PW 설정
   - ssh 통신을 위해 PW 설정 필수!
   ```
   username admin privilege 15 secret admin
   ```
   
   #### 인터페이스 IP 설정
   * test1 스위치
   ```
   interface ma1
   ip address 10.0.1.111/24
   ```
   ```
   ping 10.0.1.10
   ```
   - 서버와 통신 확인
     
   <br>
   
   * test2 스위치
   ```
   interface ma1
   ip address 10.0.2.222/24
   ```
   
   ```
   ping 10.0.2.20
   ```
   - 서버와 통신 확인
   - 본 실습에서는 management1 인터페이스를 사용
   
    <br>
   
2. ansible 서버 설정
   ```
   vi /etc/ansible/hosts
   ```
   - ansible 호스트 설정 파일 수정<br>
     
   ```
   [arista_node:vars]
   ansible_connection=network_cli
   ansible_network_os=eos
   ansible_user=admin
   ansible_ssh_pass=admin

   [arista_node]
   10.0.1.111
   10.0.2.222
   ```
   <br>
   
   ```
   ansible arista_node -m ping
   ``` 
   - ansible을 통한 핑테스트<br>
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/ab303f33-a896-4717-ab78-d156af1eee40)
   - 지문 등록을 하지 않아도 핑테스트는 가능!
   - BUT. 플레이북 task는 실행 불가능<br>

   ```
   ssh-keyscan -t rsa 10.0.1.111 >> ~/.ssh/known_hosts
   ssh-keyscan -t rsa 10.0.2.222 >> ~/.ssh/known_hosts
   ```
   - 스위치의 공개키를 받아와 known_hosts 파일에 저장
   - ansible 서버의 known_hosts 파일에 스위치의 공개키 지문이 남겨져있어야 플레이북 task 실행 가능
   

## 3. Playbook으로 config 설정해보기

1. 호스트 네임 변경해보기
   ```
   vi /etc/ansible/name-test.yml
   ```
   - playbook 생성
   ```
   ---
   - name: Config to Arista Switch
   hosts: arista_node
   gather_facts: no
   tasks:
     - name: Set hostname on the switch
       arista.eos.eos_config:
         lines:
           - hostname test
   ```
   - 붙여넣기<br>
   ```
   ansible-playbook name-test.yml
   ```
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/1f29be69-06b8-4531-ae82-4f8392ba2449)

   
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/d36e313a-7948-4479-8384-08645cc67501)
   - 성공적으로 hostname이 바뀐 것을 확인

2. 인터페이스 설정해보기
   ```
   vi /etc/ansible/interface-test.yml
   ```
   - playbook 생성
   ```
   ---
   - name: Update Arista EOS Device Configuration
     hosts: arista_node
     gather_facts: no
     tasks:
       - name: Change interface configuration
         arista.eos.eos_config:
           lines:
             - no switchport
             - description playbook test
           parents: ["interface Ethernet3"]
   ```
   - 붙여넣기<br>
   ```
   ansible-playbook interface-test.yml
   ```
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/d2a43cd7-8001-49dc-aee1-6f1277101011)

   
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/444dd185-ec26-427a-bc1a-884567adc453)
   - 성공적으로 interface가 설정된 것을 확인

   <br>

   * 예제 playbook
   
   ```
   ---
   - name: Deploy SSH key to Arista Switch and set hostname
     hosts: arista_node
     gather_facts: no
     tasks:
       - name: Set SSH key for user admin   # 매니지드 서버에 대해서 공개키 배포
         eos_config:
           lines:
             - username admin ssh-key ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjrVojkISZIG7LIf7USSkvMGReZtc2yJO3yIcHY0rlGIQTAsl/VCdi4DknnPelgxDrO85wSing6Ws6Dn7cpzukhpQ5iv+Ydo+13odvsxpJKJczSf1rb1RdDvZKEddok3PjgK3fZ9Ed/mywc9O5kEaT19R8V1deJgncBiGr6G93qXj7AF9SUKQm1gMEP50g3Hl/omRVHQsrintcQgBSTulxdvk1zP1MmjktBHR2nsVLqgLkiGInOWzXiEl7Y5HsX+8JM+kLsxnRScebdDzB6BSEFdtgaVte56ySN9nH1VMMy1vWDltXpGamNagk1y5MtC6LQMFY/xciJchTUYKJK1jNeHBhklPXDfEsMYH3dwYs/4fY4yq+HZdDDGMY1bVAweTxcJFUbwFH3xK5o72muE0M8Trk0hklhGbq7X7NP5t0jyDTFA9CheiTa3b40B/Ctf+Uxmg2hkmYTeWjYN1r+OA04B0289K1YXvHqY9yJ0p9S1wasnMMTp3z8r+zXABiDf0= root@ubuntu22-server
   
       - name: Set hostname on the first switch  # 1번 스위치에 대해서 호스트 이름 변경
         eos_config:
           lines:
             - hostname ahnahn1
         when: inventory_hostname == '172.16.0.100'
   
       - name: Set hostname on the second switch  # 2번 스위치에 대해서 호스트 이름 변경
         eos_config:
           lines:
             - hostname ahnahn2
         when: inventory_hostname == '10.0.0.100'
   
       - name: Configure VLANs  # VLAN 10을 'Marketing'으로 구성
         eos_config:
           lines:
             - name Marketing
           parents: ["vlan 10"]
         register: result
       - debug:
           var: result
   
       - name: Configure VLANs  # VLAN 20을 'Engineering'으로 구성
         eos_config:
           lines:
             - name Engineering
           parents: ["vlan 20"]
   
       - name: Configure VLANs  # VLAN 30을 'HR'으로 구성
         eos_config:
           lines:
             - name HR
           parents: ["vlan 30"]
   
       - name: Assign IP addresses to interfaces  # Ethernet2인터페이스에 10.10.10.1/24 주소 할당.인터페이스Layer 3 모드로 전환
         eos_config:
           lines:
             - no switchport
             - ip address 10.10.10.1/24
           parents: ["interface Ethernet2"]
   
       - name: Assign IP addresses to interfaces # Ethernet3인터페이스에 10.20.20.1/24 주소 할당.인터페이스Layer 3 모드로 전환
         eos_config:
           lines:
             - no switchport
             - ip address 10.20.20.1/24
           parents: ["interface Ethernet3"]
   
       - name: Assign IP addresses to interfaces # Ethernet4인터페이스에 10.30.30.1/24 주소 할당.인터페이스Layer 3 모드로 전환
         eos_config:
           lines:
             - no switchport
             - ip address 10.30.30.1/24
           parents: ["interface Ethernet4"]
   
       - name: Configure static routes  # 기본 게이트웨이로의 정적 라우트와 192.168.100.0/24 네트워크로의 추가 정적 라우트를 구성
         eos_config:
           lines:
             - ip route 0.0.0.0/0 10.0.0.1
   
       - name: Configure static routes
         eos_config:
           lines:
             - ip route 192.168.100.0/24 10.10.10.2
   # BGP를 활성화하고 AS 번호를 65001로 설정한 다음 이웃 10.0.0.2와의 연결을 구성하고 BGP를 통해 두 개의 네트워크를 알림
       - name: Configure BGP 
         eos_config:
           lines:
             - router bgp 65001
             - neighbor 10.0.0.2 remote-as 65002
             - network 10.10.10.0 mask 255.255.255.0
             - network 10.20.20.0 mask 255.255.255.0
           parents: ["router bgp 65001"]
   ```




## 4. ansible-galaxy 사용해보기

### ansible-galaxy Arista EOS 컬렉션 주소 : https://galaxy.ansible.com/ui/repo/published/arista/eos/docs/
#### 해당 모듈을 클릭 후, 사용 문법 및 예제 참고하여 사용

1. ansible 서버의 공개키 스위치에 배포해보기
   ```
   ssh-keygen
   ```
   - 키 생성하기<br>
   
   ```
   vi /etc/ansible/key-test.yml
   ```
   - playbook 생성
   ```
   ---
   - name: Update Arista EOS Device Configuration
     hosts: arista_node
     gather_facts: no
     tasks:
       - name: create new user
         arista.eos.eos_user:
           name: admin
           sshkey: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
           state: present
   ```
   - 붙여넣기<br>
   ```
   ansible-playbook key-test.yml
   ```
   ```
   ssh admin@10.0.1.111
   ```
   ```
   ssh admin@10.0.2.222
   ```
   - 패스워드 없이 접속되는 것을 확인

<br>

2. command 모듈 사용하여 현재 config 출력해보기
   
   ```
   vi /etc/ansible/command-test.yml
   ```
   - playbook 생성
   ```
   - name: Collect routing table from Arista EOS device
     hosts: arista_node
     gather_facts: no
     tasks:
       - name: Gather routing information
         arista.eos.eos_command:
           commands:
             - show running-config
         register: running_info
   
       - name: Print running information
         debug:
           var: running_info.stdout_lines
   ```
   - 붙여넣기<br>
   ```
   ansible-playbook command-test.yml
   ```
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/b02a30b1-c2a1-43b0-a3ee-5bb68ab34ff9)

   - running-config가 출력되는 것을 확인

<br/><br/>


3. VRF, Vlan, interface 생성 및 설정
- eos_vrf, eos_vlan등의 모듈을 사용하여 생성하면 속도가 느림
  → eos_config 모듈이 속도가 가장 빠름

```
---
- name: Configure VRFs and VLANs on Arista EOS devices
  hosts: arista_node
  gather_facts: no
  vars:
    # VRF variables
    vrf_base_name: "VRF"
    vrf_base_id: 1
    vrf_count: 50
    rd_base: "65000"
    vrfs: "{{ range(vrf_base_id, vrf_base_id + vrf_count) | map('regex_replace', '^', vrf_base_name) | list }}"
    rds: "{{ range(vrf_base_id, vrf_base_id + vrf_count) | map('string') | map('regex_replace', '^', rd_base + ':') | list }}"

    # VLAN variables
    vlan_base_id: 1
    vlan_count: 50
    vlans: "{{ range(vlan_base_id, vlan_base_id + vlan_count) | list }}"
    vlan_descriptions: "{{ range(vlan_base_id, vlan_base_id + vlan_count) | map('regex_replace', '^', 'VLAN_') | list }}"

    # IP variables for interface configuration
    ip_base_third_octet: 1  # Assuming all interfaces will use 1.1.x.1
    ips: "{{ range(vlan_base_id, vlan_base_id + vlan_count) | map('regex_replace', '^', '1.1.') | map('regex_replace', '$', '.1/24') | list }}"

  tasks:
    - name: Configure VRFs 
      arista.eos.eos_config:
        lines:
          - "vrf instance {{ item.0 }}"
          - "rd {{ item.1 }}"
          - "description Description for {{ item.0 }}"
        parents: "vrf instance {{ item.0 }}"
        match: exact
      loop: "{{ vrfs | zip(rds) | list }}"

    - name: Configure VLANs 
      arista.eos.eos_config:
        lines:
          - "vlan {{ item.0 }}"
          - "name {{ item.1 }}"
        parents: ["vlan {{ item.0 }}"]
        match: exact
      loop: "{{ vlans | zip(vlan_descriptions) | list }}"

    - name: Configure interfaces with VLANs and assign to VRFs
      arista.eos.eos_config:
        lines:
          - "interface vlan {{ item.0 }}"
          - "vrf {{ item.1 }}"
          - "ip address {{ item.2 }}"
        parents: ["interface vlan {{ item.0 }}"]
        match: exact
      loop: "{{ vlans | zip(vrfs, ips) | list }}"
```

<br/><br/>


4. 생성한 리소스 및 설정 삭제
```
---
- name: Remove VRFs and VLANs on Arista EOS devices
  hosts: arista_node
  gather_facts: no
  vars:
    # VRF variables
    vrf_base_name: "VRF"
    vrf_base_id: 1
    vrf_count: 50
    vrfs: "{{ range(vrf_base_id, vrf_base_id + vrf_count) | map('regex_replace', '^', vrf_base_name) | list }}"

    # VLAN variables
    vlan_base_id: 1
    vlan_count: 50
    vlans: "{{ range(vlan_base_id, vlan_base_id + vlan_count) | list }}"

  tasks:
    - name: Remove VRFs 
      arista.eos.eos_config:
        lines:
          - "no vrf instance {{ item }}"
        match: exact
      loop: "{{ vrfs | list }}"

    - name: Remove VLANs
      arista.eos.eos_config:
        lines:
          - "no vlan {{ item }}"
        match: exact
      loop: "{{ vlans | list }}"

    - name: Remove interfaces
      arista.eos.eos_config:
        lines:
          - "no interface vlan{{ item }}"
        match: exact
      loop: "{{ vlans | list }}"
```
   
   
