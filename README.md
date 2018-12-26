Kubernetes 클라우드 소스 코드 인수 인계
===
ICNS 연구실에서 개발했던 쿠버네티스 개발 소스코드 인수인계에 대한 README 입니다.


## 1. API 서버, 데이터베이스 컨테이너 생성
쿠버네티스가 제대로 설치되었다는 가정 하에, 아래의 명령어를 실행해 라벨링을 수행합니다. 마스터 노드 중 하나를 선택해 `primary-master=true` 라벨을 지정합니다.

    # root@icns-47:~# kubectl label node icns-47 primary-master=true 

    # root@icns-47:~# kubectl get no --show-labels | grep primary
      icns-47    Ready     master    ... kubernetes.io/master=true,primary-master=true

아래의 명령어를 입력해 제공된 두 개의 컨테이너(Deployment) 를 생성합니다. 이는 각각 API 서버와 MySQL에 해당합니다.

    # root@icns-47:/home/alicek106/cluster-dev# kubectl apply -f dev-deployment.yaml
    # root@icns-47:/home/alicek106/cluster-dev# kubectl apply -f mysql-deployment.yaml

컨테이너가 잘 생성되었는지 확인합니다. 참고로, 컨테이너 (Deployment) 가 생성되는 동시에 해당 Deployment를 위한 Service가 동시에 생성됩니다.

    # root@icns-47:/home/alicek106/cluster-dev# kubectl get deployment
      NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
      kubernetes-dev   1         1         1            1           1h
      mysql            1         1         1            1           1h

그렇지만 `kubernetes-dev` 컨테이너는 remote Python Interpreter를 제공할 뿐 쿠버네티스 제어 관련 소스코드가 내부에 들어있는 것은 아닙니다. 소스코드의 업로드는 Pycharm을 통해 진행합니다. 

아래의 명령어를 입력해 default 계정에 clusterrole 권한을 부여합니다. API 서버는 default 계정을 통해 쿠버네티스 클러스터를 제어하게 됩니다.

    # kubectl create clusterrolebinding dashboard-admin-alicek106 \
    -n default --clusterrole=cluster-admin --serviceaccount=default:default

## 2. 프로젝트 세팅
세팅해야 할 프로젝트는 총 두 개입니다. 하나는 API 서버이고, 다른 하나는 API 서버에 사용자의 요청을 전달하는 웹 서버입니다. 첨부된 프로젝트인 `kubernetes-dev` 와 `kubernetes-web` 프로젝트를 Pycharm에서 Open 합니다.
### 2.1 kubernetes-dev
`kubernetes-dev` 프로젝트는 Deployment, Service 등의 생성, 삭제 등의 기능을 수행하며, 쿠버네티스를 실제로 제어하는 핵심 모듈입니다. 쿠버네티스 마스터 노드에서 컨테이너 (Deployment) 로서 실행되고, 컨테이너 내부에 마운트되는 `default ` serviceaccount의 secrets을 사용해 쿠버네티스에 접근합니다.

PyCharm의 File - Setting - Project Interpreter - .. 를 선택해 원격 인터프리터 SSH 연결을 선택합니다. 자세한 연결 방법은 아래의 링크를 참고합니다.

 > https://blog.naver.com/alice_k106/221337670333

기본적인 접근 정보는 아래와 같습니다. 30000번 포트는 위에서 생성한 API 서버 컨테이너의 22번 포트와 연결되어 있음에 유의합니다.

| User   |      Password   |  Port |Interpreter 위치 |
|----------|:-------------:|:------:|:------:|
| root |  theoryofhappiness | 30000 | /root/python_env.sh |


쿠버네티스 마스터 노드에서 아래의 명령어를 입력해 MySQL 컨테이너의 IP를 얻습니다.
    # root@icns-47:~# kubectl describe po $(kubectl get po | grep mysql | awk '{print $1}') | grep IP | awk '{print $2}'
      10.233.75.4

출력된 IP를 `config.json`의 `host` 부분에 바꿔 넣습니다. 아래는 변경된 예시를 나타냅니다. (또는 IP 대신 mysql 을 입력해도 됩니다. 쿠버네티스 자체 DNS 덕분에 Deployment의 이름을 자동으로 인식해 Discovery가 가능하기 때문입니다. 예전에 뭔가 이유가 있어서 Alias가 안먹혀서 IP를 직접 썼었었는데, 지금 해보니 mysql 써도 됩니다)

<pre><code>{
    "mysql":{
        "host":"10.233.75.4",
        "user":"root",
        "password":"...",
        "db":"cloud"
    }
}
</code></pre>

위의 설정을 마친 뒤에는 main.py 파일을 실행해 API를 실행할 수 있습니다. 아래와 같이 출력이 되었다면 정상적으로 실행된 것입니다.

![이미지](https://i.imgur.com/a4bZr38.png)



### 2.2 kubernetes-web
프로젝트를 import한 뒤 해당 프로젝트에 대한 virtualenv를 새롭게 설정해야 합니다. 이 작업은 여기서는 별도로 설명하지는 않지만, 아래의 방법을 통해 설정할 수 있습니다.

 > File - Settings - Project Interpreter - 오른쪽의 톱니바퀴 모양 - Add Local - New Environment

새롭게 로컬 인터프리터를 생성하는 것은 시간이 조금 걸릴 수 있습니다. 생성 뒤에는 필요한 패키지를 설치합니다. 패키지의 목록은 아래와 같지만, 필요하다면 추가로 더 설치할 수 있습니다.

    1. flask-RESTful
    2. pytz
    3. python-dateutil
    4. requests

다음으로, `config.json` 에서 API 서버의 Endpoint를 설정합니다. 본 문서를 따라했다면 마스터 노드의 30001 포트로 API 서버에 접근할 수 있습니다.

     {
        "api-server": "http://163.180.117.XX:30001",
        ....


그 뒤, kubernetes-web.py를 실행하면 웹 대시보드에 접속할 수 있습니다. 로컬의 인터프리터를 사용하기 때문에 localhost:5000/v1/ui/login 으로 로그인 페이지에 접근이 가능합니다.

![이미지](https://i.imgur.com/GYu38Te.png)


### 2.3 Custom Configuration
본 문서를 작성할 시점을 기준으로, icns-47, icns-123, icns-128 서버에 쿠버네티스 dev cluster가 구성되어 있습니다. 추후 Live 서버를 구성하려면 서버 관리자와 상의하세요.

현재는 icns-128 서버에만 PerCV Lab의 Object Removal 이미지가 존재하기 때문에, 해당 패키지 생성은 icns-128에서만 수행되도록 소스코드에 설정되어 있습니다.

![이미지](https://i.imgur.com/tvCfAfe.png)

위의 소스코드는 `kubernetes-dev` 프로젝트의 `controller.DplController` 클래스에 작성되어 있습니다.
