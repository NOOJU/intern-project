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
   

3. 카카오 클라우드 콘솔 > 전체 서비스 > Virtual Machine 접속
4. bastion의 Public IP 복사
5. 브라우저 주소창에 {복사한 IP 주소}:8080 입력
6. 이미지 실행 확인

## 3. Container 레지스트리에 이미지 업로드

1. 도커 로그인
   - 접속 중인 Bastion VM 인스턴스에 명령어 입력
   #### **lab4-3-1**
   ```
   docker login ${PROJECT_NAME}.kr-central-2.kcr.dev --username ${ACC_KEY} --password ${SEC_KEY}
   ```

2. 로그인 성공 시 출력되는 `Login Succeeded` 확인
3. 생성한 이미지 태그하기
   #### **lab4-3-3**
   ```
   docker tag ${DOCKER_IMAGE_NAME} ${PROJECT_NAME}.kr-central-2.kcr.dev/kakao-registry/${DOCKER_IMAGE_NAME}:1.0
   ```

4. 이미지 태그 확인
   #### **lab4-3-4**
   
   ```
   docker images
   ```
   - 현재 두 개의 이미지가 정상적으로 출력되는지 확인
   
5. 이미지가 정상적으로 태그되었는지 확인
   - ex) kakao-k8s-cluster.kr-central-2.kcr.dev/kakao-registry/demo-spring-boot  1.0
     
6. 이미지 업로드하기
   #### **lab4-3-6**
   ```
   docker push ${PROJECT_NAME}.kr-central-2.kcr.dev/kakao-registry/${DOCKER_IMAGE_NAME}:1.0
   ```
7. 카카오 클라우드 콘솔 > 전체 서비스 > Container Registry > Repository 접속
8. 생성한 Repository `kakao-registry` 클릭
9. 이미지 업로드 상태 확인


