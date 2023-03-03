---
title: 노드에 확장된 리소스 알리기
content_type: task
---


<!-- overview -->

이 페이지에서는 노드에 확장된 리소스를 명시하는 방법을 보여준다.
확장된 리소스는 클러스터 운영자에게 쿠버네티스가 알지 못하는
노드 레벨의 리소스들을 알릴 수 있도록 한다.




## {{% heading "prerequisites" %}}


{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}




<!-- steps -->

## 노드의 이름 가져오기

```shell
kubectl get nodes
```

연습을 위해 노드 중의 하나를 선택하자.

## 노드 중 하나에 새로운 확장된 리소스 알리기

노드에 새로운 확장 리소스를 등록하기 위해, 쿠버네티스 API 서버에 HTTP PATCH 요청을 보내자.
예를 들어, 노드에 4개의 동글(dongle)이 연결되어 있다고 가정하자.
아래는 노드에 4개의 동글 리소스를 알리는
PATCH 요청의 예이다.

```shell
PATCH /api/v1/nodes/<your-node-name>/status HTTP/1.1
Accept: application/json
Content-Type: application/json-patch+json
Host: k8s-master:8080

[
  {
    "op": "add",
    "path": "/status/capacity/example.com~1dongle",
    "value": "4"
  }
]
```

여기서 쿠버네티스는 동글이 무엇이고, 어떤 일을 하는지 알 필요가 없다.
이전 PATCH 요청은 쿠버네티스에게 노드가 동글이라고 불리는
4개의 무언가를 가지고 있다고 알려준다.

쉽게 쿠버네티스 API 서버에 요청을 보낼 수 있도록 프록시를 시작하자.

```shell
kubectl proxy
```

또다른 명령 창에서, HTTP PATCH 요청을 보낸다.
`<your-node-name>`을 노드 이름으로 대체하자.


```shell
curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "add", "path": "/status/capacity/example.com~1dongle", "value": "4"}]' \
http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

{{< note >}}
이전의 요청에서, `~1`는 patch 경로의 / 문자를 인코딩하기 위함이다.
JSON-Patch의 명령은 JSON-Pointer로 해석된다.
더 많은 정보는, [IETF RFC 6901](https://tools.ietf.org/html/rfc6901)의
섹션 3을 확인하라.
{{< /note >}}

결과는 노드가 동글 4개분을 가지고 있다는 것을 보여준다.

```
"capacity": {
  "cpu": "2",
  "memory": "2049008Ki",
  "example.com/dongle": "4",
```

노드의 상세 정보를 확인하자.

```
kubectl describe node <your-node-name>
```

결과는 동글 리소스를 다시 보여준다.

```yaml
Capacity:
 cpu:  2
 memory:  2049008Ki
 example.com/dongle:  4
```

이제, 애플리케이션 개발자들은 일정 수량의 동글을 요청하는 파드들을 생성할 수 있다.
[Assign Extended Resources to a Container](/docs/tasks/configure-pod-container/extended-resource/)를 확인해보자.

## 논의

확장된 리소스는 메모리와 CPU 리소스와 비슷하다. 예를 들어,
노드 위에서 실행되는 모든 컴포넌트들이 일정 부분 공유하는 노드의 메모리와 CPU 같이,
노드 위에서 실행되는 모든 컴포넌트들이 공유하는 일정 수의 동글들이 있을 수 있다.
그리고 애플리케이션 개발자가 메모리와 CPU를 일정 부분 요청하여
파드를 만들 수 있는 것 처럼, 동글들을 일정 수량 요청하여
파드를 만들 수 있다.

확장된 리소스는 쿠버네티스에서 불투명하다. 쿠버네티스는 그들이 무엇인지 아무것도 알지 못한다.
단지 쿠버네티스가 노드가 일정 분량을 가지고 있다는 것을 알 뿐이다. 확장된 리소스는
정수의 형태로 알려져야 한다. 예를 들어, 노드는 4개의 동글을 할당할 수 있지만, 4.5개의
동글을 할당할 수는 없다.

### 저장소 예제

노드가 800 GiB의 특수한 종류의 디스크 저장소를 가지고 있다고 가정하자.
이 특수한 저장소에 example.com/special-storage와 같은 이름을 생성할 수 있을 것이다.
그 후 특정 사이즈로 나눈 청크를 알린다. 예를 들어 100 GiB라고 해보자.
이 경우, 노드는 example.com/special-storage 타입의
리소스 8개를 알릴 수 있다.

```yaml
Capacity:
 ...
 example.com/special-storage: 8
```

만약 특수한 저장소에 임의의 수량을 가진 요청을 보내고 싶다면, 1 바이트 크기의 청크들로 나뉜
특수한 저장소를 알릴 수 있을 것이다. 이 경우, example.com/special-storage 타입의
800Gi개의 리소스를 알려지는 것이 된다.

```yaml
Capacity:
 ...
 example.com/special-storage:  800Gi
```

그러면 컨테이너는 특수한 저장소의 바이트를 800Gi까지의 임의의 숫자로 요청할 수 있을 것이다.

## 정리하기

아래는 노드에서 동글 알림을 삭제하는 PATCH 요청이다.

```
PATCH /api/v1/nodes/<your-node-name>/status HTTP/1.1
Accept: application/json
Content-Type: application/json-patch+json
Host: k8s-master:8080

[
  {
    "op": "remove",
    "path": "/status/capacity/example.com~1dongle",
  }
]
```

쉽게 쿠버네티스 API 서버에 요청을 보낼 수 있도록 프록시를 시작하자.

```shell
kubectl proxy
```

또다른 명령 창에서, HTTP PATCH 요청을 보내자.
`<your-node-name>`을 노드 이름으로 대체하자.

```shell
curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "remove", "path": "/status/capacity/example.com~1dongle"}]' \
http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

동글 알림이 제거됐는지 확인하자.

```
kubectl describe node <your-node-name> | grep dongle
```

(아무런 결과도 보이지 않을 것이다)




## {{% heading "whatsnext" %}}


### 애플리케션 개발자들을 위한 문서

* [Assign Extended Resources to a Container](/docs/tasks/configure-pod-container/extended-resource/)

### 클러스터 관리자들을 위한 문서

* [네임스페이스에 대한 메모리의 최소 및 최대 제약 조건 구성](/ko/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)
* [네임스페이스에 대한 CPU의 최소 및 최대 제약 조건 구성](/ko/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)



