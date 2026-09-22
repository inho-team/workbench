# 이사(Director) 세션 실행 방식

결론: 이사(Director) 세션은 OMT의 headless 런타임(`claude -p --resume`, 스트림 JSON)으로 감싸서 실행한다.

## 맥락

앱은 OMT 구조 내에서 이사 역할을 하는 에이전트와 대화하는 인터페이스를 제공한다. 데스크톱 본체 앱이 이 에이전트 세션을 효율적으로 띄우고 상태와 출력을 읽어와야 한다.

## 대안

- OMT headless 런타임(`claude -p --resume`, 스트림 JSON) (선택)
- 전경 터미널(`orca terminal send/read`)을 통한 이사 세션 제어

## 결정

이사 세션 실행 방식으로 OMT headless 런타임을 결정한다. 이 선택은 사용자가 확정했다.

## 결과 및 감수하는 위험

- **통신 효율:** 표준 입출력을 통해 구조화된 JSON 데이터(`step_update`, `result` 등)로 상태와 메시지를 스트리밍받아 파싱하기 쉽고 화면 반영이 빠르다.
- **의존성:** OMT headless 런타임 구현과 계약 버전에 앱이 강하게 결합된다.
