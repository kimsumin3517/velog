---
title: Vercel 무료 플랜으로 조직 Private 레포 배포하기 - GitHub Actions 우회기
series: 배포·인프라 (1/1)
published:
velog:
---

Vercel은 레포를 연결만 하면 push마다 자동 배포되는 마법 같은 경험을 준다.
그런데 무료(Hobby) 플랜에는 잘 알려지지 않은 제약이 있다.
**조직(Organization) 소유의 Private 레포는 연결 자체가 안 된다.**

팀 프로젝트는 보통 조직 레포에 있다.
막히는 지점이 정확히 여기다.

## 선택지 정리

1. **레포를 Public으로 전환** - 무료로 직접 연결 가능. 대신 코드가 공개되고, Hobby 제약상 계정 소유자의 커밋만 자동 배포된다
2. **레포를 개인 계정으로 이전** - 개인 Private 레포는 무료로 되니까. 대신 레포가 조직 밖으로 나간다
3. **유료(Pro) 결제** - 정석. 월 $20
4. **포크해서 개인 레포를 경유** - 되긴 하는데, 조직 레포에 머지될 때마다 포크를 수동 동기화해야 한다. 배포가 자동이 아니게 된다

한동안 4번으로 버텼는데, "머지했는데 배포는 옛날 버전"인 날이 반복됐다.
그러다 다섯 번째 길을 찾았다.

## 원리: Git 연동은 방아쇠일 뿐이다

Vercel의 구조를 뜯어보면, Git 연동이 하는 일은 "push를 감지해서 빌드를 시작"하는 방아쇠 역할이다.
호스팅·CDN·배포 기록·롤백 같은 본체는 그 뒤에 있다.
그렇다면 **방아쇠만 GitHub Actions로 바꾸면** 된다.
빌드를 GitHub 러너에서 하고, 결과물만 Vercel CLI로 업로드하는 것이다.
Vercel이 공식 가이드로 안내하는 지원 경로이기도 하다.

```yaml
name: Vercel Production Deployment

on:
  push:
    branches: [main]

env:
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install --global vercel@latest
      - run: vercel pull --yes --environment=production --token=${{ secrets.VERCEL_TOKEN }}
      - run: vercel build --prod --token=${{ secrets.VERCEL_TOKEN }}
      - run: vercel deploy --prebuilt --prod --token=${{ secrets.VERCEL_TOKEN }}
```

세 단계의 의미:

- **vercel pull** - Vercel에 등록된 프로젝트 설정·환경변수를 러너로 내려받는다. 빌드가 Vercel 밖에서 도니까 필요하다
- **vercel build** - 러너에서 빌드를 돌려 결과물을 만든다
- **vercel deploy --prebuilt** - 결과물만 업로드한다. `--prebuilt`가 없으면 Vercel이 한 번 더 빌드해서 시간이 2배가 된다

시크릿 3개(토큰, 조직 ID, 프로젝트 ID)만 레포에 등록하면 끝이다.

## 밟은 함정 두 개

**함정 1: 프로젝트 스코프 토큰**
Vercel 토큰을 만들 때 최소 권한 원칙으로 특정 프로젝트 스코프를 골랐더니, CI에서 `Could not retrieve Project Settings` 에러가 났다.
프로젝트 스코프 토큰은 CLI의 설정 조회를 통과하지 못한다.
팀 스코프 + All Projects로 재발급하니 해결됐다.

**함정 2: Sensitive 환경변수**
환경변수를 실수로 Sensitive로 저장했는데, 배포 전에 확인해보니 `vercel pull`이 내려받은 파일에 값 대신 문자열 `[SENSITIVE]`가 들어 있었다.
Sensitive 변수는 Vercel 서버 안에서 빌드할 때만 복호화되므로, **러너에서 빌드하는 구조에서는 값을 받을 수 없다.**
그대로 배포했다면 모든 API 요청이 `[SENSITIVE]/api/...`라는 주소로 나갈 뻔했다.
외부 빌드 구조에서 환경변수는 Sensitive 없이 저장해야 한다.

## 덤으로 얻은 것

Hobby의 Git 연동에는 "계정 소유자의 커밋만 자동 배포"라는 제약도 있는데, CLI 배포는 커밋 author가 아니라 **토큰**으로 인증하므로 이 제약이 사라진다.
팀원이 머지해도 배포가 정상적으로 돈다.
우회로였는데 팀 협업 관점에서는 오히려 더 튼튼해졌다.

## 정리

- Vercel의 본체는 호스팅·CDN·롤백이고, Git 연동은 교체 가능한 방아쇠다
- 무료 + 조직 + Private을 다 지키는 길은 GitHub Actions + CLI 조합
- 프로젝트 스코프 토큰과 Sensitive 환경변수는 이 구조에서 지뢰다
- 나중에 Pro로 옮기려면 워크플로 지우고 대시보드에서 연결하면 끝 - 갈아타는 비용이 거의 없다
