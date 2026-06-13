# QRORDER Admin

사장님용 QR 테이블오더 운영 사이트입니다.

## 배포 URL

https://blossom0948.github.io/QRORDER-ADMIN/

## 주요 기능

- Google 로그인 기반 사장님 온보딩
- 로그인 후 매장 만들기
- 기본 메뉴와 테이블 QR 자동 생성
- 주문 대기, 조리중, 완료 상태 관리
- 새 주문과 직원 호출 딩동 알림
- 메뉴 추가, 삭제, 품절, 가격, 설명, 옵션, 사진 수정
- Firestore 실시간 주문 동기화

## Firebase 설정

초기 설정은 [FREE_SETUP_GUIDE.md](FREE_SETUP_GUIDE.md)를 따릅니다.

사장님 계정이나 `admins/{UID}` 문서는 수동으로 만들지 않습니다. Firebase에서 Google 로그인을 켠 뒤, 사이트에서 `Google로 시작하기`를 누르면 됩니다.
