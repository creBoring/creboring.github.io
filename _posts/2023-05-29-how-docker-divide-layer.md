---
layout: post
title: "[Docker] Docker가 Image Layer를 구성하는 방법"
summary: "Docker는 무엇을 기준으로 이미지 레이어를 나눌까?"
author: creboring
date: '2023-05-29 02:13:21 +0530'
category: [Docker]
thumbnail: /assets/img/posts/category/Docker-family.jpeg
title-img: /assets/img/posts/category/blacksmith.jpg
keywords: Docker, container, Image, Layer, 도커, 이미지, 레이어
permalink: /blog/how-docker-divide-image-layer/
usemathjax: true
---


<figure>
    <img src="/assets/img/posts/category/Docker-family.jpeg" class="img-fluid">
</figure>

최근 AWS 계정의 리소스를 이전하는 과정에서 이미지를 직접 pull & push 하게 되었는데, 개발계 이미지를 먼저 pull 한 상태로 운영계의 이미지를 가져오니 **대부분의 레이어는 already exists로 캐시를 사용하고, 가장 마지막 단계만 별도로 download 되었습니다**. 도커가 빌드 효율과 시간 단축을 위해 레이어를 만들어 저장하고 있다는 사실은 알고있었지만, **어떠한 기준으로 레이어를 나누게 되는 것인지** 의문이 들어 원리를 분석해보았습니다. 

## 결론
도커는 `Dockerfile`을 읽어들여, **파일 시스템에 변화를 주는 커맨드마다 새로운 이미지 레이어를 만듭니다**. 즉, Dockerfile을 읽어들여 각 줄마다 이미지 레이어를 만든다는 말은 틀린 말은 아닙니다. 하지만, 모든 줄마다 레이어를 만드는 것이 아닌 파일 시스템에 변화가 발생하는 경우만 이미지 레이어를 생성합니다. 

또한, 도커 빌드 엔진은 이미지 레이어의 공간 효율과 안정성을 위해 꾸준히 빌드 방식을 변경하고 개선해가고 있습니다. 따라서, **도커 엔진 버전, 빌드 라이브러리의 종류에 따라 결과물은 조금씩 차이가 있을 수 있습니다**. (Docker 엔진 **23 버전**부터는 `buildx` 를 기본 builder로 사용합니다)<br>
도커 공식문서에서는 이미지 레이어를 만들 수 있는 상황들에 대해 아래와 같이 설명했습니다.
> Each layer is only a set of differences from the layer before it. Note that **both adding, and removing files will result in a new layer**.<br>
(https://docs.docker.com/storage/storagedriver/)

파일 시스템에 변화를 주지 않는 무의미한 커맨드의 반복이나, echo와 같이 stdout을 발생시키는 커맨드, `LABEL` 과 같은 메타 데이터를 수정하는 명령들은 새로운 이미지를 만들지 않습니다. 메타 데이터의 경우 이미지의 별도 메타 데이터 저장 공간에 JSON 형식으로 저장됩니다.


## 원리

해당 원리를 정리하기까지 많은 시행착오를 겪었습니다. 동일한 Dockerfile로 이미지를 만들어도 엔진의 버전에 따라 결과물이 달라지고 예상을 벗어나는 커맨드들이 있었습니다.. (해당 부분은 아래에서 상세히 다루겠습니다.)

위의 Docker Document에서도 명시되었듯이, **이미지 레이어**는 **‘파일시스템의 변경사항을 캡처하는 단위’** 입니다. 아래에는 이미지 레이어와 그 원리에 대해 분석 및 정리한 내용을 다룹니다.

### 도커 이미지 레이어란?

<figure>
    <img src="/assets/img/posts/2023-05-29/container-filesystem.jpg" class="img-fluid" width="60%">
    <figcaption><small>컨테이너 파일시스템의 레이어 구조</small></figcaption>
</figure>

도커는 이미지를 만들 때 하나의 단일 스냅샷으로 만드는 것이 아닌, 여러 개의 계층(layer)을 가진 다수의 스냅샷으로 나누어 저장합니다. 이는 여러 이미지 파일들을 관리할 때, 이미지들에서 **중복되는 영역을 하나의 레이어를 통해 관리**하여 공간 & 시간 효율을 얻기 위함입니다. 

이는 같은 데이터를 기반으로 하는 다중 이미지에서 중복되는 영역을 하나의 스냅샷으로 관리할 수 있음을 뜻합니다. 마찬가지로 해당 레이어를 각 이미지마다 다운로드 받을 필요도 없어지기 때문에 **이미지의 다운로드 시간도 크게 절약됩니다**. (이는 특히, 이미지의 버전을 관리할 때, 신규 버전을 빌드하는 과정에서 기존 레이어를 그대로 활용할 수 있어 공간, 시간적으로 큰 효율을 가져다 줍니다.)

도커의 레이어 방식은 이미지뿐만 아니라 컨테이너가 실행될 때도 사용됩니다. 이미지를 이용해 새로운 컨테이너를 실행하면, 이미지 레이어 스냅샷들이 복원되고 신규 컨테이너 공간에 `mount` 됩니다. 그리고 그 이후 해당 mount 위에 새로운 레이어가 구성되는데, 해당 컨테이너 환경 내에서 발생하는 모든 변경사항이 이 새로운 레이어에 저장됩니다. 이를 **컨테이너 레이어**라 부릅니다.

이는 해당 **컨테이너에서의 변경사항이 기존 이미지 레이어에 영향을 가하지 않고 해당 컨테이너의 격리된 공간 내에서만 행해짐**을 뜻합니다.

기존 이미지 레이어들이 Read Only로 읽기만 가능한 것과 반대로, 새로 만들어진 컨테이너 레이어는 Read와 Write가 모두 가능합니다. 대신, 별도의 namespace 공간에서 새로운 레이어를 띄워 작업하기 때문에, 해당 컨테이너가 종료되면 변경사항 또한 모두 함께 삭제됩니다.

### 도커 이미지 분산의 핵심 'UnionFS'
위와 같이 파일 시스템을 여러겹의 레이어로 만들고 복원해 사용하기 위해서는, 해당 레이어들을 하나의 파일 시스템처럼 동작하게 하는 시스템이 필요합니다. 이를 가능하게 하는 것이 도커 이미지 레이어에서 핵심 역할을 하는 `Union File System(UnionFS)` 입니다. (~~Unix File System 아님~~)

UnionFS는 **두 개 이상의 디렉토리를 하나의 디렉토리인 것처럼 합쳐서 보여주는 시스템**입니다. 아래 IBM 공식문서에 기입된 예시를 통해 살펴보면, 두 디렉토리간에는 '상위' 와 '하위' 관계가 있고 두 디렉토리의 내용물이 하나의 디렉토리처럼 합쳐져 보여주게 됩니다. 이 때, 두 디렉토리에서 겹치는 정보가 있다면 상위 디렉토리의 데이터를 사용합니다.

<figure>
    <img src="/assets/img/posts/2023-05-29/ufstree.jpg" class="img-fluid" width="60%">
    <figcaption><small>IBM 문서: https://www.ibm.com/docs/en/zos/2.5.0?topic=planning-managing-union-file-system-ufs</small></figcaption>
</figure>

해당 UnionFS는 Linux에서도 지원하기 때문에 아래와 같은 커맨드로 쉽게 테스트해볼 수 있습니다. 기존에 파일 시스템을 mount 하는 구문이랑 크게 다르지 않습니다.

```
mount -t ufs -o upperdir=dir1,lowerdir=dir2,workdir=wrk -f myufs dirM
```

도커는 이 UnionFS 를 이용해 이미지 레이어를 구성합니다. 여러겹의 레이어를 아래에서부터 하나씩 Union 하여 최종적으로 사용자에게는 모든 레이어가 합쳐진 파일 시스템이 제공됩니다. 

이를 구현하기 위한 다양한 스토리지 드라이버의 호환성을 제공하고 있는데, 기본적으로 사용하게 되는 툴인 `OverlayFS` 와 `vfs`,`devicemapper` 등을 지원합니다. (아직 저도 OverlayFS 이외에는 사용해보지 못했습니다.)

> https://docs.docker.com/storage/storagedriver/select-storage-driver/


### 도커가 이미지 레이어를 만드는 단위
본론으로 돌아와서, Docker가 어떠한 경우에 새로운 레이어를 만들고, 어떠한 데이터들이 하나의 레이어에 합쳐져서 표현되는지 분석해보았습니다. 먼저 **Docker Document**에서 이미지 레이어와 관련된 내용들을 모아보면 아래와 같은 내용들이 확인됩니다.

> Commands that modify the filesystem create a layer. The `FROM` statement starts out by creating a layer from the ubuntu:18.04 image. The `LABEL` command only modifies the image’s metadata, and **does not produce a new layer**. The `COPY` command adds some files from your Docker client’s current directory. The first `RUN` command builds your application using the make command, and **writes the result to a new layer**. The second `RUN` command removes a cache directory, and **writes the result to a new layer**. Finally, the `CMD` instruction specifies what command to run within the container, which only modifies the image’s metadata, which **does not produce an image layer**.<br>
**Each layer is only a set of differences from the layer before it**. Note that both adding, and removing files will result in a new layer. <br><br>
(https://docs.docker.com/storage/storagedriver/)

위 문장에서 언급된 내용을 토대로 정리하면, **Docker Layer는 이전 레이어와의 차이점을 저장하는 하나의 집합**입니다. 해당 차이란 파일 시스템에서 데이터가 생성되거나 삭제, 수정되는 것을 의미하며, 대표적인 커맨드들에 대한 예시는 아래와 같습니다.

- `FROM`: FROM은 기존 이미지를 참조하는 문장으로, 새로운 레이어를 만들진 않지만 해당 이미지의 레이어들을 가져옵니다.
- `LABEL`: LABEL은 이미지의 메타데이터만 수정하므로 새로운 레이어를 만들지 않습니다.
- `COPY`: (이미지를 만드는지에 대한 명확한 언급이 없습니다.)
- `RUN`: make로 어플리케이션을 빌드한 결과물은 새로운 레이어를 만듭니다.
- `CMD`: 컨테이너가 무슨 커맨드를 실행할 것인지에 대한 메타 데이터만 수정하므로 새로운 레이어를 만들지 않습니다.

하지만, 위 공식문서 내용만으로는 실제로 어떻게 동작하는지 명확치 않은 부분들이 존재합니다. 그래서 아래 실제 테스트를 통해 Dockerfile을 빌드했을 때 어떻게 동작하는지 테스트해보았습니다. 

#### 실제 테스트
테스트는 **도커 엔진 20** 버전, 스토리지 드라이버로 **OverlayFS(overlay2)**를 사용했습니다. 테스트는 총 4가지 사례를 살펴볼 예정이며, 테스트에 사용된 분석 방식은 아래와 같습니다.

1. Dockerfile로 Docker Image를 빌드하고 `Inspect` 명령어를 통해 **이미지 layer들의 ID**를 확인합니다. (RootFS의 Layers를 확인합니다.)
2. 실제 이미지 레이어가 저장된 디렉토리로 이동해 각 레이어에 저장된 **변경점**을 확인합니다. (*'/var/lib/docker/overlay2'* 에 위치해있습니다.)
3. Dockerfile의 각 라인과 비교하며, 어떤 변경사항들이 레이어로 기록되었는지 확인하고, 레이어로 만들어지지 않은 라인도 체크합니다. 

> Overlay2의 경우 **도커 이미지 레이어의 ID가, 실제 저장된 디렉토리의 ID와 일치하지 않습니다**. 다만, 각 레이어 디렉토리는 자신 바로 하위 레이어의 ID를 담고 있는 `lower` 파일이 존재하는데, 이를 통해 레이어들의 순서를 찾을 수 있습니다. (최하단 레이어는 lower 파일이 존재하지 않습니다.)

<br>
##### 1) 중복 라인이 많은 데이터
위 공식 문서에 언급된 **COPY, RUN** 커맨드를 여러번 중복해서 실행시켜보았습니다. 같은 커맨드가 두 번 이상 실행될 때부터는 더 이상 파일 시스템에 변화를 주지 않지만, 이미지 레이어를 만드는지 확인해보았습니다.

**1.1. 이미지 빌드 후 Inspect**<br>

**1.1.1. Dockerfile 작성 내용:**<br>
RUN mkdir -p로 동일한 디렉토리를 계속해서 생성합니다. 그리고 같은 파일을 COPY로 여러번 복사시켰을 때 이미지를 각각 생성하는지 확인해보았습니다.

``` 
FROM alpine:latest

RUN mkdir -p /test
COPY ./test.txt .
COPY ./test.txt .
COPY ./test.txt .
RUN mkdir -p /test
RUN mkdir -p /test

RUN mkdir -p /test2
RUN mkdir -p /test2
COPY ./test.txt .
RUN mkdir -p test

CMD ["echo", "test123"]
```

**1.1.2. build된 이미지의 inspect 결과:** <br>
총 7개의 레이어가 만들어졌으나, 그 중 4개의 레이어는 같은 ID를 가집니다. 가장 아래에 있는 레이어는 bb01bd... 레이어 입니다.
```
"RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:bb01bd7e32b58b6694c8c3622c230171f1cec24001a82068a8d30d338f420d6c",
                "sha256:e73837c20ff7e118ae774bfc8563887950aa0b877ace5818406ca38cde408b5b",
                "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
                "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
                "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
                "sha256:4c60400fd9c5d9903cd0d9e5793486de0c8a6ff9e5a7a964a76603f793f0709d",
                "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49"
            ]
        },
```
<br>
**1.2. 각 레이어의 diff(변경사항) 확인**

**1.2.1. 'bb01b...' (첫 번째):**<br>
가장 최하단에는 기반 이미지인 alpine 이미지가 저장되어 있습니다. alpine 레이어는 하나의 레이어로 이루어져있기에 diff 안에는 alpine 이미지가 통째로 들어가있습니다.
```console
[root@ip-172-31-47-234 diff]# ls
bin  dev  etc  home  lib  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
**1.2.2. 'e7383...' (두 번째):**<br>
두번째 레이어는 Dockerfile의 두 번째 명령어인 `RUN mkdir -p /test` 실행 결과가 저장되었습니다. 디렉토리를 새로 만들어 파일 시스템에 변화가 생겼으니 새로운 레이어를 만듭니다.
```console
[root@ip-172-31-47-234 diff]# ls
test
```
**1.2.3. '2321f...' (세 번째):**<br>
세번째 레이어는 Dockerfile 세 번째 명령어인 `COPY ./test.txt .` 실행 결과가 저장되었습니다. 현재까지는 정상적으로 커맨드마다 레이어를 만드는 모습이 보입니다.
```console
[root@ip-172-31-47-234 diff]# ls
test.txt
```
**1.2.4. '2321f...' (네 번째, 다섯 번째):**<br>
네번째, 다섯번째 레이어도 Dockerfile 세 번째, 네 번째 명령어인 `COPY ./test.txt .` 실행 결과가 동일하게 저장되었습니다. 같은 파일을 여러번 COPY하는 구문임에도 불구하고 별도의 레이어로 생성되었습니다. 앞의 레이어와 다른점이 존재하지 않음에도 새로운 레이어가 추가되었습니다.
```console
[root@ip-172-31-47-234 diff]# ls
test.txt
```

...결국 moby/moby(도커 엔진 프로젝트) 소스코드를 직접 확인해보았는데, 해당 코드들이 빌드툴 및 버전에 따라 동작이 다르긴 하지만, COPY, ADD 커맨드의 경우 동작 도중에 반드시 신규 Layer를 형성하는 것으로 확인되었습니다. (해당 내용은 또 추후에 버전이 업그레이드되면서 변경될 수 있습니다)

**1.2.5. '4c604...' (여섯 번째):**<br>
여섯번째 레이어는 아래 보시는 바와 같이 `RUN mkdir -p /test2` 에 대한 변경사항입니다. Dockerfile 에서는 총 2번 실행했으나 하나의 레이어만 만들어졌습니다. `RUN` 커맨드의 경우 앞의 레이어와 diff가 존재하지 않는다면 레이어를 별도로 생성하지 않는 것으로 보입니다.
```console
[root@ip-172-31-47-234 diff]# ls
test2
```

**1.2.6. '2321f...' (일곱 번째):**<br>
일곱번째 레이어는 다시 `COPY ./test.txt .` 커맨드를 실행하고 또 새로운 레이어를 만들었습니다. 위 2,3,4 단계와 마찬가지로, 파일 시스템에 변화는 없으나 별도의 레이어를 구성했습니다. 레이어가 이렇게 중복되어도, 같은 데이터는 가장 상위의 레이어 것을 사용하기 때문에, test.txt 파일은 최종적으로 현재 데이터가 사용됩니다.
```console
[root@ip-172-31-47-234 diff]# ls
test.txt
```

추가적으로, `COPY ./test.txt .` 커맨드가 마지막 레이어를 구성한 것으로 보아, 나머지 두 `RUN`, `CMD` 커맨드는 레이어를 만들지 않은 것으로 보여집니다. (CMD의 경우 Docker Docs에도 메타데이터만 변경된다고 명시되어 있습니다.) 

<br>
##### 2) 기존 빌드 된 이미지 레이어의 재사용
위 1)번 테스트 결과를 이미지로 저장하고, 도커 파일을 조금 수정한 후 다시 이미지를 빌드하면 이전 과정에서의 이미지 레이어들을 잘 사용할까요? 테스트 결과를 확인해보았습니다. 

이번 테스트는 `$ docker inspect` 커맨드를 수행하여 나온 레이어를 참고해, 이전 이미지와 동일한 레이어를 참조하고 있는지 테스트해보겠습니다.

**2.1. 기존 이미지 레이어 확인:**<br>
기존 이미지는 위 테스트한 결과물과 동일합니다.

**Dockerfile**
```
FROM alpine:latest

RUN mkdir -p /test
COPY ./test.txt .
COPY ./test.txt .
COPY ./test.txt .
RUN mkdir -p /test
RUN mkdir -p /test

RUN mkdir -p /test2
RUN mkdir -p /test2
COPY ./test.txt .
RUN mkdir -p test

CMD ["echo", "test123"]
```

**RootFS Layers**
```shell
"Layers": [
    "sha256:bb01bd7e32b58b6694c8c3622c230171f1cec24001a82068a8d30d338f420d6c",
    "sha256:e73837c20ff7e118ae774bfc8563887950aa0b877ace5818406ca38cde408b5b",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
    "sha256:4c60400fd9c5d9903cd0d9e5793486de0c8a6ff9e5a7a964a76603f793f0709d",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49"
]
```

**2.2. 신규 이미지 레이어 확인:**<br>
신규 이미지는 기존 이미지에서 마지막 커맨드들을 조금 수정하여 빌드하였습니다. 새로 만들어진 Dockerfile과 이를 이용해 빌드한 결과는 다음과 같습니다.

**Dockerfile**<br>
```
FROM alpine:latest

RUN mkdir -p /test
COPY ./test.txt .
COPY ./test.txt .
COPY ./test.txt .
RUN mkdir -p /test
RUN mkdir -p /test

RUN mkdir -p /test3
RUN mkdir -p /test4
COPY ./test.txt .
RUN mkdir -p test

CMD ["echo", "abcd"]
```

위 dockerfile로 빌드한 결과물은 아래와 같습니다. `COPY ./test.txt .`가 중복해서 실행된 5번째 레이어까지는 이전 이미지와 동일하게 사용하며, 변경된 `mkdir` 결과만 새로운 레이어로 구성되었습니다.
```shell
"Layers": [
    "sha256:bb01bd7e32b58b6694c8c3622c230171f1cec24001a82068a8d30d338f420d6c",
    "sha256:62ce259ac4c37c0089b64d7730de9a712aed823d551cb3be0198e863770270e5",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49",
    "sha256:9185d911decdb64a3e66d677e10358db680d20d3129b04ddeff29d42049ed03b",
    "sha256:b045589602f06ac8099d43842a27af4baabb961c1b2124fb53a513b1b46f7723",
    "sha256:2321fd38793b1edf46c072f5e2a4ccddf389ba08f4cda5ef69f864aabb514c49"
]
```

실제 빌드 과정에서 보인 output들을 확인했을 때도, 이전에 이미 존재하는 레이어는 cache를 사용했다고 출력합니다. (`---> Using cache` 가 출력된 레이어들은 새로 만들지 않고, 기존 레이어를 사용합니다)
```console
[root@ip-172-31-47-234 test1]# docker build . -t test2
Sending build context to Docker daemon  3.072kB
Step 1/12 : FROM alpine:latest
 ---> 5e2b554c1c45
Step 2/12 : RUN mkdir -p /test
 ---> Using cache
 ---> 15097f0653e3
Step 3/12 : COPY ./test.txt .
 ---> Using cache
 ---> c653feac3674
Step 4/12 : COPY ./test.txt .
 ---> Using cache
 ---> f0d628b1c1c8
Step 5/12 : COPY ./test.txt .
 ---> Using cache
 ---> 35bee63626c0
Step 6/12 : RUN mkdir -p /test
 ---> Using cache
 ---> 95a6c1a40d6f
Step 7/12 : RUN mkdir -p /test
 ---> Using cache
 ---> ea31766c5ccb
Step 8/12 : RUN mkdir -p /test3
 ---> Running in ac61415c0785
Removing intermediate container ac61415c0785
 ---> cabc3a05a8a3
Step 9/12 : RUN mkdir -p /test4
 ---> Running in 9e1e6d074403
Removing intermediate container 9e1e6d074403
 ---> a792038ddccf
Step 10/12 : COPY ./test.txt .
 ---> 2d79f08b45bf
Step 11/12 : RUN mkdir -p test
 ---> Running in 3a1a76ddd163
Removing intermediate container 3a1a76ddd163
 ---> 557414197371
Step 12/12 : CMD ["echo", "abcd"]
 ---> Running in cf1a1f334b69
Removing intermediate container cf1a1f334b69
 ---> 8fd2042267cd
Successfully built 8fd2042267cd
Successfully tagged test2:latest
```

테스트 결과, 신규 이미지는 기존 이미지의 레이어를 잘 활용하여 변경된 부분만 빌드 및 저장된 것으로 확인되었습니다.

#### 테스트 결과
테스트 결과, Dockerfile이 각 커맨드들을 파싱하여 실제 동작을 수행할 때, 각 커맨드마다 레이어를 만들지 만들지 않을지 먼저 결정이 되고, 몇몇 `RUN`과 같은 커맨드들은 파일시스템에 변화가 없을 경우 레이어를 만들지 않는 것으로 보입니다. 또한 그렇게 만들어진 이미지 레이어들은 다른 이미지를 생성할 때도 활용되어 효율적으로 이미지가 관리되고 있는 것이 확인되었습니다.

그리고 이 테스트 결과는, 위에 명시하진 않았지만 엔진 버전마다 조금씩 다르게 나옵니다. 현재의 분석이 특정 문제를 해결하기 위함이 아니였기 때문에, 이 모든 차이점들을 다 분석하지는 않기로 했습니다. 중요한 것은 각 **Dockerfile의 커맨드들은 새로운 레이어를 만들 가능성이 있기 때문에, 중간에 불필요한 레이어가 만들어지지 않도록 개발과정에서 신경써야한다는 점**입니다.


## 더 나아가서 <small>(공간 효율적인 Dockerfile 작성 방법)</small>
앞선 분석을 통해 도커 이미지 레이어의 생성 원리를 이해하는 것에서 더 나아가, 이를 응용해 실제 도커를 사용하면서 **효과적으로 Dockerfile 을 작성하는 방법**에 대해 공유하고 마치겠습니다.

도커 이미지는 배포되는 과정에서 여러번 pull & push 되기 때문에 **이미지의 크기가 작을수록 배포 속도가 향상되며 disk 공간이 낭비되지 않습니다**. 앞서 언급했듯 도커 이미지는 층이 만들어져있기 때문에, 최종 이미지에 사용되지 않는 파일들이 중간 레이어에 남아있을 수 있습니다. 이미지 크기를 효율적으로 관리하기 위해선, 이러한 중간 레이어의 불필요한 크기를 줄여야 합니다.
이를 해결하기 위한 좋은 방법은, 레이어로 만들어질 필요 없는 커맨드들을 이전 커맨드와 합쳐서 실행시키는 것입니다.

실제 예시를 보며 설명드리겠습니다. 아래 예시는 Docker Docs에 언급되어 있는 간단한 예시로, 기본 **빌드 커맨드**에 **빌드 캐시를 삭제하는 커맨드**를 **하나의 커맨드로 합쳐** 실행하여, 레이어의 크기가 커지지 않도록 방지하는 예시입니다.

**Dockerfile (before)**
```shell
# syntax=docker/dockerfile:1
FROM ubuntu:18.04
LABEL org.opencontainers.image.authors="org@example.com"
COPY . /app
RUN make /app
RUN rm -r $HOME/.cache
CMD python /app/app.py
```

위와 같은 예시 Dockerfile이 있습니다. 위 파일의 4번째 라인이 `RUN make /app` 을 통해 일련의 빌드 과정을 수행하고, 5번째 라인에서 `RUN rm -r $HOME/.cache`를 수행해 이미지에 불필요한 캐시를 삭제하고 있습니다. 그러나, 이 경우 4번, 5번 라인이 각각 실행되기 때문에, 5번 라인에서 캐시를 삭제하더라도 4번의 이미지 레이어에서는 여전히 캐시가 남아있게 됩니다.

이 경우 캐시를 삭제하는 행위가 이미지의 크기를 줄이는데 전혀 영향을 주지 못합니다. 이를 효율적으로 변경하기 위해선 Dockerfile을 아래와 같이 변경하면 됩니다.

**Dockerfile (after)**
```shell
# syntax=docker/dockerfile:1
FROM ubuntu:18.04
LABEL org.opencontainers.image.authors="org@example.com"
COPY . /app
RUN make /app \
    && rm -r $HOME/.cache
CMD python /app/app.py
```

위와 같이 변경하면, 기존에 빌드하고 캐시를 삭제하던 과정이 하나의 레이어로 합쳐지게 되면서, 이미지 레이어 내에 해당 캐시 데이터는 어디에도 존재하지 않게 됩니다. 이제는 해당 캐시 크기만큼 이미지의 크기가 줄어드는 효과를 볼 수 있습니다.


## 마무리
간단하게 공식 문서나 구글링으로 찾아보려했으나, 구글링으로 나오는 내용들은 다 제각각 다르고.. 글들의 주장에는 근거가 명확치 않아 실제로 다양한 테스트를 해보며 분석하게 되었습니다.
그러나 특정 문제상황을 해결하기 위한 분석이 아니었기에, 각 버전마다 너무나 많은 차이점들이 있었고, 이 때문에 어디까지 살펴봐야하나 많이 헤매다 정리를 끝마친 것 같습니다.

한 가지 명확한 점은, **이 게시글로 분석한 내용도 추후 다른 엔진 버전이 업데이트되면 내용이 달라질 수 있다는 점**입니다. 다양한 오픈소스들이 빠르게 변화하고 있고, 공식 문서가 노후화 되거나 아예 존재하지 않는 경우도 있습니다. 

앞으로도 빠르게 변화하는 툴들 속에서 다양한 오픈소스를 접하게 될텐데, 가벼운 구글링 결과에만 맹신하지 않고 동작 원리를 이해하려는 최소한의 노력이 필요할 것 같습니다. 간단하게나마 테스트를 하여 수행자의 이해 속에서 툴을 사용하는 습관을 들여야겠습니다.

## (추가: 23 엔진 버전)
Docker 23 엔진버전부터는 Linux 환경의 기본 builder가 buildx(BuildKit) 으로 변경되었습니다. 신규 버전으로도 테스트를 해보았는데, 역시나 결과가 또 조금 다릅니다.

<figure>
    <img src="/assets/img/posts/2023-05-29/docker-release-23.png" class="img-fluid" width="80%">
    <figcaption><small>Docker Release Note 23.0</small></figcaption>
</figure>

23 이전버전까지는 moby 프로젝트 내에 있던 builder 툴을 기본으로 사용했는데, 이후부터는 moby의 별도 프로젝트였던 BuildKit을 default builder 로 사용하게 됩니다.

## 참고 자료
- <small>내용 참고 / <a href="https://docs.aws.amazon.com/ko_kr/amazondynamodb/latest/developerguide/WorkingWithItems.html#WorkingWithItems.ConditionalUpdate" target="_blank">Docker Doc - [DynamoDB] Working with items and attributes</a></small>
- <small>사진 출처(Freepik) / <a href="https://www.freepik.com/free-vector/blacksmith-with-hard-work-strength-symbols-flat-illustration_15329625.htm#query=blacksmith&from_query=smithy&position=1&from_view=search&track=sph" target="_blank">blacksmith with hard work - macrovector</a></small>
- <small>사진 출처(packt) / <a href="https://subscription.packtpub.com/book/application-development/9781788992329/1/ch01lvl1sec05/understanding-docker-images-and-layers" target="_blank">container filesystem</a></small>