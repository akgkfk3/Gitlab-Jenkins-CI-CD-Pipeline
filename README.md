## 📝 프로젝트 소개

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/cc852294-1859-4ea8-b107-1d6961cd342e)

> Gitlab 및 Jenkins를 활용한 CI/CD Pipeline을 구축하여 배포 과정을 자동화하는 프로젝트입니다.
>
> 프로젝트 인원 : 1명
> 
> 프로젝트 기간 : 2023.01.05 ~ 2023.01.06

<br/>

## 🛠 사용 기술

<img src="https://img.shields.io/badge/linux-FCC624?style=for-the-badge&logo=linux&logoColor=black"> <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=Docker&logoColor=black"> <img src="https://img.shields.io/badge/Docker Swarm-2496ED?style=for-the-badge&logoColor=black"> <img src="https://img.shields.io/badge/Apache Web Server-D22128?style=for-the-badge&logo=Apache&logoColor=black"> <img src="https://img.shields.io/badge/Gitlab-FC6D26?style=for-the-badge&logo=Gitlab&logoColor=black"> <img src="https://img.shields.io/badge/jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=black">

<br/>

## 🔎 프로젝트 구상도

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/747c11d2-b601-4444-b57e-3c242e152008)


<br/>

## 🌏 환경 구축

<details>
<summary>Gitlab 및 Jenkins 서버 구축</summary>

<br/>

<b>1. Gitlab 설치 (Rocky Linux 8)</b>

- Add Gitlab CE Repository

```
wget https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh
chmod +x script.rpm.sh
os=el dist=8 ./script.rpm.sh
```

- Install Gitlab CE

```
dnf install gitlab-ce -y
```

- Configure Gitlab CE

```
vim /etc/gitlab/gitlab.rb
-------------------------------------------------------
external_url "https://gitlab.shucloud.site"

# Enable the Let's encrypt SSL
letsencrypt['enable'] = true

# This is optional to get SSL related alerts
letsencrypt['contact_emails'] = ['akgkfk9@gmail.com']

# This example renews every 7th day at 12:30
letsencrypt['auto_renew_hour'] = "12"
letsencrypt['auto_renew_minute'] = "30"
letsencrypt['auto_renew_day_of_month'] = "*/7"
-------------------------------------------------------

gitlab-ctl reconfigure
```

- Check initial_root_password

```
cat /etc/gitlab/initial_root_password
```

<b>2. Jenkins 설치 (Rocky Linux 8)</b>

- Install Java

```
dnf update -y
dnf install java-17-openjdk -y
```

- Add Jenkins Repository

```
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
```

- Install Jenkins

```
dnf install jenkins
systemctl start jenkins
systemctl enable jenkins
```

- Check initial_root_password

```
cat /var/lib/jenkins/secrets/initialAdminPassword
```


</details>

<details>
<summary>Docker Swarm을 이용한 웹 서버 구축</summary>

<br/>

<b>1. Docker 설치 (Rocky Linux 8)</b>

- Add Docker Repository

```
sudo dnf check-update
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

- Install Docker

```
sudo dnf install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

<b>2. Docker-Compose 설치</b>

- Install Docker-Compose

```
wget https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-linux-x86_64
chmod +x docker-compose-linux-x86_64
mv docker-compose-linux-x86_64 /usr/bin/docker-compose
```

<b>3. Docker Swarm Cluster 구축</b>

- Common Configuration

```
sudo firewall-cmd --add-port=2377/tcp --permanent
sudo firewall-cmd --add-port=7946/tcp --permanent
sudo firewall-cmd --add-port=7946/udp --permanent
sudo firewall-cmd --add-port=4789/udp --permanent

setenforce 0
```

- Manager Node

```
docker swarm init --advertise-addr 192.168.100.101 --default-addr-pool 10.100.101.0/24
```

- Worker Node

```
docker swarm join \
--token SWMTKN-1-4qxedy87gygenejrw06hlqpuwfm6erulccfj1jhnmsn0kehbnb-2ld4g3zo36bzu8d8ss4115rhq 192.168.100.101:2377
```

<b>4. DockerFile을 이용하여 httpd 이미지 빌드</b>

- build Container Image

```
vim httpd_Dockerfile
-----------------------------------------------------------
FROM httpd:latest
RUN apt-get update -y
RUN apt-get install -y git
WORKDIR /usr/local/apache2/htdocs
RUN rm -rf *
RUN git clone https://gitlab.shucloud.site/root/test.git .
EXPOSE 80
CMD httpd -DFOREGROUND
-----------------------------------------------------------
docker build -t web:0.1 -f httpd_Dockerfile .
```

<b>5. yml 파일을 이용한 Docker Stack 배포</b>

```
cat <<EOF > web.yml
version: "3.11"

services:
  web:
    image: web:0.1
    deploy:
      mode: replicated
      replicas: 10
      placement:
        constraints: [node.role == worker]
      restart_policy:
        condition: on-failure
        max_attempts: 3
    ports:
      - "8888:80"
    networks:
      - web

networks:
  web:
    external:
      name: web
EOF

docker stack deploy -c web.yml MyStack
```

<b>6. 배포 확인</b>

```
[root@manager ~]# docker stack ls
NAME      SERVICES
MyStack   1

[root@manager ~]# docker service ls
ID             NAME          MODE         REPLICAS   IMAGE     PORTS
q9j2ztc5aiqa   MyStack_web   replicated   10/10      web:0.1   *:8888->80/tcp

[root@manager ~]# docker service ps MyStack_web
ID             NAME             IMAGE     NODE      DESIRED STATE   CURRENT STATE           ERROR     PORTS
dx9ybuqicxfa   MyStack_web.1    web:0.1   worker01   Running         Running 2 minutes ago
neddwi2cpkpb   MyStack_web.2    web:0.1   worker01   Running         Running 2 minutes ago
po0qig9dy5xl   MyStack_web.3    web:0.1   worker01   Running         Running 2 minutes ago
scrgi7fooydp   MyStack_web.4    web:0.1   worker02   Running         Running 2 minutes ago
3zsn71hwbqoh   MyStack_web.5    web:0.1   worker02   Running         Running 2 minutes ago
nf9uet9cwo4m   MyStack_web.6    web:0.1   worker02   Running         Running 2 minutes ago
tsa08tuiirz9   MyStack_web.7    web:0.1   worker03   Running         Running 2 minutes ago
fxwozf1a3r6u   MyStack_web.8    web:0.1   worker03   Running         Running 2 minutes ago
eb0rj4xzlzdr   MyStack_web.9    web:0.1   worker03   Running         Running 2 minutes ago
50wmdszwm005   MyStack_web.10   web:0.1   worker03   Running         Running 2 minutes ago
```

</details>

<br/>

## 🔀 진행 순서


<details>
<summary><b>(1) Gitlab Access 토큰 생성</b></summary>

<br/>

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/c9071c74-51ea-4296-9ce0-a1331a37964a)

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/bdd3d220-391a-4f5c-98f1-c39356b3f782)

</details>

<details>
<summary><b>(2) Jenkins에 Gitlab Access Token 및 Repository 등록 </b></summary>

<br/>

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/b7e477e8-3838-4ca9-8b3b-5a8f02e9fc2e)

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/e56a41e4-92a7-4a44-a01c-df95c076aa12)

</details>

<details>
<summary><b>(3) Jenkins Pipeline 생성 </b></summary>

<br/>

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/52c46e32-ab52-4bcf-8ff6-8ded19e41be8)

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/4ba40d5e-b7cc-41de-9980-0da8a2191d87)

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/977fdc56-9f5d-4ab7-9dca-b763c0331776)

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/76d4588b-128a-42e4-9a2b-1262569d3586)

</details>

<details>
<summary><b>(4) Gitlab Webhook 설정 </b></summary>

<br/>

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/69ce704e-5f0f-48a6-82a4-639f4be0f496)

- Secret Token은 Jenkins에서 Pipeline을 만들 때, generate한 Token 값을 입력한다.

</details>

<details>
<summary><b>(5) 테스트</b></summary>

<br/>

- <b>Jenkins Console 확인</b>

![image](https://github.com/akgkfk3/Gitlab-Jenkins-CI-CD-Pipeline/assets/55624470/b33f855d-a68d-42c9-9a3c-3d1449495d5f)



- <b>배포 확인 </b>

```
[root@manager ~]# docker service ps MyStack_web
ID             NAME                 IMAGE     NODE      DESIRED STATE   CURRENT STATE                 ERROR     PORTS
qyiakyf8azd4   MyStack_web.1        web:0.2   worker01   Running         Running 3 minutes ago
dx9ybuqicxfa    \_ MyStack_web.1    web:0.1   worker01   Shutdown        Shutdown 3 minutes ago
xa93v9ekng7w   MyStack_web.2        web:0.2   worker01   Running         Running 2 minutes ago
neddwi2cpkpb    \_ MyStack_web.2    web:0.1   worker01   Shutdown        Shutdown 2 minutes ago
dvi9gfbaplnt   MyStack_web.3        web:0.2   worker01   Running         Running about a minute ago
po0qig9dy5xl    \_ MyStack_web.3    web:0.1   worker01   Shutdown        Shutdown about a minute ago
y15nv3qhmkae   MyStack_web.4        web:0.2   worker02   Running         Running 3 minutes ago
scrgi7fooydp    \_ MyStack_web.4    web:0.1   worker02   Shutdown        Shutdown 3 minutes ago
squoybqt5nzj   MyStack_web.5        web:0.2   worker02   Running         Running 3 minutes ago
3zsn71hwbqoh    \_ MyStack_web.5    web:0.1   worker02   Shutdown        Shutdown 3 minutes ago
7aps56a199hg   MyStack_web.6        web:0.2   worker02   Running         Running 2 minutes ago
nf9uet9cwo4m    \_ MyStack_web.6    web:0.1   worker02   Shutdown        Shutdown 2 minutes ago
ubtxyrcqauus   MyStack_web.7        web:0.2   worker03   Running         Running about a minute ago
tsa08tuiirz9    \_ MyStack_web.7    web:0.1   worker03   Shutdown        Shutdown about a minute ago
eauaa2xl408w   MyStack_web.8        web:0.2   worker03   Running         Running about a minute ago
fxwozf1a3r6u    \_ MyStack_web.8    web:0.1   worker03   Shutdown        Shutdown about a minute ago
gtnbaftutbav   MyStack_web.9        web:0.2   worker03   Running         Running 2 minutes ago
eb0rj4xzlzdr    \_ MyStack_web.9    web:0.1   worker03   Shutdown        Shutdown 2 minutes ago
i7fncj3jc2vr   MyStack_web.10       web:0.2   worker03   Running         Running 2 minutes ago
50wmdszwm005    \_ MyStack_web.10   web:0.1   worker03   Shutdown        Shutdown 2 minutes ago
```

</details>

<br/>

## 📋 느낀 점

- CI/CD Pipeline을 구축하여 배포 과정을 자동화함으로써 개발 생산성을 높일 수 있어 좋았다.

- 하지만 실제 운영 환경을 기준으로 하였을 때에는 어떻게 배포할 것인지 배포 전략 (롤링, 블루그린, 카나리)에 대한 추가적인 학습이 필요할 것 같다!

<br/>

## 📄 참고 문헌

- https://www.atlantic.net/dedicated-server-hosting/how-to-install-gitlab-on-rocky-linux-8/

- https://www.atlantic.net/dedicated-server-hosting/how-to-install-jenkins-on-rocky-linux-8/

- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-rocky-linux-8

- https://github.com/docker/compose/releases











