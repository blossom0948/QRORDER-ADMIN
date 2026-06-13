# Firebase 무료 운영 연결 가이드

이 방식은 사장님 계정을 Firebase 콘솔에서 매번 만들지 않습니다. 사장님이 사장님 사이트에서 `Google로 시작하기`를 누르면 로그인한 Google 계정이 자동으로 매장 소유자가 됩니다.

## 1. Firebase에서 처음 한 번만 할 일

1. Firebase 프로젝트 생성
2. 요금제는 `Spark` 무료 플랜 유지
3. Firestore Database 생성
4. Authentication > Sign-in method로 이동
5. `Google` 제공업체를 켜고 저장
6. Authentication > Settings > Authorized domains로 이동
7. 아래 도메인이 없으면 추가

```txt
blossom0948.github.io
admin.blossom0948.cloud
```

중요: `Authentication Users`에서 사장님 이메일/비밀번호를 직접 만들 필요가 없습니다. Firestore에 `admins/{UID}` 문서를 만들 필요도 없습니다.

## 2. Firestore Rules 적용

Firebase Console에서 `Firestore Database > Rules`로 이동한 뒤, 저장소의 [firestore.rules](firestore.rules) 내용을 붙여넣고 게시합니다.

이 규칙은 다음을 막습니다.

- 로그인하지 않은 사용자의 메뉴 수정
- 다른 사장님 매장의 주문 조회
- QR 토큰이 없는 손님의 주문 생성
- 30개를 넘는 비정상 장바구니 주문

## 3. 사장님 사이트에서 할 일

1. https://blossom0948.github.io/QRORDER-ADMIN/ 접속
2. `Google로 시작하기` 클릭
3. 본인 Google 계정 선택
4. 매장명 입력
5. 테이블 수 입력
6. `매장 만들기` 클릭

그러면 Firestore에 아래 구조가 자동 생성됩니다.

```txt
stores/{storeId}
  name
  ownerUid

stores/{storeId}/menus/{menuId}
  name, desc, price, category, image, badge, options, soldOut, sortOrder

stores/{storeId}/qrSessions/{token}
  tableNo, active, expiresAt

stores/{storeId}/orders/{orderId}
  tableNo, qrToken, items, total, status, createdAt
```

## 4. 손님 QR 링크

매장 생성 후 사장님 사이트에 `테이블 QR 링크`가 표시됩니다.

예시:

```txt
https://blossom0948.github.io/QRORDER-CUSTOMER/?store=abc123&table=05&token=랜덤토큰
```

도메인 연결 후에는 사장님 사이트에서 생성되는 QR 링크가 아래 주소로 바뀝니다.

```txt
https://order.blossom0948.cloud/?store=abc123&table=05&token=랜덤토큰
```

이 링크를 QR 코드로 만들면 됩니다. QR 코드 이미지는 무료 QR 생성 사이트나 프린터 프로그램으로 만들 수 있습니다.

## 5. blossom0948.cloud 도메인 연결

사장님과 손님 사이트는 서로 다른 서브도메인으로 나누는 것을 권장합니다.

```txt
사장님 사이트: admin.blossom0948.cloud
손님 사이트: order.blossom0948.cloud
```

도메인을 산 곳의 DNS 관리 화면에서 아래 레코드를 추가합니다.

```txt
Type: CNAME
Name: admin
Value: blossom0948.github.io

Type: CNAME
Name: order
Value: blossom0948.github.io
```

그다음 GitHub 저장소 설정에서 각각 Custom domain을 입력합니다.

```txt
QRORDER-ADMIN    -> admin.blossom0948.cloud
QRORDER-CUSTOMER -> order.blossom0948.cloud
```

GitHub Pages의 `Enforce HTTPS`는 DNS 확인이 끝난 뒤 켭니다. DNS 전파와 HTTPS 인증서 발급은 시간이 걸릴 수 있습니다.

## 6. 실제 운영 흐름

상용 QR 오더도 보통 사장님에게 DB 콘솔을 직접 만지게 하지 않습니다. 가입, 매장 생성, QR 발급, 메뉴 관리를 관리자 화면에서 처리하고 내부 시스템이 데이터베이스를 대신 만듭니다. 지금 구현도 그 흐름에 맞춰 Firebase 콘솔 작업을 최초 설정으로만 줄였습니다.

## 7. 무료 한도 감각

Firebase Spark 플랜의 Cloud Firestore 무료 할당량은 작은 매장 MVP에는 충분한 편입니다. 단, 정확한 한도는 공식 가격표에서 다시 확인해야 합니다.

- Firestore 저장공간: 1GiB 수준
- 읽기: 하루 수만 회 수준
- 쓰기: 하루 수만 회 수준

작은 식당 한 곳에서 하루 수백 건 주문을 테스트하는 정도는 보통 무료 범위 안에서 시작할 수 있습니다.

## 8. 주의할 점

- Firebase `apiKey`는 웹 공개용 설정이라 코드에 있어도 됩니다.
- Firebase Admin SDK private key는 절대 GitHub에 올리면 안 됩니다.
- 메뉴 사진을 크게 업로드하면 Firestore 문서가 커질 수 있습니다. 실제 운영에서는 Firebase Storage로 분리하는 것이 좋습니다.
- 실제 선불 결제는 Firebase만으로 되지 않습니다. TossPayments 같은 PG가 필요합니다.

## 9. 현재 구현 상태

- 사장님 로그인: Google 로그인 기반 Firebase Auth
- 매장 생성: Firestore `stores`
- 기본 메뉴 자동 생성: `stores/{storeId}/menus`
- 테이블 QR 자동 생성: `stores/{storeId}/qrSessions`
- 주문 실시간 표시: `stores/{storeId}/orders`
- 기존 localStorage 데모: URL에 `store`와 `token`이 없을 때 fallback으로 유지
