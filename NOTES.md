# Dreamhouse LWC 실행 & 구조 정리

## 1. 실행 순서

Tomcat 같은 로컬 서버가 없다. 소스를 Salesforce 클라우드의 **내 org**(나만의 Salesforce 인스턴스)에 업로드(deploy)해야 컴파일되고 동작한다.

| # | 작업 | 명령 / 비고 |
|---|---|---|
| 1 | 저장소 fork & clone | |
| 2 | Salesforce Developer 계정 가입|
| 3 | Salesforce CLI(`sf`) 설치 | https://developer.salesforce.com/tools/salesforcecli |
| 4 | org 로그인 (브라우저 인증) | `sf org login web -s -a mydevorg` |
| 5 | 기본 대상 org 지정 | `sf config set target-org mydevorg` |
| 6 | 소스 업로드 | `sf project deploy start -d force-app` |
| 7 | 앱 조회 권한 부여 | `sf org assign permset -n dreamhouse` |
| 8 | 샘플 데이터 넣기 | `sf data tree import -p data/sample-data-plan.json` |
| 9 | 브라우저로 열어 UI 확인 | `sf org open` → 앱 런처 → Dreamhouse |

> 참고: 여기서 쓴 org는 **Developer Edition org**(계속 쓰는 개인 org)다.
> README의 다른 방법인 **Scratch org**는 Dev Hub로 만드는 일회용 테스트 org(기본 7일)라서 서로 다르다.

## 2. 구조와 흐름

### 핵심 개념
- 진입 JS(`main.jsx` 같은 것)가 없다. 페이지 껍데기(헤더, 탭 바, 라우팅)는 **Salesforce Lightning 런타임**이 갖고 있다.
- 개발자는 XML 메타데이터로 **어떤 페이지에 어떤 컴포넌트를 어디에 둘지**만 선언한다.
- 파일 이름이 곧 API 이름이고, 메타데이터끼리 이 이름으로 서로를 참조한다.

### 업로드 대상
`sfdx-project.json`의 `packageDirectories`에 적힌 폴더(`force-app`)가 org에 올릴 소스다. 배포할 때만 쓰이고 런타임에는 관여하지 않는다.

### 화면이 열리는 흐름 (Property Finder 탭 예시)

"GetMapping 시점"에 가장 가까운 건 사용자가 URL로 들어오는 순간이다. 이 URL 규칙은 Salesforce에 내장돼 있어서 컨트롤러를 매핑하지 않는다.

```
https://<org>.lightning.force.com/lightning/n/Property_Finder   ← 탭 API 이름

applications/Dreamhouse.app-meta.xml
  <tabs>Property_Finder</tabs>                 대메뉴(탭 바) 정의
        ▼
tabs/Property_Finder.tab-meta.xml
  <flexiPage>Property_Finder</flexiPage>       탭을 누르면 열릴 페이지
        ▼
flexipages/Property_Finder.flexipage-meta.xml  레이아웃과 컴포넌트 배치
  left   : barcodeScanner, propertyFilter
  center : propertyListMap
  right  : propertySummary, daysOnMarket
        ▼
lwc/propertyFilter/                            실제 UI 부품 (js / html / css / js-meta.xml)
```

### 탭 종류
| 탭 | 정의 위치 | 열리는 화면 |
|---|---|---|
| `Property_Explorer`, `Property_Finder`, `Settings` | `tabs/`의 `<flexiPage>` | `flexipages/`의 AppPage |
| `Property__c`, `Broker__c` | `tabs/`의 `<customObject>` | 객체 목록 → 레코드 상세(`Property_Record_Page` 등) |
| `standard-home`, `standard-Contact`, `standard-File` | 없음 (Salesforce 기본 탭) | Salesforce 기본 화면 |

### LWC 규칙
- 폴더 이름, `.js`, `.html` 파일의 기본 이름이 같아야 한다 (`propertyTile/propertyTile.js`).
- 페이지에 배치하려면 `*.js-meta.xml`에 `isExposed=true`와 `target`(AppPage, RecordPage, HomePage 등)을 선언한다.
- HTML에서 다른 컴포넌트는 `c-` + kebab-case 이름으로 쓴다: `<c-property-tile>`는 `lwc/propertyTile/`을 가리킨다.

### 데이터 흐름 (Spring MVC와 비교)
```
Spring : fetch('/api/..') → @GetMapping Controller → Service → Mapper XML → JSON → JSP/JS
LWC    : import getPagedPropertyList from '@salesforce/apex/PropertyController.getPagedPropertyList';
         @wire(getPagedPropertyList, {...}) properties;   → classes/PropertyController.cls (@AuraEnabled, SOQL)
```
- URL 매핑이 없고, import 경로가 곧 서버 메서드에 바인딩된다.
- 반환 객체(`PagedResult`)는 자동으로 JSON 직렬화된다.
- `@wire`는 파라미터(`'$searchKey'`)가 바뀌면 자동으로 다시 호출한다.
- 단순 레코드 조회는 Apex 없이 `lightning/uiRecordApi`의 `getRecord`로 할 수 있다.

### 컴포넌트 간 통신
같은 페이지에 배치된 컴포넌트들은 공통 부모 JS가 없어서 `messageChannels/`의 채널(이벤트 버스)로 통신한다.
```
propertyFilter  --publish FiltersChange-->    propertyTileList, propertyListMap (Apex 재조회)
propertyTile 클릭 --publish PropertySelected--> propertySummary (상세 표시)
```
