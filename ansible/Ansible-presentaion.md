# Ansible을 활용한 스위치 실습
<br>
서버에 ansible을 설치하고 스위치에서 사용가능하도록 설정합니다. 간단한 playbook 예제를 통해 스위치에 config를 해봅니다.<br>
ansible-galaxy를 활용한 arista EOS 모듈을 사용한 playbook으로 스위치 config 실습을 진행합니다.

### 실습 구성도
![image](https://github.com/NOOJU/intern-project/assets/127095828/d6ea7527-b552-466d-b8b5-5c1c709085e6)


## 1. ansible 서버(관리 노드)에 ansible 설치 및 설정
* Hypervisor: Pnetlab
* OS : Ubuntu server 22.04
* root 권한으로 진행
* 실습 환경 : ansible [core 2.16.5], python version = 3.10.12, jinja version = 3.0.3
<br>

1. 네트워크 설정
   - ansibe과 필요 패키지들을 다운받기 위해 네트워크가 설정되어있어야함
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
   - 버전 확인 : ansible [core 2.16.5], python version = 3.10.12, jinja version = 3.0.3

   
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
   - ansible 노드 설정 파일 수정<br>
     
   ```
   [arista_node]
   10.0.1.111 ansible_connection=network_cli ansible_network_os=eos ansible_user=admin ansible_ssh_pass=admin ansible_become=yes ansible_become_method=enable
   10.0.2.222 ansible_connection=network_cli ansible_network_os=eos ansible_user=admin ansible_ssh_pass=admin ansible_become=yes ansible_become_method=enable
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
   ssh-keygen
   ```
   ```
   ssh-keyscan -t rsa 10.0.1.111 >> ~/.ssh/known_hosts
   ssh-keyscan -t rsa 10.0.2.222 >> ~/.ssh/known_hosts
   ```
   - ansible 서버의 known_hosts 파일에 스위치의 공개키 지문이 남겨져있어야 플레이북 task 실행 가능<br>
   

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
       eos_config:
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

3. 인터페이스 설정해보기
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
       - name: create new user
         arista.eos.eos_config:
           lines:
             - no switchport
             - description playbook test
           parents: ["interface Ethernet3"]
   ```
   - 붙여넣기<br>
   ```
   ansible-playbook name-test.yml
   ```
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/d2a43cd7-8001-49dc-aee1-6f1277101011)

   
   ![image](https://github.com/NOOJU/intern-project/assets/127095828/444dd185-ec26-427a-bc1a-884567adc453)
   - 성공적으로 interface가 설정된 것을 확인


## 4. ansible-galaxy 사용해보

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
       eos_config:
         lines:
           - hostname test
   ```
   - 붙여넣기<br>
   ```
   ansible-playbook name-test.yml
   ```
