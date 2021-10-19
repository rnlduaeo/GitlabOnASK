# Gitlab EE로 Alibaba Cloud 환경에서 CI/CD pipeline 구축하기
시중에는 Jenkins, Gitlab, Travis 등 CI/CD 파이프라인을 위한 다양한 툴이 존재합니다. 오늘은 그 중에서도 GitLab EE(상용 버전)을 가지고 EE버전에서만 활용할 수 있는 고급 기능들과 GitLab을 알리바바 클라우드에서 가장 모범적으로 운영할 수 있는 가이드를 설명합니다. 

## Gitlab on Alibaba 권장 아키텍처
![](https://github.com/rnlduaeo/alibaba/blob/master/gitlab_bestpractice_architecture.png?raw=true)
이번 가이드에서는 ECI 기반 Alibaba Cloud Serverless Kubernetes(ASK)과 NAS file storage를 사용합니다. GitLab은 파이프라인에 등록된 CI/CD Job을 실행할 때 별도의 runner를 호스팅하여 사용할 수 있는데, 이를 통해 리소스를 분산시켜 gitlab ecs instance의 비용을 줄일 수 있고 또 고성능의 Job을 무리없이 돌릴 수 있다는 장점을 가져갈 수 있습니다. 그 외에도 위의 아키텍처의 장점을 정리하면 아래와 같습니다.

1) k8s의 Deployment와 PVC를 통한 높은 서비스 가용성 확보
2) Serverless 기반이기 때문에 k8s master/slave 노드를 관리할 필요가 없음
3) Build될 때만 pod가 trigger되어 실행되기 때문에 비용이 절감됨 ([ASK는 pod가 실행되는 시간에만 과금](https://www.alibabacloud.com/help/doc-detail/89142.htm?spm=a2c63.l28256.b99.9.24e73b03K4PfcM)됨)
4) 고성능을 필요로하거나 무거운 작업을 돌릴 때 리소스 확장성이 용이
> Note: Gitlab Cloud (SaaS) 버전을 사용하더라도 runner는 따로 운용 가능합니다. 

## 용어 설명
- CI/CD: Continues Integration/Continues Delivery(Deployment)의 약자로 반복적인 코드 변경 사항을 지속적으로 빌드/테스트/배포하는 프로세스를 의미합니다. 가령 개발자가 저장소로 push할 때마다 생성된 스크립트 세트를 통해 자동으로 빌드하고 테스트하여 오류나 버그의 발생 가능성을 줄일 수 있습니다. 이 과정은 승인을 거쳐 배포까지 자동화하는 과정으로 확장될 수 있습니다.
<img src="https://docs.gitlab.com/ee/ci/introduction/img/gitlab_workflow_example_11_9.png" alt="drawing" width="700" height="400"/>. 
- [ASK](https://www.alibabacloud.com/help/doc-detail/86366.htm?spm=a2c63.l28256.b99.623.112d7d1bwAOgqd): Alibaba Cloud Serverless Kubernetes의 약자로 k8s를 구성하는 master/slave 노드 모두 알리바바에서 관리하는 전체 관리형 k8s 환경입니다. 사용자는 k8s 인프라 구성 요소에 신경을 덜고 애플리케이션 동작에 더 집중할 수 있습니다.  
- [ECI](https://www.alibabacloud.com/help/doc-detail/89129.htm?spm=a2c63.p38356.b99.2.2b672ec73atwGk): container를 실행시킬 수 있는 환경을 serverless 형태로 제공하는데 ASK(Alibaba Serverless Kubernetes)에서는 단일 pod로 동작합니다. 즉, k8s 환경에서는 ECI=pod 라고 생각하시면 됩니다. eci는 pod가 실행되는 시간에만 cpu/memory unit price 기준으로 과금됩니다.
- [Gitlab runner](https://docs.gitlab.com/runner/): GitLab Runner는 GitLab CI/CD와 함께 작동하여 파이프라인에서 작업을 실행하는 애플리케이션입니다.  
- [Gitlab runner executor](https://docs.gitlab.com/runner/executors/): Gitlab Runner는 여러 시나리오에서 빌드를 실행하는 환경을 의미합니다. 빌드가 실행되면 Gitlab runner은 등록된 executor를 실행시켜 작업을 수행합니다. 이번 가이드에서는 [Kubernetes executor](https://docs.gitlab.com/runner/executors/kubernetes.html)를 사용합니다. 모든 프로젝트를 빌드하기 위한 dependency를 docker image에 넣을 수 있기 때문에 깔끔한 build 환경을 구현할 수 있습니다.   
Build가 trigger되면 kubernetes executor는 build, helper, service-[0-9] container가 포함된 싱글pod가 실행되는데, helper가 git clone, download artifact 같은 작업을 수행하고 build container는 실제 빌드 작업 수행하며 service container는 database와 같은 서비스 컨테이너가 필요할 경우 실행됩니다.   
- [Docker-in-Docker(dind)](https://docs.gitlab.com/runner/executors/kubernetes.html#using-dockerdind): 말 그대로 docker 안에 docker를 실행시키는 환경입니다. Kubernetes executor에서 docket image를 build하려면 dind가 필요하고, dind는 컨테이너의 priviledge mode를 필요로 합니다. Priviledge mode로 컨테이너를 실행시키면 Docker 데몬이 호스트 시스템의 기본 커널에 액세스할 수 있는 위험성이 있기 때문에 권장되는 것이 바로 kaniko build 입니다.  
- [Kaniko](https://github.com/GoogleContainerTools/kaniko): kaniko는 docker 데몬에 의존하지 않고 userspace에서 dockerfile내의 명령어를 수행하기 때문에 priviledge mode 없이도 docker image를 build할 수 있어 더 안전 합니다. Gitlab에서도 권장하고 있는 바입니다. [kaniko image build 방법](https://docs.gitlab.com/ee/ci/docker/using_kaniko.html)은 클릭하여 확인해 주세요.  
- [Helmfile](https://github.com/roboll/helmfile): 선언적으로 helm 차트를 실행하여 변경, 버전 관리에 유용하도록 helmfile를 사용하여 gitlab-runner를 배포하겠습니다.   

## Prerequisite
kubernetes, helm 등에 대한 친숙함(?). 
Code Repository: https://github.com/rnlduaeo/GitlabOnASK

## Part 1. Gitlab instance deployment

### 설명
이번 실습에서는 Gitlab EE 버전의 고급 기능들도 함께 살펴볼 것이기 때문에 CE가 아닌 EE 버전을 설치하도록 하겠습니다. 또한 Notification/alerm 등의 메세지를 전송하기 위해 Aliababa Cloud의 [DirectMail](https://www.alibabacloud.com/help/doc-detail/29414.htm?spm=a2c63.l28256.b99.2.7ece6327a7KeQn)을 사용하고, gitlab instance의 주기적인 백업을 위해 [OSS](https://www.alibabacloud.com/help/doc-detail/31817.htm?spm=a2c63.l28256.b99.32.20665139tWd3bc)를 사용합니다. 

### 클라우드 리소스 생성
단계적 절차 가이드 문서는 ["클라우드 리소스 생성 및 설정 가이드"](https://www.alibabacloud.com/help/doc-detail/31817.htm?spm=a2c63.l28256.b99.32.20665139tWd3bc)를 참조하시되, Gitlab 설치는 [Gitlab 공식 사이트의 가이드](https://about.gitlab.com/install/#centos-8)를 참조합니다. 
> Note: "클라우드 리소스 생성 및 설정 가이드"에서는 CE edition: 무료버전을 설치합니다. 그 부분만 건너 뛰고 공식 가이드에 따라 EE를 설치해 주시면 됩니다. [CE와 EE의 기능 차이](https://about.gitlab.com/features/by-paid-tier/)는 클릭하여 확인해 주세요. 

## Part 2. Gitlab runner deployment
Gitlab runner를 ASK위에 설치합니다. runner는 항시 떠 있지만 나머지 pod는 사용자가 정의한 stage안의 job 이 실행될 때마다 실행되었다 사라집니다. 즉, pipeline이 실행될 때만 리소스를 가져다 사용하는 구조이기 때문에 비용을 매우 절약할 수 있습니다. 동적으로 생성되었다 사라지는 Pod들이 공유하여 스토리지 볼륨을 바라볼 수 있도록 NAS 형태의 PV를 사용할 것입니다.   
<img src="https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-14%20at%204.03.51%20PM.png?raw=true" alt="drawing" width="600" height="470"/>

### 주의사항
- nfs 외에도 cloud disk를 사용할 수도 있지만 ECI가 서로 다른 물리적 시스템에 흩어져 있기 때문에 동시 CI 작업이 있는 경우 이전 작업이 실행될 때까지 기다렸다가 그 다음 작업을 시작하기 전에 볼륨을 umount 해야 합니다. 따라서 nas가 여러 pod가 공유하여 읽고/쓰는 시나리오에 더 적합합니다.
- ECI는 권한 문제로 Dynamic PV는 지원하지 않습니다. Static PV만 지원하기 때문에 volume이 필요하면 아래 처럼 수동으로 PV를 생성해 줘야 합니다. 

### ASK 생성
[ASK 생성 가이드](https://www.alibabacloud.com/help/doc-detail/86377.htm?spm=a2c63.l28256.b99.639.44a87d1bHQISmX)를 따라 cluster에 gitlab runner와 runners(kubernetes executor) 구성 설정을 deploy 합니다. 를 생성합니다.

추후 해당 클러스터와의 API 통신을 위해 [Cluster information > Connection information]의 정보를 가지고 kubeconfig file 구성합니다.
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-14%20at%205.40.24%20PM.png?raw=true)

### NAS File Storage 생성
- [NAS file system을 생성](https://www.alibabacloud.com/help/doc-detail/27530.htm?spm=a2c63.p38356.879954.4.387db99fyUhhIn#section-5jo-0kj-jn5)합니다. ASK가 생성된 같은 VPC와 vswitch를 선택합니다. 그래야 추가적인 설정이나 퍼포먼스 문제 없이 데이터를 읽고 쓸수 있습니다. 필요한 만큼 File system을 생성합니다. 이번 가이드에서는 cache 스토리지와 mvn warehouse 스토리지 두개를 생성합니다.   
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-14%20at%204.52.02%20PM.png?raw=true)
- [mount target을 생성](https://www.alibabacloud.com/help/doc-detail/27531.htm?spm=a2c63.p38356.879954.5.387db99fKMirNm#section-6xi-a3u-zkq)합니다. ASK가 위치한 VPC와 Vswitch를 선택하여 생성합니다. 'IP Address of mount target' 값을 복사합니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-14%20at%204.52.29%20PM.png?raw=true)

### NAS PV, PVC 생성
spec.nfs.server에 복사한 address를 붙여넣기 합니다.  .

주의사항:  
- spec.capacity.storage 을 nas console에서 보이는  Total Capacity와 맞춰줘야 합니다. 두개가 다를 경우 mount error가 발생합니다. 현재 기준, nas에서 얼마만큼의 볼륨을 지정하던지 간에 10PiB로 보이니, (100Gib로 지정하더라도 100Gib 이상 사용하면 자동 payg로 전환되기 때문에 total capacity는 최대치로 보입니다.) spec.capacity.storage: 10Pi 로 지정합니다.
- Extream NAS file system일 경우 subdirectory를 /share로 지정해야 합니다. 

nas-pv.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-runner-cache-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Pi
  mountOptions:
  - nolock,noresvport,noacl,hard
  - vers=3
  - rsize=1048576
  - wsize=1048576
  - proto=tcp
  - timeo=600
  - retrans=2
  nfs:
    path: /share
    server: <IP Address of mount target for cache>
```

spec.nfs.server에 복사한 address를 붙여넣기 합니다.  
mvn-pv.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: gitlab-runner-maven-pv
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Pi
  mountOptions:
  - nolock,noresvport,noacl,hard
  - vers=3
  - rsize=1048576
  - wsize=1048576
  - proto=tcp
  - timeo=600
  - retrans=2
  nfs:
    path: /share
    server: <IP Address of mount target for mvn warehouse>
```
nas-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-runner-cache-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Pi
  volumeName: gitlab-runner-cache-pv
```

mvn-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-runner-maven-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Pi
  volumeName: gitlab-runner-maven-pv
```

4개의 object를 모두 생성합니다. 
```
kubectl apply -f nas-pv.yaml
kubectl apply -f mvn-pv.yaml
kubectl apply -f nas-pvc.yaml
kubectl apply -f mvn-pvc.yaml 
```

네트워크 통신에 문제가 없다면 [ASK cluster 이름 클릭 > Volumes > Persistent Volume Claims]에서 다음과 같이 잘 'bound' 된 것을 확인할 수 있습니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-20%20at%2012.04.43%20AM.png?raw=true)

[자세한 nas pv 생성 가이드](https://www.alibabacloud.com/help/doc-detail/189293.htm?spm=a2c63.p38356.879954.12.3c666ac2K8HmTW#task-2022002)는 클릭하여 확인합니다.

### [선택사항] image push to ACR/deploy to ASK 등을 위한 credential 생성
아래의 credential은 gitlab의 CI/CD variables로 정의하거나 k8s의 secret/configmap 으로 관리할 수 있습니다. 두가지 모두 방법을 보여드리기 위해 ACR authentication은 secret으로 생성하고, ACR에 최종적으로 저장된 docker image를 ASK에 deploy하기 위한 ASK 접속 정보는 gitlab의 CI/CD variables로 정의해 보겠습니다. 

- ACR authentication을 secret으로 생성  
  [ACR(Alibaba Container Registry)](https://www.alibabacloud.com/help/doc-detail/257112.htm?spm=a2c63.p38356.b99.4.25734a650wF5ha)에 docker image를 push하기 위하여 authentication을 생성합니다. ECI는 password-free image pull을 지원하지만 push에는 authentication이 필요합니다.  
  [ACR 초기 세팅](https://www.alibabacloud.com/help/doc-detail/198690.htm?spm=a2c63.l28256.b99.12.3bc11e88mJLbd4)은 클릭해서 확인합니다. 

  ```
  kubectl create secret docker-registry registry-auth-secret --docker-server=${xxx} --docker-username=${xxx} --docker-password=${xxx}
  ```

  생성된 secret은 아래 command로 확인하거나 직접 console에 들어가 [Cluster 클릭 > Configuration > Secrets]에서 확인 가능합니다. 
  ```
  kubectl get secret registry-auth-secret --output=yaml
  ```

- ASK kubeconfig는 gitlab CI/CD variables로 생성  
  [자세한 설정 가이드](https://docs.gitlab.com/ee/ci/variables/)는 깃랩 공식 문서를 참조합니다. 
여기서 secret을 쓰게 될 경우 deploy할 새로운 클러스터를 추가할 때마다 helmfile을 update 해줘야 하므로 번거로울 수 있습니다. 따라서 최종적으로 deploy할 kubernetes의 접속 정보를 복사하여 KUBE_CONFIG라고 하는 variable에 저장하여 사용하겠습니다.
  ![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-19%20at%209.52.14%20PM.png?raw=true)

  base64로 kubeconfig 값을 인코딩하여 variable에 저장했습니다.
  ```
  cat ~/.kube/config | base64
  ```

  ![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-19%20at%209.50.19%20PM.png?raw=true) 
  
  
  KUBE_CONFIG라고 하는 variable에 저장합니다. 

### [선택사항] imageCache 설정
특정 VK version 이후로 부터는 imageCache가 default로 설치되어 있습니다. [imageCache를 사용](https://www.alibabacloud.com/help/doc-detail/273116.htm?spm=a2c63.l28256.b99.653.d5bc7d1bJBdaGX)하면 k8s local cache를 사용함으로써 image를 pulling하는 시간을 감소시켜 pod creation을 훨씬 빠르게 할 수 있습니다. 평균적으로 ~분 단위로 소요되는 시간이 ~초 단위로 줄어듭니다.  

```
apiVersion: eci.alibabacloud.com/v1
kind: ImageCache
metadata:
  name: gitlab-runner
spec:
## gitlab runner, CI/CD 등에 사용하는 모든 이미지
  images:
  - gitlab/gitlab-runner-helper:x86_64-latest
  - gitlab/gitlab-runner:alpine-v13.0.0
  - registry.cn-hangzhou.aliyuncs.com/eci/kaniko:1.0
  - registry.cn-hangzhou.aliyuncs.com/eci/kubectl:1.0
  - registry.cn-hangzhou.aliyuncs.com/eci/java-demo:1.0

```

### Gitlab runner deployment
이번 가이드에서는 아래 gitlab 공식 다큐먼트를 참조하여 예시 helmfile를 작성하였습니다. 예시를 참조하여 실제 환경에 맞게 config를 구성하시면 됩니다.    
Reference link:   
[Gitlab Runner install in k8s (helm chart)](https://docs.gitlab.com/runner/install/kubernetes.html).  
[values.yaml](https://gitlab.com/gitlab-org/charts/gitlab-runner/blob/main/values.yaml). 
[kubernetes executor](https://docs.gitlab.com/runner/executors/kubernetes.html). 

[Menu > Admin > Overview > Runners] 에서 runner를 등록하기 위한 URL과 token을 복사하여 해당 부분에 붙여넣기 합니다.   
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-18%20at%204.43.23%20PM.png?raw=true)

helmfile.yaml
```
repositories:
- name: gitlab 
  url: https://charts.gitlab.io/
releases:
- name: gitlab-runner
  namespace: default
  chart: gitlab/gitlab-runner
  version: 0.33.1
  values:
  - image: gitlab/gitlab-runner:alpine-v13.0.0 ## alpine base image로 사용
    gitlabUrl: https://gitlab.haemieee2.xyz ## gitlab admin console에서 복사
    runnerRegistrationToken: "9mGD3soBfos8psSDp-xd" ## gitlab admin console에서 복사
    rbac: ## k8s 리소스에 대한 조작 권한 부여
      create: true
      resources: ["pods", "pods/exec", "secrets", "services"]
      verbs: ["get", "list", "watch", "create", "patch", "delete"]
      clusterWideAccess: false
    metrics:
      enabled: false
    runners:
    ## [[runners.kubernetes.volumes.pvc]] section에서 사전에 생성된 nas shared volume이 마운트 될 수 있도록 지정 
    ## [[runners.kubernetes.volumes.secret]] 를 통해 secret를 runner pod에 마운트 (Alibaba Container Registry에 push하기 위한 authentication 정보), Gitlab CICD variable로도 지정 가능 - 선택사항
    ## 그외 runners에 대한 config.toml 에 세팅할 수 있는 값들은 https://docs.gitlab.com/runner/executors/kubernetes.html#the-available-configtoml-settings 참조
      config: |
        concurrent = 2
        check_interval = 0
        [[runners]]
          [runners.kubernetes]
            namespace = "default"
            pull_policy = "if-not-present"
            cpu_limit = "0.5"
            cpu_request = "0.5"
            memory_limit = "1Gi"
            memory_request = "1Gi"
            helper_cpu_limit = "0.5"
            helper_cpu_request = "0.5"
            helper_memory_limit = "1Gi"
            helper_memory_request  = "1Gi"
            helper_image = "gitlab/gitlab-runner-helper:x86_64-latest"
            [runners.kubernetes.pod_annotations]
              "k8s.aliyun.com/eci-image-cache" = "true"
            [[runners.kubernetes.volumes.pvc]] 
              name = "gitlab-runner-cache-pvc"
              mount_path = "/cache"
              readonly = false
            [[runners.kubernetes.volumes.pvc]]
              name = "gitlab-runner-maven-pvc" 
              mount_path = "/root/.m2"
              readonly = false
            [[runners.kubernetes.volumes.secret]]
              name = "registry-auth-secret"
              mount_path = "/root/.docker"
              read_only = false
              [runners.kubernetes.volumes.secret.items]
                ".dockerconfigjson" = "config.json"
```

아래 명령어는 $HOME/.kube/config 파일을 참조하여 API로 연결된 k8s cluster에 gitlab runner와 runners(kubernetes executor) 구성 설정을 deploy 합니다. value 설정 변경이 필요할 때마다 이 helmfile 하나만 수정 후 다시 apply하면 자동으로 변경사항만 골라내어 적용되기 때문에 간편합니다. 
```
helmfile apply
```

### Demo CI/CD 실행하기
Code repository: https://github.com/rnlduaeo/GitlabOnASK/tree/main/java-demo-oct  
.gitlab-ci.yml 
```
cache:
  paths:
  - /cache
stages:
  - package
  - build
  - deploy
mvn_package_job:
  image: registry.cn-hangzhou.aliyuncs.com/eci/kaniko:1.0
  stage: package
  script:
    - mvn clean package -DskipTests
    - cp -f target/demo.war /cache #nas volume(/cache에 마운트)에 war artifact를 저장합니다.
build_and_publish_docker_image_job:
  image: registry.cn-hangzhou.aliyuncs.com/eci/kaniko:1.0
  stage: build
  script:
    - mkdir target
    - cp /cache/demo.war target/demo.war
    - echo $CI_PIPELINE_ID
    - kaniko -f `pwd`/Dockerfile -c `pwd` --destination=gitlab-registry.ap-northeast-1.cr.aliyuncs.com/project1/ gitlab-repository:$CI_PIPELINE_ID # 전단계(mvn_package_job)에서 저장한 artifact 기반으로 docket image를 build하고(kaniko build를 사용) ACR에 push합니다. 
deploy_k8s_job:
  image: registry.cn-hangzhou.aliyuncs.com/eci/kubectl:1.0
  stage: deploy   
  script:
    - mkdir -p ~/.kube
    - echo $KUBE_CONFIG | base64 -d > ~/.kube/config  # gitlab CI/CD variables에 저장된 KUBE_CONFIG 를 사용하여 target server에 deploy합니다. 
    - sed -i "s/IMAGE_TAG/$CI_PIPELINE_ID/g" deployment.yaml
    - cat deployment.yaml
    - kubectl apply -f deployment.yaml
```

![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-19%20at%2011.30.04%20PM.png?raw=true)
## Troubleshooting
- CI/CD pipeline이 실행되면 새로운 runner pod가 실행되었다 없어지는 것을 확인할 수 있습니다. 
![](https://github.com/rnlduaeo/alibaba/blob/master/Screen%20Shot%202021-10-19%20at%2011.32.48%20PM.png?raw=true)

- runner pod를 describe 해보면 아래와 같이 build와 helper containers로 구성된 pod임을 알 수 있으며 기타 기 정의된 gitlab CI/CD variables와 구성한 volumes 등을 확인할 수 있습니다. 위의 helmfile에서 config.toml로 구성한 runners의 secret과 기타 volume등은 여기서 확인 가능합니다. 
  ```
  $ kubectl describe pod runner-d1uxg4p4-project-9-concurrent-0vfwm2

  Name:         runner-d1uxg4p4-project-9-concurrent-0vfwm2
  Namespace:    default
  Priority:     0
  Node:         virtual-kubelet-ap-northeast-1b/172.29.206.209
  Start Time:   Tue, 19 Oct 2021 23:33:34 +0900
  Labels:       pod=runner-d1uxg4p4-project-9-concurrent-0

  .......

  Containers:
    build:
      Container ID:  containerd://f5059875921e668a5bdf2c7fc4844592d2572bcd0668d2d6c52f5054e2d0836b
      Image:         registry.cn-hangzhou.aliyuncs.com/eci/kaniko:1.0

  ........

    helper:
      Container ID:  containerd://d85794bb358b0c3b525da1786f9fff094279db1944ebd7a19d0f7bbacbe0255a
      Image:         gitlab/gitlab-runner-helper:x86_64-latest

  ......

  Volumes:
    registry-auth-secret:
      Type:        Secret (a volume populated by a Secret)
      SecretName:  registry-auth-secret
      Optional:    false
    gitlab-runner-cache-pvc:
      Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
      ClaimName:  gitlab-runner-cache-pvc
      ReadOnly:   false
    gitlab-runner-maven-pvc:
      Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
      ClaimName:  gitlab-runner-maven-pvc
      ReadOnly:   false
    repo:
      Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
      Medium:
      SizeLimit:  <unset>
    default-token-mncmc:
      Type:        Secret (a volume populated by a Secret)
      SecretName:  default-token-mncmc
      Optional:    false
  QoS Class:       Guaranteed
  Node-Selectors:  <none>
  Tolerations:     node.kubernetes.io/not-ready:NoExecute
                  node.kubernetes.io/unreachable:NoExecute
  Events:
    Type    Reason                   Age   From          Message
    ----    ------                   ----  ----          -------
    Normal  SuccessfulHitImageCache  18s   EciService    [eci.imagecache]Successfully hit image cache imc-6weel91oaxj0az23sz6z, eci will be scheduled with this image cache.
    Normal  Pulled                   7s    kubelet, eci  Container image "registry.cn-hangzhou.aliyuncs.com/eci/kaniko:1.0" already present on machine
    Normal  Created                  7s    kubelet, eci  Created container build
    Normal  Started                  7s    kubelet, eci  Started container build
    Normal  Pulled                   7s    kubelet, eci  Container image "gitlab/gitlab-runner-helper:x86_64-latest" already present on machine
    Normal  Created                  7s    kubelet, eci  Created container helper
    Normal  Started                  7s    kubelet, eci  Started container helper
  ```
- script의 경우 일정 시간 sleep을 걸어두고 직접 pod에 들어가여 (default container가 build 이므로 build container로 접속하게 됩니다.) 스크립트를 작성하거나 문제 발생 시 트러블 슈팅합니다. 
  ```
  kubectl exec --stdin --tty <runner-pod-name> -- /bin/bash
  ```

- 혹여나 runners pod 구동시 pending 상태로 멈춰 있으면 ASK console에 들어가 event를 확인합니다. SDKproviderfailed error 가 뜨면 ticket으로 처리합니다. VK version의 이슈일 수 있는데 백앤드에서 업그레이드하면 문제는 해결됩니다. (현재 시점 2021.10.19 이후에 생성된 ASK의 경우 이 문제가 발생하지 않으나 이전에 생성된 ASK는 문제 발생 여지가 있습니다.)

- gitlab instance 백업 설정에서 ossfs 마운트 시 -oallow_other 옵션을 안주면 다른 user(가령 gitlab user)가 해당 디렉토리에 대한 read/write/config 권한을 받지 못하여 이슈가 발생할 수 있으니 위의 옵션은 반드시 지정합니다.

- gitlab runner image를 gitlab/gitalb-runner:latest 로 지정할 경우 [문제](https://gitlab.com/gitlab-org/charts/gitlab-runner/-/issues/97)가 발생할 수 있으니 alpine 이미지 사용할 것



