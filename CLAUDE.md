# 냉파를 부탁해 — Claude용 설치·운영 안내

이 레포는 claude.ai 아티팩트로 동작하는 단일 HTML 앱(`naengpa.html`)이다. 사용자가 설치·업데이트·재고 채우기를 부탁하면 아래를 따른다.

## 설치 (새 아티팩트로 올리기)

1. `naengpa.html`을 수정하지 말고 그대로 Artifact 도구로 publish한다.
   - `capabilities`: `{"db": {}, "sample": {}, "downloads": true}`
   - `icon`: `"fridge"`
   - `description`: "냉장고·냉동실·실온 재고를 Claude와 대화로 관리하는 앱"
2. 발행된 링크를 사용자에게 알려 주고, claude.ai에서 열면 빈 냉장고로 시작한다고 안내한다. 처음 열 때 Claude 사용·파일 저장 허용을 묻는다는 것도 알린다.
3. 같은 사용자의 계정에 이미 이 앱 아티팩트가 있는지(Artifact `list`) 먼저 확인한다. 있으면 새로 만들지 말고 업데이트 절차를 따른다.

## 업데이트 (기존 링크에 덮어쓰기)

1. 사용자의 기존 아티팩트 URL을 찾는다(`list` 또는 사용자에게 묻기). 먼저 `read`한다.
2. 레포의 최신 `naengpa.html`을 그 `url`로 publish한다. `capabilities`는 설치 때와 같은 값을 다시 넘긴다.
3. 새 링크를 만들지 않는다. 저장된 데이터(재고·기록·대화)는 아티팩트 DB에 있으므로 publish해도 유지된다.
4. 사용자가 자기 복사본을 직접 고친 적이 있다면, 덮어쓰기 전에 그 차이를 알리고 확인받는다.

## 데이터 구조 (ArtifactData로 읽고 쓰기)

| 경로 | 문서 하나 | 필드 |
| --- | --- | --- |
| `items/{id}` | 지금 있는 재료 하나 | `name`, `memo`, `cat`, `icon`, `loc`, `qty`, `due`, `bought`, `addedAt`, `staple`(상비 재료면 true: 화면에서는 원래 종류 대신 '상비' 묶음에 보이고, 요리해도 양이 줄지 않음. 기한은 그대로 따짐) |
| `events/{YYYY-MM}` | 그 달의 기록 | `list`: `{id, ts, kind, itemId, name, icon, from, to}` 배열 |
| `kitchen/main` | 우리 집 주방 장비(홈 로고 옆 아이콘, 채팅으로 수정) | `tools`: 문자열 배열(화구 수·용량은 이름에 함께, 예: "하이라이트 2구"). 추천은 이 장비로 만들 수 있는 요리만 고른다 |
| `meals/log` | 해먹은 요리 | `list`: `{id, ts, name, uses, icons, kcal, from}` 배열(최근 300개). 메뉴 추천 때 최근 2주치를 보고 겹치지 않게 고른다 |
| `chat/main` | 앱 안 대화 | `messages` 배열(최근 60개) |

- `name`: 집에서 부르는 짧은 재료 이름. 브랜드와 용량은 넣지 않는다. 맛·종류(불닭맛, 저당)는 이름에 남긴다.
- `memo`: 비고. 브랜드, 용량·중량 등. 예: `"동원, 135g"`.
- `cat`: `veg`(채소·과일), `meat`(고기·해산물), `dairy`(계란·두부·유제품), `grain`(곡물·면·빵, 가루 포함), `ready`(간편식), `side`(반찬·김치), `snack`(간식·디저트), `bake`(베이킹 재료), `sauce`(소스·양념, 잼·스프레드 포함), `etc`.
- `icon`: 앱에 내장된 이름 중 하나. mushroom, spinach, tofu, tomato, milk, yogurt, zucchini, leek, egg, granola, chicken, shrimp, dumpling, rice, pasta, tuna, onion, potato, carrot, apple, cabbage, garlic, cheese, butter, steak, fish, bread, pizza, bottle, jar, ramen, cupramen, drybag, onigiri, fried, mealpack, pickle, kimchi, tteok, sweets. 맞는 게 없으면 비우면 종류별 기본 아이콘이 쓰인다.
- `loc`: `냉장` | `냉동` | `실온`.
- `qty`: 자유 글자. 셀 수 있으면 `"6개"`, `"2팩"`, 셀 수 없으면 `"많음"`/`"반쯤"`/`"조금"`, 모르면 `"있음"`.
- `due`: `YYYY-MM-DD` 또는 `null`(기한 없음). `bought`도 같은 형식이며 모르면 넣지 않는다.
- `kind`(기록): `add`, `eat`, `eatAll`, `discard`, `edit`.

## 처음 재고 채우기

사용자가 집에 있는 재료 목록을 주면:
1. 위 규칙으로 정리하되, 분류·날짜가 애매한 것은 임의로 정하지 말고 사용자에게 묻는다.
2. `items` 컬렉션에 `batch`로 넣는다(한 번에 최대 50개). 문서 id는 겹치지 않는 임의 문자열.
3. 초기 등록은 `events`에 남기지 않는다(기록이 등록 내역으로 덮이지 않게).

## 백업에서 복원

사용자가 앱의 백업 파일(`냉파-백업-YYYY-MM-DD.json`)을 주면: `items` 배열은 각 `id`로 `items/{id}`에, `events` 객체는 키(월)별로 `events/{월}`의 `list`에, `meals` 배열은 `meals/log`의 `list`에, `kitchen` 배열은 `kitchen/main`의 `tools`에, `chat`은 `chat/main`의 `messages`에 쓴다. 기존 데이터를 덮기 전에 사용자에게 확인받는다.
