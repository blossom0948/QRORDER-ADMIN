# 무료 기준 운영 연동 가이드

이 프로젝트는 지금 GitHub Pages에서 도는 정적 MVP입니다. 실제 매장처럼 여러 기기에서 주문을 공유하려면 아래 4가지를 순서대로 붙이면 됩니다.

## 1. 먼저 선택할 추천 조합

초보자 기준 추천은 `Supabase + TossPayments 테스트 결제 + Cloudflare Worker`입니다.

- Supabase: 주문, 메뉴, 품절 상태를 실시간 DB로 저장
- TossPayments: 결제 화면과 결제 승인 테스트
- Cloudflare Worker: QR 토큰 검증처럼 숨겨야 하는 서버 로직 처리
- GitHub Pages: 지금처럼 손님/사장님 화면 배포

Firebase도 좋지만, SQL 테이블로 주문/메뉴를 직접 확인하기에는 Supabase가 초보자에게 더 직관적입니다.

## 2. Supabase 무료로 준비하기

공식 무료 플랜 기준으로 API 요청, 인증 사용자, 작은 DB, Realtime을 시작할 수 있습니다. 무료 한도는 바뀔 수 있으니 가입 전 공식 가격표를 확인하세요.

1. https://supabase.com 에 가입합니다.
2. 새 프로젝트를 만듭니다.
3. `Project Settings > API`에서 아래 2개를 복사합니다.
   - Project URL
   - anon public key
4. `SQL Editor`에서 테이블 3개를 만듭니다.

```sql
create table menus (
  id text primary key,
  category text not null,
  name text not null,
  description text,
  price integer not null default 0,
  image text,
  badge text,
  options jsonb default '[]'::jsonb,
  sold_out boolean not null default false,
  sort_order integer not null default 0,
  updated_at timestamptz default now()
);

create table orders (
  id uuid primary key default gen_random_uuid(),
  table_no text not null,
  order_no text not null,
  items jsonb not null,
  total integer not null default 0,
  note text,
  payment_mode text default 'postpaid',
  status text not null default 'pending',
  created_at timestamptz default now()
);

create table qr_sessions (
  token text primary key,
  table_no text not null,
  expires_at timestamptz not null,
  active boolean not null default true
);
```

5. `Database > Replication` 또는 Realtime 설정에서 `orders`와 `menus`를 Realtime 대상으로 켭니다.
6. 손님 화면은 `orders`에 insert, `menus`를 select합니다.
7. 사장님 화면은 `orders` 상태를 update하고 `menus`를 insert/update/delete합니다.

초보 단계에서는 Row Level Security를 처음부터 복잡하게 걸지 말고, 테스트 데이터로 동작을 확인한 뒤 정책을 잠그는 순서가 안전합니다. 운영 전에는 반드시 RLS를 켜야 합니다.

## 3. Firebase로 할 경우

Firebase를 고르면 Cloud Firestore를 사용합니다. 공식 Spark 플랜은 Firestore 읽기/쓰기 무료 할당량이 있어서 작은 MVP 테스트에 충분합니다.

1. https://firebase.google.com 에 가입합니다.
2. Firebase 프로젝트를 만듭니다.
3. Web App을 추가하고 Firebase config 값을 복사합니다.
4. Firestore 컬렉션을 아래처럼 잡습니다.
   - `menus/{menuId}`
   - `orders/{orderId}`
   - `qrSessions/{token}`
5. 손님 화면은 `menus`를 구독하고 주문 시 `orders`에 문서를 추가합니다.
6. 사장님 화면은 `orders`를 실시간 구독하고 status를 바꿉니다.

Firebase 장점은 실시간 구독이 쉽다는 점입니다. 단점은 관리자 화면에서 SQL처럼 직접 데이터 보기에는 Supabase보다 덜 직관적입니다.

## 4. TossPayments 테스트 결제

무료로 실제 돈을 받는 결제가 아니라, 테스트 모드로 결제 흐름을 붙이는 단계입니다.

1. https://docs.tosspayments.com 에서 개발자센터에 가입합니다.
2. 테스트 클라이언트 키와 테스트 시크릿 키를 확인합니다.
3. 손님 화면에는 클라이언트 키만 넣습니다.
4. 시크릿 키는 절대 GitHub Pages 코드에 넣으면 안 됩니다.
5. 결제 승인 API 호출은 Cloudflare Worker, Supabase Edge Function, Firebase Cloud Functions 같은 서버에서 처리합니다.

정식 결제를 받으려면 사업자 정보, 정산 계좌, PG 심사, 수수료 조건 확인이 필요합니다. 테스트 모드는 무료로 개발 흐름을 확인하는 용도입니다.

## 5. 서버 QR 검증

GitHub Pages만으로는 QR 토큰을 안전하게 검증할 수 없습니다. 토큰 검증 비밀키가 브라우저에 노출되기 때문입니다.

무료 기준으로는 Cloudflare Worker가 가장 단순합니다.

1. Cloudflare에 가입합니다.
2. Worker를 하나 만듭니다.
3. 환경변수로 `QR_SIGNING_SECRET`을 저장합니다.
4. Worker가 `/verify?token=...&table=05` 요청을 받습니다.
5. Worker는 토큰이 유효한지 확인하고 `{ ok: true, table: "05" }`처럼 응답합니다.
6. 손님 화면은 QR 검증 성공일 때만 주문 버튼을 활성화합니다.

운영 QR은 단순히 `?table=05`만 넣으면 안 됩니다. 누군가 URL 숫자만 바꿔 다른 테이블 주문을 넣을 수 있기 때문입니다. 반드시 만료 시간이 있는 서명 토큰을 넣어야 합니다.

## 6. 절대 GitHub에 올리면 안 되는 값

- TossPayments secret key
- Supabase service role key
- Firebase Admin SDK private key
- QR 서명 비밀키
- 결제 웹훅 검증 secret

브라우저에 들어가도 되는 값은 Supabase anon key, Firebase web config, TossPayments client key처럼 공개용으로 설계된 키뿐입니다.

## 7. 개발 순서

1. Supabase에 `menus`, `orders` 테이블을 만든다.
2. 현재 `localStorage` 저장 코드를 Supabase select/insert/update로 바꾼다.
3. 사장님 화면에서 주문 상태 변경이 손님 화면에 실시간 반영되는지 확인한다.
4. Cloudflare Worker로 QR token verify API를 만든다.
5. 손님 URL을 `?table=05&token=...` 형태로 바꾼다.
6. TossPayments 테스트 결제를 붙인다.
7. 주문 취소/환불 정책과 결제 실패 처리를 추가한다.

이 순서대로 하면 무료 범위에서 실제 운영 구조에 가장 가깝게 갈 수 있습니다.
