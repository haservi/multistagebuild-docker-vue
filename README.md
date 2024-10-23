# vue를 이용하여 도커와 멀티스테이지빌드 도커 차이 확인하기

도커는 애플리케이션을 컨테이너화하여 개발, 배포 과정을 단순화하는 강력한 도구입니다.

멀티 스테이지 빌드는 도커 이미지의 크기를 줄이고, 빌드 속도를 개선하는 데 유용한 기능입니다.

도커파일만 이용할 경우에는 구조가 단순하며, 빠른 설정이 가능합니다. 하지만, 멀티스테이지 도커에 비해 이미지 크기가 크며, 중간 빌드 결과물(ex: 컴파일 도구, 라이브러리 등)이
 모두 포함되므로 리소스 낭비가 발생할 수 있습니다.

멀티스테이지 도커를 이용할 경우 아래와 같은 장점이 있습니다.

- 이미지의 크기를 줄일 수 있음
  - 개발 단계에서 사용된 도구나 라이브러리를 최종 이미지에서 제외함으로써 이미지의 크기를 최소화
- 보안을 강화
  - 불필요한 소프트웨어가 최종 이미지에 포함되지 않기 때문에 공격 범위를 줄일 수 있음
- 빌드 속도를 향상시킬 수 있음
  - 멀티 스테이지 빌드를 사용하면 필요한 단계만 실행하여 빌드 시간을 단축

위와 같은 장점이 있긴 하지만, 멀티스테이지 도커를 위한 커맨드를 작성하기 위해서 Dockerfile이 단계가 복잡할 수 있으며,
 빌드에 필요한 단계별 설정이 필요하기 때문에 초기 설정 시간이 더 걸릴 수 있습니다.

그렇지만 도커와 멀티스테이지도커의 이미지 크기가 최대 2~10배 정도까지 차이가 날 수 있습니다.

아래는 vue frontend를 init을 Dockerfile.single과 Dockerfile.multi로 이미지를 만들때 이미지의 크기의 차이를 확인할 수 있습니다.

![image](./images/image01.png#center)

실습을 하면 우선 [vue quickstart](https://vuejs.org/guide/quick-start)에서 프로젝트를 생성 후 해당 폴더에 아래와 같이 Dockerfile들을 추가합니다.

## Dockerfile.single Script

``` bash
FROM node:20.18-alpine

WORKDIR /app

# 의존성 파일 복사 및 설치
COPY package*.json ./
RUN npm install

# 애플리케이션 소스 코드 복사
COPY . .

# 빌드
RUN npm run build
RUN npm install -g http-server

EXPOSE 8080

CMD [ "http-server", "dist" ]

```

## Dockerfile.multi Script

``` bash
# build stage
FROM node:20.18-alpine as build-stage
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# production stage
FROM nginx:1.26-alpine as production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

정확히 속성들이 맞는지는 모르겠지만 일반 Docker에서는 node 이미지를 이용해 빌드 후 http-server로 배포한 것에 비해
 멀티스테이지 Dockerfile에서는 node에서 build만 이용하고, 실제 배포된 파일을 nginx를 통해 배포하게 됩니다.

이렇게 배포하니 이미지의 크기가 꽤 줄어든 것을 확인할 수 있습니다.

Dockerfile을 쉽게 배포하기 위해서 아래와 같이 docker-compose.yml 을 아래와 같이 설정하면 좋습니다.

## docker-compose.yml Script

``` yml
version: '3'
services:
  node-app-single:
    build:
      context: .
      dockerfile: Dockerfile.single
    ports:
      - "3000:8080"

  node-app-multi:
    build:
      context: .
      dockerfile: Dockerfile.multi
    ports:
      - "3001:80"
```

그리고 시작과 종료를 아래와 같이 하면 컨테이너와 이미지를 깔끔하게 정리할 수 있습니다.

```bash
# 시작
docker-compose up

# localhost:3000 -> 일반도커
# localhost:3001 -> 멀티스테이지빌드도커

# 종료
docker-compose down --rmi all

```

멀티스테이지빌드를 이용하면 `MSA 환경`에서 여러 이미지들이 배포될 때 이미지의 크기를 꽤 줄일 수 있으므로 용량 최적화에 좋을 것으로 생각됩니다.

또한, `.dockerginore`을 이용하면 불필요한 파일을 이미지에 올리지 않을 수도 있습니다.
