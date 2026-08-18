---
title: fetch 하나로 만드는 인증 API 클라이언트 - 인터셉터, 에러 정규화, CORS
series: 프론트엔드 인증 구현 (3/5)
published:
velog:
---

토큰을 저장했으니 이제 API를 호출할 차례다.
그런데 화면마다 fetch를 쓰면서 각자 토큰을 챙기게 하면, 인증 로직이 바뀔 때마다 모든 화면을 고쳐야 한다.
이 글은 **모든 요청이 지나가는 관문 하나**를 만든 기록이다.

## fetch의 유명한 함정부터

```ts
const res = await fetch(url);   // 404여도, 500이어도 여기서 안 터진다
if (!res.ok) { /* 직접 확인해야 한다 */ }
```

fetch는 **네트워크 자체가 실패할 때만** reject하고, 4xx/5xx는 "응답을 잘 받았음"으로 친다.
`res.ok` 체크를 빼먹는 게 가장 흔한 지뢰다.
이런 체크를 화면마다 반복하지 않으려면 한 곳에 모아야 한다.

## 관문 패턴

```
화면 컴포넌트
   ↓ "갤러리 목록 줘" (토큰? 그게 뭔지 모름)
도메인 API 함수
   ↓
apiClient  ← 토큰 주입, 에러 변환이 모두 여기서 (관문)
   ↓
fetch
```

axios의 interceptor가 유명하지만, fetch를 감싸는 함수 하나로 같은 걸 만들 수 있다.
관문이 하는 일은 세 가지다.

**① 토큰 자동 주입 - 기본값은 "켬"**
API 대부분이 인증 필수라면, 다수 쪽을 기본값으로 삼아야 "새 API 함수에서 토큰 깜빡함" 같은 반복 버그가 원천 차단된다.
기본값 설계의 원칙은 **깜빡했을 때 안전한 쪽이 기본**이어야 한다는 것.

**② 공개 API는 명시적으로 "끔"**
반대로 로그인·재발급 같은 공개 API에는 토큰을 보내지 않아야 한다.
만료된 토큰이 실려 가면, 서버의 인증 필터가 경로를 따지기 전에 토큰부터 검증하다 401을 뱉을 수 있다.
"로그인하려는데 옛 토큰 때문에 로그인이 거부되는" 황당한 버그의 씨앗이다.

```ts
export function login(provider: string, body: AuthCode) {
  return api(`/api/v1/oauth/${provider}`, { method: "POST", body, auth: false });
}
```

**③ 에러 정규화**
백엔드가 에러를 `{ code, message }` 형태로 준다면, 그 체계를 화면까지 전달해야 한다.

```ts
export class ApiError extends Error {
  constructor(
    public readonly status: number,   // 410
    public readonly code: string,     // "INVITE_EXPIRED"
    message: string,                  // "만료된 초대 링크입니다."
  ) { super(message); }
}
```

화면 코드는 `err.code`로 분기만 하면 된다.
잊지 말 것 하나 - 서버가 에러를 응답한 것(ApiError)과 서버에 아예 닿지 못한 것(비행기 모드, fetch가 TypeError를 던짐)은 사용자 안내가 달라야 하는 별개 상황이다.
관문이 이 둘을 구분해서 내보내야 한다.

## CORS, 그리고 요청이 두 줄로 보이는 이유

프론트(localhost:3000)에서 백엔드(api.example.com)를 호출하면 origin이 다르다.
브라우저는 **다른 origin으로의 요청에 상대 서버의 허락**을 요구한다 - 이게 CORS다.
서버가 응답 헤더로 "이 origin은 허락함"을 밝혀야 브라우저가 응답을 열어준다.

주의할 점: origin은 프로토콜+도메인+포트 전체다.
`www.example.com`과 `api.example.com`은 서브도메인만 달라도 남남이라, 백엔드 허용 목록에 프론트 주소가 정확히 등록되어 있어야 한다.
실제로 등록 안 된 origin으로 호출했다가 403을 받고 한참 헤맸다.

그리고 개발자 도구 Network 탭을 보면 같은 요청이 두 줄로 보인다.
하나는 Type이 fetch, 하나는 preflight.
**preflight는 브라우저가 본요청 전에 자동으로 보내는 사전 확인(OPTIONS)**이다.
"이 origin에서 이런 헤더로 POST를 보내려는데 괜찮아?"라고 서버에 먼저 묻는 것.
내 코드가 보낸 게 아니니 당황하지 않아도 된다.

## 미래를 위한 이음새 하나

관문을 만들 때 요청 실행을 함수 하나로 모아뒀다.

```ts
async function request<T>(path: string, options: ApiOptions): Promise<T> { ... }

export async function api<T>(path: string, options: ApiOptions = {}): Promise<T> {
  return request<T>(path, options);   // 지금은 그냥 통과
}
```

`api()`가 지금은 한 줄이지만, 다음 글에서 이 안쪽에 "401이면 재발급 후 재시도" 로직이 끼어든다.
그때 **호출하는 화면 코드는 한 글자도 바뀌지 않는다.**
바뀔 것을 아는 지점을 미리 함수 경계로 잘라두는 것 - 이런 걸 이음새(seam)를 만든다고 한다.

## 정리

- fetch는 4xx/5xx에 안 던진다. 체크는 관문 한 곳에서
- 토큰 주입은 기본 켬, 공개 API만 명시적으로 끔 - 깜빡해도 안전한 쪽이 기본값
- 서버의 에러(코드 있음)와 네트워크 실패(코드 없음)는 별개 상황이다
- preflight는 버그가 아니라 브라우저의 표준 동작이다
