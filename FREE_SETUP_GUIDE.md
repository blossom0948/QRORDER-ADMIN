# Firebase 무료 운영 연결 가이드

이제 메뉴, 주문, QR을 Firebase 콘솔에서 손으로 만들 필요가 없습니다. Firebase에서 처음 한 번만 준비하고, 이후에는 사장님 사이트에서 로그인 후 `매장 만들기`를 누르면 자동 생성됩니다.

## 1. Firebase에서 딱 한 번 할 일

1. Firebase 프로젝트 생성
2. 요금제는 `Spark` 무료 플랜 유지
3. Firestore Database 생성
4. Authentication에서 `Email/Password` 로그인 활성화
5. Authentication Users에서 사장님 계정 생성
6. 사장님 UID 복사
7. Firestore에 아래 문서 하나만 생성

```txt
admins/{사장님UID}
  role: "owner"
```

여기까지 했으면 콘솔에서 `menus`, `orders`, `qrSessions`를 직접 만들지 않습니다.

## 2. Firestore Rules 적용

Firebase Console에서 `Firestore Database > Rules`로 이동한 뒤, 저장소의 [firestore.rules](firestore.rules) 내용을 붙여넣고 게시합니다.

이 규칙은 다음을 막습니다.

- 로그인하지 않은 사용자의 메뉴 수정
- 다른 사장님 매장의 주문 조회
- QR 토큰이 없는 손님의 주문 생성
- 30개를 넘는 비정상 장바구니 주문

## 3. 사장님 사이트에서 할 일

1. https://blossom0948.github.io/QRORDER-ADMIN/ 접속
2. Firebase Authentication에서 만든 사장님 이메일/비밀번호로 로그인
3. 매장명 입력
4. 테이블 수 입력
5. `매장 만들기` 클릭

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

이 링크를 QR 코드로 만들면 됩니다. QR 코드 이미지는 무료 QR 생성 사이트나 프린터 프로그램으로 만들 수 있습니다.

## 5. 무료 한도 감각

Firebase Spark 플랜의 Cloud Firestore 무료 할당량은 작은 매장 MVP에는 충분한 편입니다. 단, 정확한 한도는 공식 가격표에서 다시 확인해야 합니다.

- Firestore 저장공간: 1GiB 수준
- 읽기: 하루 수만 회 수준
- 쓰기: 하루 수만 회 수준

작은 식당 한 곳에서 하루 수백 건 주문을 테스트하는 정도는 보통 무료 범위 안에서 시작할 수 있습니다.

## 6. 주의할 점

- Firebase `apiKey`는 웹 공개용 설정이라 코드에 있어도 됩니다.
- 사장님 비밀번호는 절대 코드나 채팅에 보내지 마세요.
- Firebase Admin SDK private key는 절대 GitHub에 올리면 안 됩니다.
- 메뉴 사진을 크게 업로드하면 Firestore 문서가 커질 수 있습니다. 실제 운영에서는 Firebase Storage로 분리하는 것이 좋습니다.
- 실제 선불 결제는 Firebase만으로 되지 않습니다. TossPayments 같은 PG가 필요합니다.

## 7. 현재 구현 상태

- 사장님 로그인: Firebase Auth
- 매장 생성: Firestore `stores`
- 기본 메뉴 자동 생성: `stores/{storeId}/menus`
- 테이블 QR 자동 생성: `stores/{storeId}/qrSessions`
- 주문 실시간 표시: `stores/{storeId}/orders`
- 기존 localStorage 데모: URL에 `store`와 `token`이 없을 때 fallback으로 유지
