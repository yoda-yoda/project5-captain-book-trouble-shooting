# 5차 프로젝트 - **대장부(Captain Book) 개인프로젝트 (captain book project)**

## 🏅 트러블 슈팅 (현재 약 80개)

<img width="120" height="120" alt="DaeJangBu-radius-logo" src="https://github.com/user-attachments/assets/cb7f16af-1c36-4f55-b2e9-54b7f24ff938" />

---

### 리포지터리 링크

* **트러블 슈팅 :** https://github.com/yoda-yoda/project5-captain-book-trouble-shooting
* **소개 링크 :** https://github.com/yoda-yoda/project5-captain-book-overview
* **backend (spring) :** https://github.com/yoda-yoda/project5-captain-book-backend
* **frontend (react) :** https://github.com/yoda-yoda/project5-captain-book-frontend

---


# 트러블슈팅1

### 문제:

가계부 달력을 만들고, 세부 항목인 item으로 접속하는 컨트롤러, 뷰를 만들고 접속해봤더니 404 오류가 발생했다.

### 모색:

경로와, html 내의 타임리프 문법을 자세히 살폈다.

### 원인:

원인은 세부항목으로 이동하는 `<a>` 태그의 문법이 원인이었다.

즉 처음엔

```
<a href="@{|/calendar/${calendar.id}/item|}">

```

이렇게 작성해서 타임리프임을 표시하는 th 작성을 하지않았다.

그 뒤 th를 넣어서

```
<a th:href="@{|/calendar/${calendar.id}/item|}">

```

이렇게 수정했지만 이것도 문제가 있었다.

form 태그에서는 위와같은

`@{|...|}` 문법이 잘 작동했었기때문에 문제없을거라 생각했다.

왜냐하면

```
<form th:action="@{|/calendar/delete/${calendar.id}|}" method="post">

```

이렇게 form 태그에서는 `@{|...|}` 안에 `${}`를 사용하는 문법이 잘 작동했기 때문이다.

그러나

th:action 이 아닌,

th:href 에서는 엄격하게 `@{|...|}` 안에 `${}`를 쓰는 걸 권장하지 않으며,

경우에 따라 파싱이 정확히 안 될 수 있고,

브라우저에 따라 예상과 다르게 처리된다고 한다.

### 해결:

문법을

```
<a th:href="@{/calendar/{id}/item(id=${calendar.id})}">

```

이렇게 바꾸니 해결되었다.

---

# 트러블슈팅2

### 문제:

아이템 삭제 버튼 클릭시 주소창에 해당 달력의 id값이 null이었고 400오류가 발생했다.

### 모색:

파라미터 바인딩이 안되는 것 같았다.

타임리프 경로와 컨트롤러 경로를 확인해봤는데 문제가 없었다.

컨트롤러 메서드를 다시 천천히 살펴봤다.

### 원인:

```
@PostMapping("/calendar/{calendarId}/item/{calendarItemId}/delete")
public String calendarItemDelete(@PathVariable Long calendarItemId, Long calendarId){...}

```

위처럼 `@PathVariable`이 하나에만 붙어있었고, calendarId에는 붙어있지 않았다.

알고보니 `@PathVariable`은 URL에 있는 변수 개수만큼 필요했다.

### 해결:

`@PathVariable Long calendarItemId, @PathVariable Long calendarId`

이렇게 붙여 해결하였다.

---

# 트러블슈팅3

### 문제:

달력의 세부항목에 대해 수정 버튼을 눌렀다.

그러나 오류는 없었지만 수정되지 않은 기존 값이 출력되었다.

### 모색:

컨트롤러 메서드 내에서 log를 확인해보았다.

컨트롤러 내의 업데이트 메서드가 끝나고나서 log를 출력해보면 정상적으로 수정된 값이 출력되었다.

그리고 업데이트 메서드가 끝나고 리다이렉트로 조회 하는 컨트롤러 메서드로 가서 log를 출력해보면 기존값이 출력되었다.

트랜잭션 문제인것 같다는 생각이 떠올랐다.

### 원인:

의도치않은 트랜잭션 처리를 원인으로 판단하였다.

### 해결:

서비스 계층 메서드에 @Transactional 어노테이션을 활용하며 테스트 진행후 해결하였다. 

---

# 트러블슈팅 4

### 문제:

item 항목값을 수정하고 저장했을때, 오류는 안나지만 화면에는 기존 값이 출력되었다. 아래는 관련 컨트롤러와 update 코드다.

```
@GetMapping("/calendar/{calendarId}/item/{calendarItemId}/update")
public String calendarItemUpdate(@PathVariable Long calendarItemId, @PathVariable Long calendarId, Model model){

    CalendarItemResponseDto calendarItemDto = calendarItemService.getCalendarItemDto(calendarItemId);

    model.addAttribute("calendarItemDto", calendarItemDto);
    model.addAttribute("calendarId", calendarId);

    return "item-update";
}

```

```
@PostMapping("/calendar/{calendarId}/item/{calendarItemId}/update")
public String calendarItemUpdate(@PathVariable Long calendarItemId,
                                 @PathVariable Long calendarId,
                                 @ModelAttribute CalendarItemUpdateDto calendarItemUpdateDto){

    calendarItemService.updateItem(calendarItemId, calendarItemUpdateDto);

    Optional<CalendarItem> byId = rep.findById(calendarItemId);
    CalendarItem calendarItem = byId.get();
    log.info(" calendarItem.getItemTitle() = {}", calendarItem.getItemTitle());
    log.info(" calendarItem.getItemAmount() = {}", calendarItem.getItemAmount());

    return "redirect:/calendar/" + calendarId + "/item";
}

```

```
public void updateItem(Long calendarItemId ,CalendarItemUpdateDto calendarItemUpdateDto) {

    CalendarItem entity = calendarItemRepository.findById(calendarItemId)
            .orElseThrow(() -> new CalendarItemNotFoundException(ExceptionMessage.CalendarItem.CALENDAR_ITEM_NOT_FOUND_ERROR));

    entity.updateFromDto(calendarItemUpdateDto);
}

```

```
@GetMapping("/calendar/{calendarId}/item")
public String itemHome(@PathVariable Long calendarId, Model model){

    List<CalendarItemResponseDto> calendarItemResponseDtoList = calendarItemService.getAllCalendarItem(calendarId);

    CalendarResponseDto calendarResponseDto = calendarService.getCalendarDtoById(calendarId);

    model.addAttribute("calendarResponseDto", calendarResponseDto);

    model.addAttribute("calendarItemResponseDtoList", calendarItemResponseDtoList);

    return "item";
}

```

위와 같은 코드였다.

### 모색:

마지막에 조회되는 컨트롤러 메서드를 자세히 살펴봤다.

에러가 나지는 않으니 순간 DB에 반영이 안된 트랜젝션 관련 문제일거라는 생각이 스쳤다.

### 원인:

메서드에 `@Transactional` 어노테이션을 붙이지 않았고, 따라서 수정 값이 DB에 영구 반영되지 않은게 원인이었다.

즉 리다이렉트 직후 해당 EM은 닫혔고 해당 엔티티의 변경사항은 커밋되지 않은 채로 EM이 닫혔기 때문에, 마지막으로 조회할때는 DB에서 기존 데이터를 조회할 수 밖에 없던것이다.

중간 궁금점:

그러나 `@PostMapping` 의 calendarItemUpdate 메서드 안에서 log를 출력해봤을때 도대체 왜 수정된 값이 잘 출력되었던 건지 헷갈렸다.

그것은 영속성컨텍스트(Entity Manager), 1차 캐시, FlushMode.AUTO 개념 때문 이었다.

스프링에서는 기본적으로 `OpenEntityManagerInViewFilter` 라는 기능이 켜져 있어서, 영속성컨텍스트(EM)가 기본적으로 하나의 요청이 들어올때 새로 열리고 렌더링 후 닫힌다.

EM 안에서 한번 조회한 엔티티는 EM이 닫히기 전까지 관리 대상이 된다. 관리 대상은 1차 캐시가 적용된다. 즉 메모리에 기억된다. 그래서 DB에 SQL을 날려 실제 값을 직접 가져오지 않더라도 1차캐싱된 값을 가져오기에 더 빠르고 효율적으로 작동한다.

그리고 스프링은 기본적으로 `FlushMode.AUTO` 로 작동한다.

이것은 쿼리가 날아갈때 EM의 내용을 자동으로 flush 하는 기능이다.

참고로 flush는 DB에 쿼리를 날려 반영하는 기능이지만, commit과는 다르다.

flush는 해당 트랜젝션 내부에서만 반영된 DB값이 보이는 것이고, commit은 모든 트랜젝션 관점에서도 영구반영되어 그 값이 곧 영구 적용되는 것이다.

그래서 어떻게 그게 가능했던 건지 살펴봤다.

먼저 `@PostMapping` 의 calendarItemUpdate 메서드를 살펴보면 그 안에

```
calendarItemService.updateItem(calendarItemId, calendarItemUpdateDto);

```

가 있다.

그리고 그 `updateItem` 메서드는 다음과 같다.

```
public void updateItem(Long calendarItemId ,CalendarItemUpdateDto calendarItemUpdateDto) {

    CalendarItem entity = calendarItemRepository.findById(calendarItemId)
            .orElseThrow(() -> new CalendarItemNotFoundException(ExceptionMessage.CalendarItem.CALENDAR_ITEM_NOT_FOUND_ERROR));

    entity.updateFromDto(calendarItemUpdateDto);
}

```

먼저 처음 `@PostMapping` 으로 요청이 왔을때 EM이 열린다.

그리고 `updateItem` 메서드를 실행한다.

`updateItem` 내부에서

```
CalendarItem entity = calendarItemRepository.findById(calendarItemId)

```

로 entity 가 EM에 1차캐싱된다.

그리고

```
entity.updateFromDto(calendarItemUpdateDto);

```

로 엔티티의 값을 바꿨다. 이때 DB에 반영되진않았다.

그리고 `@PostMapping` 의 calendarItemUpdate 메서드로 돌아온다.

그리고나서

```
Optional<CalendarItem> byId = rep.findById(calendarItemId);
CalendarItem calendarItem = byId.get();

```

여기서 `findById()` 가 실행된다. 이때 트랜젝션이 열린다.

(왜냐하면 스프링은 repository 의 기본 메서드 중 읽기 메서드는 `@Transactional(readOnly = true)`가 그리고 쓰기 메서드는 `@Transactional` 이 내부적으로 붙어서 실행시 transaction이 열리고 메서드가 종료될때 닫히기 때문이다.)

그리고나서 select 쿼리를 날린다. 그런데 `FlushMode.AUTO` 로 인해, EM 안의 변경사항이 자동으로 flush된다.

그리고 flush 된 DB의 값은 해당 트랜젝션 안에서는 잘 보인다.

따라서

```
log.info(" calendarItem.getItemTitle() = {}", calendarItem.getItemTitle());
log.info(" calendarItem.getItemAmount() = {}", calendarItem.getItemAmount());

```

이 두 log에서는 수정된 엔티티를 잘 가져왔고 제대로 출력이 된것이다.

그리고 auto flush가 아니었더라도, 해당 엔티티는 해당 EM 내에서 추적되어 1차캐싱된 엔티티이기 때문에 변경값이 적용되어 log가 잘 출력될 것이다.

### 해결:

서비스 계층 `updateItem` 메서드에 `@Transactional` 어노테이션을 붙여 영구 반영하도록 해결하였다.

---

# 트러블슈팅 5

### 문제:

Calendar 엔티티 수정 기능을 만들고나서 작동했을때, 수정 처리 후 마지막 재조회 단계에서 예외가 발생했다.

### 모색:

오류 메시지를 보니, 모든 Calendar 리스트를 보여주는 home.html의 List가 isEmpty() 였다고 나왔다.

### 원인:

컨트롤러의 return 관련 문제였다.

수정 처리후 그대로 `return "home";` 이라고 적었기때문에 home.html 에서 꺼내는 모델명과 일치하지 않아서 생긴 문제였다.

### 해결:

컨트롤러 내용을 리다이렉트로 바꿔 해결하였다.

즉 `return "home"` 을 `return "redirect:/home"` 으로 바꿔 해결하였다.

---

# 트러블슈팅 6

### 문제:

어플리케이션을 시작하자마자 ITEM_TYPE 테이블에 데이터를 추가해놔야했기때문에 resources에 data.sql 파일을 만들어 sql을 저장해두었다.

그러나 세부항목을 추가 기능을 작동시킬때 ItemType 엔티티를 찾을 수 없다는 예외가 나왔다.

### 모색:

h2 콘솔로 확인해보니 ItemType 테이블만 생성되어있고 데이터가 삽입되지않았다.

data.sql 파일의 문법 등을 확인해보았으나 문제가 없었다.

따라서 다음과 같이 yml틀 통해 log를 확인해보았다.

```
logging:
  level:
    org.springframework.jdbc.datasource.init.ScriptUtils: DEBUG

```

### 원인1:

INSERT 하는 sql이 log에 출력되지않았다. 즉 data.sql이 작동하지않았던것이다.

### 해결1:

yml 파일에

```
spring:
  sql:
    init:
      mode: always

```

를 추가하여 해결하였다.

기본값은 mode가 `embeded` 였고, `always`로 확실히 명시해줌으로써 sql 초기화 쿼리가 잘 작동하는 것을 log로 확인할 수 있었다.

### 원인2:

초기화 sql이 잘 작동하긴했지만 또다시 테이블 안에 데이터가 삽입되지않았다.

출력 내용을 자세히 보니, hibernate 가 create table 쿼리를 날리기 이전에 초기화 쿼리를 날렸던 것이 원인같았다.

즉 테이블이 만들어지지도 않았는데 초기화 쿼리를 날려서 적용되지않았다.

### 해결2:

yml에

```
spring:
  jpa:
    defer-datasource-initialization: true

```

를 추가하여, 테이블 생성이 되고나서 sql이 작동하도록 했다.

---

# 트러블슈팅 7

### 문제:

모달창 `<div>`를 구현했을때 문제가 생겼다.

html `<table>`내부의 타임리프 forEach 스코프 안에서는 `${...}` 문법을 통해 calendar 객체 id에 접근할 수 있지만, `<table>` 태그 밖 스코프에서는 해당 객체에 접근할 방법이 없었다.

### 모색:

스코프를 충족하기위해 `<table>` 안에 컴포넌트를 넣자니 코드가 매우 길고 복잡해질것이었다.

따라서 html 태그 내부에 속성을 정의하고 값을 담아둘 수 있는 방법을 찾아보았다.

### 원인:

부트스트랩의 모달창 컴포넌트인 모달`<div>`를 `<body>` 태그 안의 맨 밑에 둔것이 원인이었다.

### 해결:

html5부터 지원하는 속성값을 담아두는 방법이 있었다.

태그 내부에 `[data-*]` 문법으로 속성 키를 정의할 수 있었고, 그 속성에 값을 담을 수 있었다.

이 속성과 자바스크립트 click 이벤트 핸들러로 문제를 해결하였다.

타임리프 용법으로는 `th:attr` 으로 사용할 수 있었다.

(예시 => `th:attr="data-calendar-id=${calendarResponseDto.id}` )

자바스크립트 click 이벤트 핸들러 함수로 클릭된 버튼의 DOM 객체로 접근하여 속성값을 얻은 후, 모달창 `<div>` 속성에 값을 전달하는 방법을 사용했다.

---

# 트러블슈팅 8

### 문제:

부트스트랩으로 구현한 수정 모달창을 열어서 확인 버튼을 누르면 모달창이 닫히도록 설정했지만, 뒤로가기를 하면 부트스트랩 기본 효과 때문에 이동이 부드럽지 않고 UX 상 껄끄럽게 느껴졌다.

### 모색:

처음엔 `pageshow` 이벤트를 활용하려했다.

따라서 `pageshow` 이벤트를 통해 뒤로가기를 감지하면 `querySelector`로 `editModal` DOM 을 수동으로 초기화하려했다.

예를들면

```
window.addEventListener('pageshow', function (event) {
  if (event.persisted === true) { ... }

```

처럼 `pageshow` 이벤트와 `event.persisted` 의 boolean 값을 활용해 뒤로가기를 감지하고나면 모달 객체의 style 이나 class 명 등을 바꿔 자동 적용을 풀어내려했다.

그러나 뒤로가기를 하고나서 풀어지는 시간이 부족해 UX 상 여전히 부드럽지않았다.

또한 바꿔야 하는 것을 전부 수동으로 바꿨다 하더라도, 수정 버튼을 2번 눌러야 다시 작동하는 버그가 있었다.

### 원인:

부트스트랩 모달창이 열리고 닫힐때 동적으로 적용되는 기본 효과 때문이었다.

버그는 수동으로 하다보니 부트스트랩의 내부 작동방식과 꼬인것으로 보인다.

### 해결:

뒤로가기 event가 아니라 모달창 수정 확인 버튼 click event에 리스너를 등록하였다.

그렇게 하니 확인을 누르고 처리되는 시간이 UX 상 자연스럽게 느껴졌다.

리스너의 함수를 구현한 방법은 modal 객체가 초기화 되기 전과 후의 속성값을 비교하여 구현했다.

또한 부트스트랩 내부 방식이 꼬인 버그 해결방법은, 수동 절차가 끝난 마지막 줄 코드에 부트스트랩에서 제공하는 모달 초기화 메서드인 `dispose()` 를 이용하여 꼬인것을 해결하였다.

---

# 트러블슈팅 9

타임리프 기반의 프로젝트를, 리액트로 변경하다가 생긴 렌더링 관련 문제.

(리액트로 변경한 이유: 타임리프 기반으로는 어느정도 틀을 알고있다는 생각이 들었다. 따라서 실무에서 많이 쓰이고, 학습이 필요한 리액트를 처음 시도해보기로 하였다.)

### 문제:

BootStrap 템플릿 활용 기반을 타임리프에서 리액트로 변경하는 과정이었다.

부트스트랩의 js 자원을 등록하기위해서 React 앱 `index.html` 에 `<script>` 태그를 추가하고 React를 실행하였다.

그런데 `main.js` 내부에서 이벤트리스너 요소가 null이라는 오류가 발생했고, 브라우저 화면이 제대로 렌더링 되지않았다.

### 모색:

`<script>` 속성에 `defer` 나 `async` 속성을 이용하는 방법을 모색했다.

### 원인:

브라우저의 js파일 실행 순서가 원인이었다. 그러나 `defer`, `async`를 해도 안됐던것은 `bundle.js`가 가장 나중에 실행되기때문이었다.

React의 컴포넌트들은 `bundle.js`로 합쳐져서 기존 `<script>` 중에서 가장 아래쪽에 추가된다.

브라우저는 가장 나중에 추가되는 `bundle.js`를 가장 나중에 실행하고, 따라서 그전에는 `main.js`가 필요로하는 값이 존재하기 않기때문에 오류가 발생하여 제대로 렌더링이 되지 않았다.

### 해결:

App 컴포넌트의 `useEffect()` 를 통해 작동 순서를 바꿈으로써 해결하였다.

`<script>main.js</script>` 구문을 `useEffect()` 에 등록해놓았고 따라서 요소가 실제 DOM에 추가되고나서 `main.js` 를 실행시키므로 없는 요소를 찾는 오류가 해결되었다.

---

# 트러블슈팅 10

### 문제:

스크롤이 조금 내려간 상태에서, 브라우저 새로고침을 한다면 처음엔 스크롤이 맨 위로 가도록 초기화 하고싶었는데, 이전 스크롤 위치를 기억하여 자동으로 돌아가는 것이 문제였다.

### 모색:

`useEffect` 로 `scrollY` 값을 초기화하려했지만, 곧 다시 자동으로 스크롤이 작동했다.

### 원인:

이전 스크롤 위치를 기억하는 브라우저의 기본 기능때문이었다.

브라우저가 이전 위치를 기억해 자동 스크롤하는 시점을 찾아보았다. 보통 그 시점은 `'load'` event 의 발생직전, 직후였다.

### 해결:

```
window.addEventListener('load', () => {
    window.scrollTo(0, 0);
});

```

이렇듯 `'load'` 에 대한 event Listener를 설정하여 scroll이 처음으로 가도록 초기화하였다.

---

# 트러블슈팅 11

### 문제:

어떤 함수 내부에서 `setState` 함수를 호출했지만 상태 변경이 되지않았다.

### 모색:

컴포넌트 내부에서 `console.log()` 를 이용해 작동 순서와 횟수를 살펴보았다.

### 원인:

원인은 `setState()` 를 감싸고있는 함수가 렌더링 되지않는데 상태 값도 옛날 값으로 고정되어 기억되는것이 문제였다.

즉 함수도 리렌더링이 되지않고 인자 또한 `setState(data+1)` 처럼 작성하면서 최신값으로 바뀌지않았다.

### 해결:

`setData(data + 1)`은 내부적으로 "기억하는 숫자 data에 1을 더하는것" 이고,

`setData(data  => data + 1)`은 "지금 실제 최신 숫자를 적용하고, 거기에 1을 더하는것" 이었기 때문에 뒤의 방법을 선택하여 해결하였다.

---

# 트러블 슈팅12

### 문제:

버튼을 실제 DOM 요소로 추출하려 하는데, 쿼리셀렉터로 직접 추출하면 종종 렌더링 시점에 따라 null 값이 들어갈 수도 있어서 `useRef` 훅을 사용해 Dom을 추출하고 같은 참조 객체를 자식 컴포넌트의 props로 넘기는 방식으로 처리하려했지만 형제관계여서 불가능했다.

### 모색:

(1) 우선 형제 컴포넌트에서 `useRef` 로 연결한 DOM요소를 또다른 형제컴포넌트로 넘기는 일반적인 방법을 찾지못해서 형제 컴포넌트를 부모 컴포넌트에 포함하도록 바꾸려했다.

그러나 같은 레벨의 섹션으로 이루어졌기때문에 부모 컴포넌트로 구성하기에는 역할이 달랐다.

(2) `useContext()` 훅을 이용하면 전역적으로 이용할 수는 있었다.

(3) 그런데 `useRef` 훅을 이용하고싶었고, 또한 해당 Dom 요소를 안전하게 추출한다고해도 이벤트리스너 등록방식보다 React의 `onClick` 방식이 더 안전했다.

그렇다면 `onClick` 핸들러를 부모 컴포넌트나 형제 컴포넌트에서 선언해야하는데, 핸들러 로직은 다른곳에 있었기때문에 등록하기가 부적절했다.


### 원인:

추출할 Dom 요소가 부모-자식 컴포넌트 관계가 아니라 형제관계여서 props로 넘길수가없었다.


### 해결:

결국 콜백 레지스트리(handlerRef) 방식을 찾았다.

이 방식은 `useRef`의 `current`값에 DOM 요소 자체를 담아 그것을 공유하는게 아니라, `current`값에 콜백 함수 자체를 담아 `current`에 함수 참조를 형제끼리 공유하는 방식이었다.

이것을 이용하면 다른곳에있는 버튼 DOM을 추출할 필요가 없어서 추출과 렌더링 순서 문제에서 안전했고 핸들러는 형제 컴포넌트 내부에서 따로 다룰수 있기때문에 적절한 방법이었다. 또한 이벤트리스너 등록이아니라, `onClick` 방법이라 더 안전했다.

---


# 트러블 슈팅13

### 문제:

`useRef`와 콜백 레지스트리(handlerRef) 방식을 혼합해서 함수를 여러곳에서 공유하려했다.

`useRef`로 객체를 만들어서 `current`값에 함수를 공유하도록 했다. 그런데 함수값이 공유 되지않았다.

### 모색:

`console.log` 로 `current` 값이 전역적으로 잘 갱신되었는지 각각 확인해보았다.

부모컴포넌트의 `current` 값과, 함수를 등록하는 컴포넌트의 `current` 값은 잘 갱신되었다.

그러나 button 을 직접 다는 컴포넌트의 `current` 값은 여전히 초기값이었다.

### 원인:

크게 2가지 원인이 있었다.

1. props를 받을때 `current`값 자체를 전달받게되면 `current` 값은 그 순간의 값으로 계속 기억되어 캡처링 되는 문제가 원인이었다.

즉 매개변수에서 전달받을때 `(props)` 또는 `( {aProps} )` 처럼 props 객체 자체 또는 props 내부의 Ref 객체 자체를 받아야 최신 객체로 연동이되는데,

`( {aProps.current} )` 처럼 전달받으면 그 값이 캡처(복사)되고 남아있어서, 옛날 값만 유지된것이 원인이었다.

1. `onClick`에 캡처된 `current` 값을 등록한게 원인이었다.
    
    `current` 값에는 함수가 담기는데, 초기값은 빈 함수로 설정하고 최신 함수는 다른곳에서 갱신하도록 구성했다.
    
    그러나 `onClick={ref.current}` 처럼 등록했다.
    
    이렇게되면 `current` 값은 선언 시점의 값이 캡쳐(복사)되어 옛날값을 계속 유지하기 때문에 최신 함수가 갱신되지 않았다.
    

### 해결:

props를 받을때 ref 객체 자체를 받아 최신 객체를 유지하였다.

그리고 `onClick={ () => ref.current() }` 처럼 익명함수 내부에서 런타임에 해당 최신값을 읽어 실행하도록 바꿔 해결하였다.

---

# 트러블 슈팅14

### 문제:

위 문제로 디버깅을 하다가 이해가 안가는 부분이 생겼다.

알고있던 리액트앱의 작동 순서와, `console.log`에 출력되는 순서가 이상했다.

`useRef` 객체 안의 값을 확인해보고 싶어서 A컴포넌트 안에 동기코드 `console.log(ref)` 를 적었다.

그리고 `current` 값을 갱신하는 코드는 B컴포넌트의 `useEffect()` 안에 있었다.

그러면 `console.log` 값에는 ref의 초기 코드가 출력되어야 했다.

`useEffect()`는 가상DOM 생성 => 실제DOM 생성 => paint 이후에 실행되는데 동기 코드는 그 이전에 실행되기 때문이다.

그런데 콘솔에 먼저 출력된 값은 최신값이었다.

### 모색:

`<Strict>` 모드 때문인가싶었지만 그것도 2번씩 실행되는것뿐이지 리액트앱 실행 순서 자체를 바꾸지는 않았을것이라 생각했다.

### 원인:

원인은 2가지 였다.

첫째는 콘솔 UI가 그려지는 시점이었다.

즉 동기코드 `console.log()` 의 실행 자체는 즉시 실행되지만, F12의 개발자 도구 "콘솔 UI에 글자가 적혀 나타나는 시점"은 `useEffect()` 코드 실행 이후일 수도 있었다.

둘째는 `ref` 객체가 라이브객체라는 것이다.

따라서 `ref`라는 라이브 객체를 `console.log(ref)` 로 출력할때는 `console.log()` 의 "실행 시점에 저장된 값"이 아니라 "콘솔 UI에 글자가 나타나는 그 순간의 ref 최신값" 인 규칙이 있었다.

### 해결:

위의 원리로 작동한것을 알고 다소 의문이 해결됐다.

---

# 트러블슈팅15. 너무 기쁜 트러블 슈팅.

### 문제:

상황1: 콜백 레지스트리 방식으로 이벤트 리스너를 Dom 요소에 안전하게 등록한 상황이었다.

상황2: 그리고 첫 로딩할때는 달력 섹션이 화면과 실제 DOM에서도 숨겨져있고, 시작 버튼을 처음 클릭할때만 화면에 `<section id=services>`인 달력 섹션을 삽입하도록 핸들러를 구성한 상황이었다.

상황3: 버튼에 `<a href=#services>` 처럼 앵커가 달려있어서, 자동스크롤이 작동해야하는 상황이었다.

그런데 시작 버튼을 클릭하면, 섹션이 DOM에 생성은 됐으나 자동스크롤이 일어나지 않는 문제가 발생했다.

### 모색:

당시 컴포넌트의 `visible` 상태 값이 true 인지 false인지에 따라 화면에 보이느냐 마느냐를(DOM을 생성하느냐 마느냐를) 설정하기 위해 `return ()`을 `if`문으로 분기하였다.

한번 더 누르면 자동스크롤이 작동했다. 따라서 작동 순서의 문제라고 생각하고 살펴봤다.

### 원인:

브라우저의 내장 기능인 앵커 기능이 너무 빨리 작동한것이 원인이었다.

DOM에서 `<section id=services>`가 실제로 생기기도 전에, `#services` 로 앵커기능이 작동하여 자동스크롤이 되지않았던것이었다.

앵커기능이 작동하기전에 렌더링을 완료시켜야했다.

### 해결1:

무거웠던 `onClick` 핸들러의 내용을, `set()`만 실행시키도록 바꿔서 리렌더링이 즉시 실행되도록 하였다.

그러면 잘 작동하긴했지만, 순서가 보장되지는 않았다.

왜냐하면 브라우저 기본 기능인 앵커 기능은, 존재하는 `#section`으로 그저 바로 이동하는 기능뿐이라 굉장히 빨리 작동하는데, 컴포넌트를 불러 다시 초기화하는 `set()`의 리렌더링 기능이 과연 그것보다 빨리 작동할수 있을 보장이 없었기 때문이다.

그래서 그냥 `visible`이 true이든 false든 앵커 기능이 바로 잘 작동할수 있도록, `<section id=services>` 태그를 DOM에 존재하도록 바꾸려 했지만, 태그 요소의 css관련 설정이 복잡하여 다른 방법을 찾아보았다.

### 해결2:

브라우저의 내장 앵커기능을 이용하지않고도 `useRef`와 `scrollIntoView` 기능을 이용해 앵커기능을 똑같이 구현하여 해결하였다.

우선 `useRef`를 이용해 `<section id=services>` 요소를 안전하게 참조했다.

그리고 `visible`이 true로 바뀐뒤 요소가 실제 DOM에 생기면, 그때 `useEffect`가 작동하며 해당 요소의 스크롤을 직접 이동시키는 `servicesSection.current.scrollIntoView({behavior: 'smooth'})` 코드를 실행하도록 구성했다.

이렇게 안전한 순서를 보장하면서도 브라우저의 앵커 기능을 똑같이 구현할수있는 방법으로 해결하였다.

---

# 트러블슈팅16

### 문제:

이전 트러블 슈팅으로 해결한 코드를 더 안전하게 작동시키기위해 `if (visible)` 를 추가해 visible이 true일때만 내부 코드가 작동하도록 리팩토링하였는데, 버튼을 다시 누르면 내부 코드가 작동하지않았다.

### 모색:

미작동이 의심가는 부분에서 `console.log()`로 출력해보았다.

### 원인:

해당 핸들러 함수를 `useCallback`으로 메모이제이션한 상태였고, 그 때 내부 변수 `visible`은 함수 정의 시점의 값으로 계속 고정되는 캡쳐링이 원인이었다.

### 해결:

`useCallback`에 의존성배열 `[visible]`을 추가해서, `visible` 값이 바뀌면 함수 또한 최신 상태로 갱신되도록 만들어 해결하였다.

---

# 트러블슈팅17

### 문제:

새로운 컴포넌트가 마운트될때, 해당 컴포넌트 바로 밑의 요소가 불필요하게 타겟 영역에 개입되어 UI, UX에 방해가 되었다.

### 모색:

타겟 요소의 밑에 있는 요소가 방해할 수 없도록하는 방법을 찾아보았다.

### 원인:

해당 컴포넌트의 내용이 짧다보니 화면에서 차지하는 비율이 적었고, 그에 따라 밑의 요소가 타겟 영역에 침범한것이 원인이었다.

### 해결:

해당 컴포넌트에 `min-height` 고정 요소를줘서 화면 일정 영역을 보장하여 해결하였다.

---

# 트러블슈팅18

### 문제:

`navigator()` 기능과 `<Route>` 기능을 이용하던중 어느 순간 브라우저 새로고침을 하니 js와 관련된 RunTime 오류가 발생했다.

### 모색:

라우팅 기능을 본격적으로 사용하고 나서부터 이런 현상이 발생해서 그것을 염두에 두고, 브라우저 콘솔에 나타나는 오류 항목을 살펴보았다. js 파일 로딩이 실패한것처럼 보였다.

### 원인:

자원을 상대경로로 적은 것이 원인이었다.

새로고침시 전체적인 순서를 살펴보면 브라우저는 새로고침시, 현재 url로 서버에 직접 자원을 요청한다.

그리고 서버는 현재 url에 따른 자원을 찾는다.

그런데 리액트 개발용 서버는 url에 대한 `<Route>`도 없고, 정적 자원도 존재하지 않으면 기본적으로 `index.html` 를 브라우저에 응답하도록 설정되어있다.

따라서 브라우저에서는 다시 `bundle.js` 을 실행하며 리액트앱이 새로 실행된다.

그러면서 필요한 자원을 다운로드 받는 과정에서 문제가 발생한것이다.

`index.html`에는 `<script src="./js/aos.js"></script>` 이렇게 상대경로로 js 파일을 설정해놓았다.

브라우저는 이것을 보고, 현재 url에서부터 탐색을 시작한다. 그런데 url 기능을 사용하면서부터 그 경로에 해당 js 파일이 없기때문에 오류가 발생한것이다.

### 해결:

상대경로를 나타내는 `.`을 빼고 절대경로로

```
<script src="/js/aos.js"></script>

```

처럼 수정해 해결하였다.

---

# 트러블슈팅 19

(큰 스트레스 / 2틀만에 원인 실마리잡아서 기쁨)

### 문제:

의문점이 생겼다.

1. 첫째 문제로는 `console.log` 출력에 왜 나오는지 모르겠는 리렌더링 출력이 계속 나타난 것이고,
2. 둘째 문제로는 그 출력의 순서가 이해되지 않던 것이었다.
    
    기존에 알고있던 작동 순서와 개념이 실제와 상충하고 너무 헷갈려 작동 원리를 제대로 알고싶었다.
    

### 모색:

코드에 순서를 확인할 `console.log()` 를 세부적으로 추가하고, Strict 모드를 일시적으로 풀고 다시 확인해보았다.

연습용 리액트 프로젝트를 만들어 비슷한 환경을 만들어 테스트해보았다.

또한 관련 개념을 깊이있게 학습하였다. (라우터는 Location Context를 관리하고 있다는 것, return jsx => `React.createElement`로 변환 => 정보를 담은 객체로 변환 => 그것을 기반으로 Fiber 노드 트리가 구성 된다는것 등)

### 원인:

url 변경으로 Location Context가 갱신되고, 그 갱신 값을 구독자들에게 업데이트시키는 리렌더링이, `setState()`로 인한 리렌더링 작업 이후에 발생하는 것이 원인이었다.

우선 리액트 라우터는 내부에 location Context를 관리하고 있어서 항상 location을 감시한다.

`useNavigator()` 훅, `<Routes>` 컴포넌트는 이 location context를 구독하고 있었다.

따라서 `useNavigator()` 로 반환받은 navigator로, `navigator('/home')` 을 실행하면 url이 바뀐다.

라우터는 이를 감지하여 구독자들의 리렌더링을 예약한다.

그래서 `useNavigator()`와 `<Routes>`를 사용하고있는 컴포넌트가 리렌더링 되어 Context를 갱신해야했다.

그런데 이 경우 `setState()` 함수와 `navigator('/home')` 이 같은 스코프에 있었는데, 보통은 리렌더링 예약이 함께 batch 처리되어 1개의 리렌더링 작업으로 합쳐져서 완료될 때도 있지만, 때때로 `setState()`는 내부적으로 업데이트 큐로 스케줄링을 하고, 라우터의 스케줄링은 그와 별개로 작동하기에 분리될 때가 있다.

그래서 `setState()`로 인한 컴포넌트 리렌더링이 먼저 일어나고, 그것이 끝나고나서 Context 갱신을 위한 리렌더링이 발생한것이다.

따라서 결론적으로 컴포넌트 리렌더링이 재출력 된것과, 그 출력 순서가 이상해보였던 것의 원인은 `setState()`로 인한 컴포넌트 리렌더링과 Context 갱신을 위한 리렌더링이 분리됐던것이다.

### 해결:

출력 이유와 순서에 관한 개념을 깊게 학습해 의문을 해결하였다.

---

# 트러블슈팅 20

### 문제:

의문점이 생겼다.

많은 시간을 투자해서 공부하고 정리해놨던 것을 나중에 다시 확인해보니 실제와 달라 의문이 생겼다. 그 부분은 다음과 같다.

"`<Routes>`가 어떤 부모 컴포넌트의 jsx 안에 직접 위치해 있는 경우에는, `<Routes>`가 호출되려면 부모 컴포넌트 자체가 다시 호출(리렌더링)된다. 그 이유는 부모 컴포넌트가 다시 실행되어야 그 안의 모든 JSX가 새로 평가되고 `<Routes>`도 그 과정에서 새로 생성되기 때문이다."

그런데 이런 현상은, 반은 맞고 반은 틀렸었다.

### 모색:

관련 개념을 학습하고, 작은 리액트 앱에서 비슷한 환경을 만들고 `console.log` 출력으로 테스트를 해보았다.

### 원인:

위 현상은 `<Route>`라는 자식 컴포넌트에 전달되는 props가 이전과(가장 최근값과) 다를때 일어나는 일이었다.

그게 아닌 경우 `<Routes>` 작동시 반드시 부모 컴포넌트가 호출되는것은 아니다.

### 해결:

관련 개념을 학습하고, 작은 리액트 앱에서 비슷한 환경을 만들고 `console.log` 출력으로 테스트한 후 의문을 해결하였다.

결과적으로 현재 기억해둘것은 2가지였다.

1. 리액트는 컴포넌트 단위로 렌더링 되며, 컴포넌트의 `props`, `state`, `context`가 바뀌면 그것이 해당 컴포넌트의 리렌더링 조건이라는것.
2. `jsx` 컴포넌트로 감싸진 문법을 객체 형태로 바꿔보면 이해가 좀더 수월해진다는것.

---

# 트러블슈팅 21

### 문제:

시작 버튼을 누르고, 이후 새로고침을하면 예상치 못한 concurrent 오류가 발생했다.

### 모색:

`setState()`와 `navigator()`가 같은 스코프 안에 섞여있는 것과, 해당 컴포넌트가 `useNavigate()`를 구독하여 발생하는 렌더링 중복 관련 문제라고 인식했다.

그래서 우선 `navigator()`를 분리하기 위해 유틸리티 컴포넌트로 추출하도록 시도했다.

그러나 해당 컴포넌트를 외부에서 가져오긴했지만, 사용하는 것 자체가 `useNavigate()`를 구독하는 것과 같아서 결과는 그대로였다.

### 원인:

(트러블슈팅 19) 에서 처럼, `setState()`와 `navigator()`가 같은 스코프에 섞였을때 같은 batch로 스케줄링 되지않아서 각각 따로 리렌더링을 발생시키는 것과 해당 컴포넌트가 `useNavigate()`를 구독하여 발생하는 렌더링 중복 관련 문제를 주 원인으로 판단했다.

### 해결:

`setState()`와 `navigator()`를 분리할 수 있도록 리팩토링하여 해결하였다.

예를들어 `setState()`는 `fetchHandler` 안에서 다루고, `navigator()`는 `useLayoutEffect` 안에서 다루도록 재설계하였다.

---

# 트러블슈팅 22

### 문제:

`"/home/error"` 에서 새로고침을 하면 `fetchError` 컴포넌트안에서 `cannot properties` 런타임 오류가 발생했다.

### 모색:

`console.log()`를 세부적으로 적어놓고, 출력된 결과를 참고하여 리액트 앱 렌더링 순서에 대해 재학습 하였다.

### 원인:

원인1: 라우터가 기존 url을 인지하고 있어서 그대로 작동한 것이 원인이었다.

원인2: 해당 path는 이전 path에서 fetch와 state 등의 작업 후 결과 데이터를 받아 props 로 갱신되는 구조로 운영되는 컴포넌트 였어서, 직접 접근하면 빈 값이었던게 원인이었다.

리액트 `npm start`의 개발 모드에서는, 새로고침시 해당 url에 대한 정적자원을 먼저 찾아보고, 없으면 `index.html`을 자동으로 응답해준다.

하지만 응답 후에도 url은 유지되고 있기때문에 라우터가 url을 인지하고 작동하여 이런 오류가 발생했다.

### 해결:

props으로 넘겨주는 해당 state 값에 초기값을 셋팅하여 해결하였다.

그리고 새로고침시, 버튼으로 다시 앱을 가동하도록 하고싶어서

```
<script>
if (location.pathname !== "/") {
  window.location.replace("/");
}
</script>

```

이렇게 루트로 리다이렉트하는 코드를 `index.html`에 추가하였다.

---


# 트러블슈팅23

### 문제:

어떤 path에서 브라우저의 뒤로가기를 눌렀더니, 값이 다시 표시되는것을 예상했지만 `loading` 표시가 나왔다.

### 모색:

뒤로가기 버튼을 눌렀을때 브라우저와 라우터의 작동방식을 재학습했다.

### 원인:

시작 버튼을 눌러야만 특정 path에서, fetch와 state 작업 등이 모두 처리되고, 그 데이터를 다른 path의 element에 props로 전달하는 구조가 원인이었다.

### 해결1:

부모 컴포넌트의 props로만 데이터를 받아 의존했던 구조를, 자식 컴포넌트 자체에서 상태를 관리하는 구조로 바꿔서 해결하도록 리팩토링을 시도했다.

그러나 부모 컴포넌트에서 자식 컴포넌트로 props를 넘겨 표시하는 구조는 그대로 두기로 하였다.

### 해결2:

왜냐하면 현재 구조상 그렇게 해야 `/home/error` 에서 뒤로가기를 눌렀을때, `/create` 에서 뒤로가기를 눌렀을때의 모든 상황이 한번에 처리되기 때문이다.

다음으로 생각해야 할 것은 fetch 실행의 조건을 확장하는 것이었다.

즉 시작 버튼으로만 fetch가 실행 되는게 아니라, path가 `"/home"` 으로 매치될 때에도 fetch가 실행되도록 하는 경우의 수를 생각해야했다.

하지만 자식 컴포넌트(`CalendarHome`) 안에서만 fetch를 해야하는 방법밖에 떠오르지 않았다.

그런데 계속 고민해보니 문제는 새로고침과 뒤로가기였고, 그렇다면 부모컴포넌트가 호출될 때마다 fetch가 실행돼서 상태값을 지니고 있으면 되었다.

왜냐하면 새로고침은 리액트 앱이 다시 실행되므로 최상위 컴포넌트부터 호출되니까 부모 컴포넌트도 호출되고, 그 때 fetch가 실행돼서 데이터를 props로 내려받을것이고, 해당 url이 그대로 라우팅되기 때문에 props도 자식 컴포넌트로 그대로 넘어갈 것이라 문제가 없었다.

뒤로가기는 `[location.key]`를 의존성배열로 설정해서 fetch를 실행하면 문제가 해결되었다.

따라서 부모컴포넌트가 호출될 때마다, fetch로 상태값을 갱신하도록 리팩토링하여 해결하였다.

---

# 트러블슈팅24

### 문제:

리액트의 `fetch()` 메서드에서, HTTP 요청의 content type 을 어떤것으로 설정해야하나 의문이 생겼다.

### 모색:

처음엔 타임리프 기반이었던 프로젝트를, 중간에 리액트 기반으로 변경했고 개발을 진행중이었다.

이때 `fetch`를 `post` 방식으로 처음 사용해봤는데, 타임리프 기반일때는 브라우저에서 사용자에게 Input 값을 입력받아 `submit` 버튼만 눌리면, 브라우저에서 자동으로 HTTP 메시지를 만들어 스프링에 요청이 되었지만 `fetch()` 를 이용해 요청할땐 HTTP 요청 메시지의 header나 content-type 등을 개발자가 직접 지정해줘야 한다는 것을 알게되었다.

이때 Content-type 에 대한 개념에 의문이 생겨 더 깊게 학습하고자 했다.

### 원인:

HTTP 메시지의 Content-type의 종류, 특징, 장단점 등에 대한 의문점이 원인이었다.

### 해결:

HTTP 메시지와 Content-type(JSON, `application/x-www-form-urlencoded`, `multipart/form-data` 등) 개념을 재학습해 의문을 해결하였다.

학습 중에 JSON 방식이 보편적인것은 알고있으나, 그 이유가 궁금해서 더 학습했다.

내가 이해한 이유는 다음과 같았다.

1. 웹페이지는 html로 이루어져있다. html 요소는 자바스크립트로 조작할 수 있다. 자바스크립트 객체 구조 자체가 곧 json 포맷과 같다. 그래서 html 입력값을 바로 변수에 자바스크립트 객체 구조로 할당이 가능한것 등, html과 자바스크립트 객체 구조는(JSON은) 상호 호환이 용이하다.
2. 데이터가 중첩 구조 등 복잡한 경우엔 더욱 json 형식으로 표현하는것이 편리하다.
    
    만약 `application/x-www-form-urlencoded` 을 이용한다면 파싱의 까다로움 등으로 거의 불가능하다.
    
3. 자주 쓰이는 백엔드 스프링에서도 `@RequestBody` 를 지원하여 json 포맷 요청을 Dto로 편리하게 할당할 수 있다.
    
    따라서 결과적으로 JSON 포맷 방식을 사용하기로 하였다.
    

---

# 트러블슈팅25

### 문제:

달력 생성 `create` 페이지에서 저장 버튼으로 POST 요청을 했을때, post fetch를 작동 시키고 navigator를 home으로 보내는 과정을 구성해야했는데, 렌더링 순서가 복잡해 경우의 수 파악이 어려웠다. 즉 `/create` 에서 달력을 저장했을때 `/home` 으로 보내는 완벽한 로직을 찾는데 어려움이 있었다.

### 모색:

우선 스프링 컨트롤러 메서드에 `redirect:home` 을 없앴다. 그 이유는 다음과 같다.

기존 스프링 컨트롤러 메서드에 `/home` 에 대한 `postMapping` 메서드의 return 값에 `redirect:home`이 있었는데, 이것이 있으면 `await fetch`에서 302 응답을 받은뒤(기본적인 설정인 `redirect:follow` 로 인해) 자체적으로 해당 location에 GET요청을 보낸다.

따라서 최종적으로 `await` 앞 변수에 담기는 값은 GET 요청에 따른 `response` 객체가 된다.

이렇게 되면, post fetch의 로직에도 get fetch의 로직과 똑같이 `setState()` 함수들을 구비해야하고 그렇게 되면 중복되는 코드가 많아지고, 현재 구성에 따른 상태 변경도 많아진다.

따라서 우선 `redirect:home`을 없앴다.

### 원인:

복잡한 기존 구조가 주 원인이었다.

즉 `navigator()`로 인한 리렌더링, 그에 따라 의도된 상태변화, `navigator()`와 `setState()`를 분리하고자 하는 의도 등에서 비롯된 다소 복잡한 기존 구조에다가, 새로운 로직을 조립하려 했던것이 것이 주 원인이었다.

### 해결:

따라서 기존 구조를 다시 리팩토링하여 해결하였다.

1. 스프링에서 POST `/home`의 return 값인 `redirect:home`을 없앴다.
2. get fetch와 post fetch의 에러 처리 로직은 똑같았기 때문에 둘을 하나의 fetch handler 안으로 묶었고, 넘겨주는 인자에 json 데이터가 있느냐 없느냐를 `if`문으로 분기해 get fetch 와 post fetch를 분리했다.
3. post fetch는 단순 저장만하고 끝나도 괜찮았기 때문에 `setState()`를 호출하지 않도록 하여 상태변화 복잡도를 없앴다.
4. 개발 의도와 렌더링 순서를 고려하여 구성하였다.
    
    예를들어 post fetch를 비동기로 실행 => `navigator()`를 호출 => `useLayoutEffect`에서 `[location.key]`로 감지해 `loading`을 `false`로 초기화 => 그 이후에 get fetch를 비동기로 실행하도록 하여서 순서에 대한 문제가 발생하지 않도록 리팩토링하였다.
    

---

# 트러블슈팅26

### 문제:

fetch 에러 처리 컴포넌트에서는 에러 객체 props를 받는데, 이때 `props.status` 등의 접근자에서 null오류가 발생할 수 있다는 것을 알게되었다.

### 모색:

fetch 도중 `catch()`에 에러 객체가 잡힐때, 에러 객체의 경우의 수와 객체 구조를 찾아보았다.

### 원인:

fetch 중에 오류가 나면 100~500 코드를 가진 일반 HTTP 오류는 물론 네트워크, CORS오류 등 또한 Response 객체에 담긴다고 생각했던 것이 원인이었다.

네트워크 오류 발생시 `catch()` 에 잡히는 객체는 `status` 또는 `statusText`를 가지지않고, 자바스크립트 표준인 `name`이나 `message`를 가지고있었기때문에 `fetchError` 컴포넌트의 `status`, `statusText` 라는 접근자에서 null 오류가 발생할 수 있었다.

### 해결:

```
catch (resObject) {

  setLoaded(true);
  setError(true);

  // HTTP 오류일때(Response 객체가 할당됨)
  if (resObject instanceof Response) {
    setErrorResInstance(resObject);
  } else { // 네트워크 등 오류일때(Response 객체가 할당되지않음)
    setErrorResInstance({
      status: 0,
      statusText: resObject.name
    })
  }
}

```

위와 같이 `catch(resObject)`이후 `if`문으로 `Response`객체인지 아닌지 분기처리하였고, HTTP오류가 아닐때는 `setErrorResInstance()` 에 직접 커스텀 접근자에 맞게 객체를 만들어 할당하여 해결하였다.

---

# 트러블슈팅27

### 문제:

```
useLayoutEffect(() => {
    if (loaded && error) {
      navigator("/home/error");
    }
  }, [catchFlag])

```

위같은 기존 코드에서 `if`문이 필요하지 않은것같아 없앴더니, 홈으로 돌아가면 주소창이 `/home/error` 로 시작되었다.

### 모색:

`if` 문을 다시 복구해서 실행해봤더니, 다시 원래대로 돌아왔다.

### 원인:

원인은 해당 `useLayoutEffect`는 첫 마운트시 무조건 실행되기때문이었다.

그래서 `if` 문이 없을땐 무조건 `"/home/error"`로 url이 바뀌었던 것이다.

### 해결:

`if (servicesSection.current.style.display === 'block')` 로 조건식을 추가해 해결하였다.

현재 첫 마운트할때는 무조건 `display === none` 인 설정때문에, 방어 코드가 잘 작동해 해결되었다.

---

# 트러블슈팅28

### 문제:

달력을 맨처음 저장한 뒤에 `/home`으로 돌아가면 "달력을 만들어 주세요." 라는 글이 없어야했는데 있었다.

### 모색:

처음엔 props 등이 갱신되지않아서 이전 Fiber 노드를 그대로 재사용해서 업데이트가 안된건가 생각했지만 상태값과 같이 리렌더링되기 때문에 그건 아니라고 판단하였다.

다음은 혹시 저장하는 post fetch가, 조회하는 get fetch보다 늦게 완료됐을 가능성을 따져봤다. 그러나 두 fetch 실행 사이에는 갭이 꽤 있어서 그럴 가능성은 희박하다고 생각했다.

그래도 확인을 위해 스프링 백엔드에서 get, post 메서드에 각각 종료 출력을 표시해보고, 무엇이 먼저 실행되는지 살펴봤다.

충격적으로 아래와 같이 get 이 먼저 출력되었다.

```
Hibernate: select c1_0.id,c1_0.created_at,c1_0.date,c1_0.title,c1_0.updated_at from calendar c1_0
get home 완료
Hibernate: insert into calendar (created_at,date,title,updated_at,id) values (?,?,?,?,default)
post home 완료

```

스프링에서 get 작업이 먼저 완료된것이다.

더 알아보기위해 스프링과 리액트에 시작과 완료 로그를 출력해본 결과 알아낸 사실이 몇가지 있었다.

1. 당연히 코드 순서상은 항상 post fetch가 get fetch 보다 먼저 실행되기는 했다.
2. post fetch가 get fetch 보다 늦게 끝난다. 그러나 이것은 맨 처음 달력을 추가했을때만 그렇다. 이후부터는 정상적으로 post fetch가 더 빨리 완료된다.
3. 스프링에서의 출력은 항상 일관적이었다. 즉 post가 먼저 들어오면 post를 먼저 처리하여 post fetch가 먼저 끝나고, get이 먼저 들어오면 get을 먼저 처리하여 get fetch가 먼저 끝났다.

따라서 스프링 내부적인 수행 시간이라기 보다는, 리액트의 get fetch와 post fetch 중에 누가 더 먼저 스프링에게 요청을 보내느냐의 차이로 보였다.

1. 그러는 와중에 다시 테스트하기 위해 리액트와 스프링 서버를 껐다 켜보면, 갑자기 첫번째 등록임에도 불구하고 잘될 때가 있었다

### 원인:

출력결과와 몇번의 테스트로 모색해본 결과, 첫 fetch시 브라우저 네트워크 캐싱 및 스프링 첫 요청에 대한 초기화 등을 원인으로 규명했다.

### 해결:

리액트 상에서, 명시적으로 `await`을 이용해 순서를 보장하여 해결하였다.

`await post fetch`로 저장이 확실히 진행된 이후 `navigator()`를 호출하도록 하였다.

---

# 트러블슈팅29

### 문제:

스프링을 통해 받은 json 데이터를 `map`으로 분해하는 상황이었다.

즉 json 객체 요소를 `<tbody>`안에서 각각 나열하기위해 `{ 데이터.map() }`을 하였고, 그 `return` 값은 `<tr> <td><td/> ... <tr/>` 이었다.

이때 key값이(id값이) 필요하다고 생각해 `<tr>`이 아닌 각 `<td>` 요소에 key 값을 할당했다.

그런데 `[Keys should be unique so that components maintain thier identity across updates]` 에러가 출력됐고, 그 이유가 의문이었다.

### 모색:

배열의 `map()`에서 key의 역할과 원리를 의문이 생겨 학습하였다.

### 원인:

key의 역할과 장단점, key와 연관된 Fiber 노드 개념이 정립되지 않았던 것이 주원인이었다.

### 해결:

`map()` 콜백의 `return` 값에서 최상위 요소에 key값을 두는것이 정석적인 방법이었다.

아래 개념들을 학습하여 사용 이유를 학습해 의문을 해결하였다.

[리액트의 배열 요소에 key가 있을때] =>

1. React는 `React.Element` 로 객체를 만들고, 해당 객체를 기반으로 Fiber 노드를 만든다. 그리고 다음 렌더링때 새로 만든 `ReactElement` 객체에 key가 있으면 같은 key를 가진 Fiber 노드를 기존 Fiber 트리에서 찾아낸다.
2. 대부분의 경우 같은 key이고 같은 type이라면 새로 만든 `ReactElement`의 정보(props, children 등)를 그 기존 Fiber 노드에 덮어씌운다. 즉, 기존 Fiber를 재사용(reuse) 하면서 업데이트한다. 즉, 컴포넌트는 언마운트/마운트되지 않고, 그대로 유지되고 state, DOM, refs 등이 유지된다.

---

# 트러블슈팅30

### 문제:

타임리프 기반이었던 프로젝트를 리액트 기반으로 바꾸면서 모달 관련 기능 리팩토링을 하고있었다.

이 때 `<tbody>`안에 `calendarResponseDto.id` 라는 값이 필요했는데, 모달창에서는 그 값에 접근할 수가 없었다.

### 모색:

타임리프 기반에서 썼던 방법처럼 자바스크립트 `querySelector`로 요소를 가져온뒤 해당 값을 모달요소로 할당하는 방법을 먼저 생각해봤지만 더 좋은 방법이 있을것 같았다.

### 원인:

모달창 내부의 스코프와 `<tbody>`스코프가 별개였던 것이 원인이다. `calendarResponseDto.id` 값은 `<tbody>` 태그 안에서만 존재했다.

### 해결:

리액트에서는 요소와, 컴포넌트의 변수 할당이 같은 스코프 내에서 이뤄질 수 있었기때문에 해결하였다.

컴포넌트에 `let` 변수를 할당하고, `onClick` 핸들러에 값을 할당하는 로직을 추가해 편리하게 해결할 수 있었다.

---

# 트러블슈팅31

### 문제:

타임리프 기반을 리액트 기반으로 바꾸다가, 수정페이지에서 수정을 할때 `input`의 `placeholder`로 기존 값이 보이도록 설정하고싶었다.

### 모색:

타임리프에서는 Model에 해당 객체를 담아 `th:placeholder` 로 간편하게 표현할 수 있었다.

리액트에서 해당 객체 또는 해당 객체의 `title`이나 `date` 값을 컴포넌트에 전달하면 될것같다.

### 원인:

해당 id의 객체 값이나 객체의 속성 값을 해당 컴포넌트에서 모르는것이 원인이었다.

### 해결:

`navigator( '/...', { state: ... } );` 처럼 navigator props 기능으로 해당 `calendarResponse` 객체 자체를 props로 전달하여 해결하였다.

이렇게하면 해당 라우터로 이동했을때 `location.state` 로 값을 전달받을 수 있었다.

---

# 트러블슈팅32

### 문제:

타임리프 기반의 수정 페이지를 리액트 기반으로 리팩토링 하다가 Form 형태를 없앨 수 있지않을까 의문이었다.

즉 처음엔 `<Form onSubmit = { (e) => e.preventDefault()  } >` 처럼 이벤트 객체를 받은뒤 `e.preventDefault()` 로 브라우저 기본 submit 작동 방식을 막고, 라우팅을 변경하고 데이터를 전달하는 형태 였는데 굳이 submit을 이용할 필요가 있나 싶었다.

### 모색:

목적을 생각해보았다. 목적은 확인버튼을 눌렀을때 url을 이동하고 json 데이터를 전달하는것 뿐이었다.

### 원인:

타임리프 기반은 url 변경과 데이터를 전달하는 기능을 브라우저의 기본적인 submit, action을 이용하고, 리액트 기반은 브라우저의 submit 기능이 필요없고 `navigator`와 `onClick` 핸들러로 진행 가능한 것이 원인이었다.

### 해결:

`<Form>` 태그를 없애고 `<button onClick = {}>` 으로 대신해 원래 문제는 해결할 수 있었다. 그러나 이렇게 하면 이후에 브라우저에서 `required` 기능이 작동안한다는 것을 알게되어 다시 `<Form>` 태그로 복구하였다.

---

# 트러블슈팅33

### 문제:

수정 모달창에서 확인을 누르고 수정 페이지로 넘어갔는데, url에서 `calendarId`에 해당하는 부분이 `undefined`로 감지됐다.

### 모색:

모달창을 열면 해당 달력 객체를 컴포넌트 변수에 할당해주는 `updateBtnClickHandler` 함수와 수정 확인 버튼 클릭 후 `navigate` 해주는 핸들러인 `updateNavigateHandler`에 `console.log` 를 출력해보았다.

### 원인:

```
let calendarResponseDto = {};

const updateBtnClickHandler = (calendarResponseDto) => {
       calendarResponseDto = calendarResponseDto
};

```

이렇게 컴포넌트 변수의 이름과, 매개변수 이름을 똑같이 둔것이 원인이었다.

처음 생각에는 `let` 변수에 매개변수가 대입되어 갱신될 줄 알았는데 지역변수 스코프가 우선시되어서 `let` 변수에는 초기값인 빈 객체 `{}`만 들어있던 것이다.

### 해결:

컴포넌트 변수의 이름을 매개변수의 이름과 다르게 하여 해결하였다.

---

# 트러블슈팅34

### 문제:

수정 모달창에서 확인을 누르고 수정 페이지로 넘어갔는데, 기존의 스크롤이 없어지고, 요소 일부가 화면에서 잘리는 현상이 발생했다.

### 모색:

모달과 관련이 있을거라 생각해 모달을 열고닫아보면서 개발자 콘솔의 element를 비교해보았다.

### 원인:

`body` 요소에서 `overflow: hidden`이 사라지지 않은것이 원인이었다.

`overflow: hidden`은 요소 컨텐츠가 box 사이즈를 넘어가면 잘라버리는 속성이다.

모달창 닫기를 DOM으로 직접 조작하다보니, 해당 속성이 없어지지않고 유지된것이 원인이었다.

### 해결:

모달창의 확인 버튼에도 `data-bs-dismiss="modal"` 속성을 추가해서, 공식적으로 안전하게 모달 이펙트가 제거되도록 만들어 해결하였다.

---

# 트러블슈팅35

### 문제:

이제 어느정도 라우팅 path 배치가 완료되어서 다시 모든 경우에 스크롤을 특정 요소로 향할 수 있도록 `scrollIntoView`로 자동스크롤을 적용했으나, 때때로 UX가 부드럽지않은 문제가 발생했다.

예를들어 `/home` 으로 가면 `title`로 자동스크롤을 하도록 만들었는데, 뒤로가기를 하면 `/home` 인데도 불구하고 `title`이 아니라 바로 이전에 있던 위치로만 자동스크롤이 되었다.

또한 같은 `/home`에서 앵커가 작동하면 앵커로 스크롤 되어야하는데, 스크롤이 충돌하는듯 보이고 앵커의 위치는 예상보다 아래로 밀렸다.

### 모색:

자동스크롤을 트리거하는 `useLayoutEffect`를 살펴보고, 뒤로가기시에 이전 스크롤 위치를 기억하여 자동 스크롤하는 브라우저의 기본 기능을 떠올렸다.

### 원인:

첫째는 뒤로가기시에 이전 스크롤 위치를 기억하여 자동 스크롤하는 브라우저의 기본 기능이 원인이었다.

둘째는 `[location.key]` 가 바뀔때 `get fetch`와 자동스크롤이 트리거 되는데, 이때 앵커로인해 바뀐 경우에도 트리거가 되는 것이 원인이었다.

즉 앵커가 작동하면 기본적으로 스크롤이 앵커 요소로 이동하는데, `[location.key]`도 바뀌게 되면서 `get fetch`가 작동한다.

이때 컴포넌트가 언마운트->재마운트 되는데 이 과정에서 다른 컴포넌트로 잠시 바뀌므로 요소의 height가 줄어들고 이에 따라 앵커 스크롤이 예상과 다르게 보인 것이다.

또한 새로 만든 트리거인 `useEffect()` 도 `[location.key]` 변경으로 작동하는데, 앵커로 인한 변화까지 감지해버려서 충돌이 일어난 것도 원인이었다.

### 해결:

`App.js`에

```
window.history.scrollRestoration = 'manual';

```

를 추가하여 해결하였다.

이전 스크롤 위치를 기억하여 자동 스크롤하는 브라우저의 기본 기능을 `'manual'`로 바꾼것이다.

그리고 충돌을 일으켰던 함수 조건문에

```
!window.location.href.includes("#")

```

를 추가하여 앵커에 의한 변화는 감지되지 않도록 만들어 해결하였다.

---

# 트러블슈팅36

### 문제:

현재 로직상으로 delete fetch를 목적으로 `fetchHandler`를 실행시키면, 필요없는 `requestData` 객체 까지 넘겨야하는 문제가 발생했다.

즉 리액트에서 요청할때도, 스프링 컨트롤러에서 요청을 받을때도 불필요한 객체를 억지로 넘겨야했다.

### 모색:

`fetchHandler`에서 인자를 넘겨줄때 Create, Read, Update, Delete 에 따라 확실히 분기할 수 있는 방법을 생각해봤다.

### 원인:

기존 로직의 분기가 원인이었다.

기존 로직에서는 `fetchHandler`에 넘겨주는 인자에 `requestData`가 없으면(undefined로 false면) 자동으로 GET으로 이어지고, `requestData`가 있으면(true면) 자동으로 POST로 이어졌다.

즉

```
if (!requestData) { ... } // GET
if (requestData) { ... }  // POST

```

그러나 delete 로직은 `if (!requestData)` 에 해당하면서도 POST로 이어져야했는데 이러한 분기가 구현되지 않은 상태였다.

### 해결:

fetch를 실행할때 넘겨주는 인자인 `pathname` 자체를 `if`문 분기로 리팩토링하여 해결하였다.

즉 기존에는 `requestData`로만 `if`문 분기를 했었지만 리팩토링 후 아래 코드처럼 `pathname`을 분기로 활용하였다.

```
if (pathname === "/home" && !requestData)
else if ( (pathname === "/home" || pathname.startsWith("/calendar/update/")) &&
        requestData)
else if (pathname.startsWith("/calendar/delete/"))

```

---

# 트러블슈팅 37

### 문제:

Calendar Item 컴포넌트를 구성하는 상황에서, 기존 fetch Handler 와 로직을 그대로 두고 구현하려했다.

이때 Calendar Item 컴포넌트의 url path는  `"/calendar/:calendarId/item"` 형태 였다.

그리고 해당 컴포넌트 외부에 위치한 `useLayoutEffect()` 에서 `if( location.pathname === url )`조건식에 사용할 확정된 `url`이 필요한 상황이었다.

그러나 `":calendarId"` 이부분은 동적파라미터이기 때문에 `"location.pathname === url"` 방식으로 판별하는것이 불가능했다.

### 모색:

`useMatch()` 라는 리액트훅을 사용하면 현재 pathname이 설정한 패턴과 일치하는지 확인 가능했다.

그러나 리액트훅은 `useLayoutEffect()` 에서 사용할 수 없었다.

### 원인:

동적 파라미터를 가진 url을 라우터 외부에서 `===` 으로(확정된 값으로) 비교하려던 것이 원인이다.

### 해결:

```
const regExp = /^\/calendar\/\d+\/item$/;

```

처럼 정규표현식을 활용해 해결했다.

`"/calendar/:calendarId/item"` 이런 형식의 url 패턴을 정규표현식 객체에 설정해두고, `.test()` 를 조건식에 이용하여 해결하였다.

---



# 트러블슈팅 38

### 문제:

이전 트러블 슈팅으로 useLayoutEffect()에서 정규표현식을 이용한 분기로 원하는 로직이 작동하도록 만들었다.

그런데 이후 fetchHandler() 에 도달하면 또 다시 url에 따른 분기를 만들어내야했다.

그렇게 되면 fetchHandler() 내에서 또 정규표현식 객체를 만들거나(중복 수행),

매개변수로 객체를 전달해야한다.(때에 따라 불필요한 매개변수 전달)

또한 if문 조건식이 더 복잡해지거나 중복 코드가 늘어나게된다.

fetchHandler의 리팩토링 필요성을 느꼈다.

### 모색:

이리저리 고쳐보다가, 굳이 pathname으로만 분기할 필요가 없다는 생각이 들었다.

사실 분기하는 이유는 fetch 로직의 차이였다.

fetch 로직의 차이는 그냥 서버의 응답만 얻기위함이냐,

body 없는 POST냐,

body가 있는 POST냐 등의 차이였다.

그 차이는 곧 GET, POST, DELETE 동작의 차이였다.

따라서 http Method명으로 분기하면 될것같다는 생각이 들었다.

### 원인:

fetchHandler() if문 분기를 pathname으로 설정한 것이 원인이었다.

같은 코드를 재활용할수 있으면서도, 더 쉽게 분기할 수 있는 방법이 있었다.

### 해결:

fetch Handler 매개변수 구성을 바꾸고,

if문 분기 조건식을 httpMethod에 따라 분기되도록 바꾸고,

스프링의 컨트롤러 메서드 어노테이션을 HTTP 메서드형식으로 바꿔서 해결하였다.

예를들면

fetchHandler(pathname, requestData) 를

fetchHandler(pathname, httpMethod, requestData)로 바꾸고,

if (pathname === "/home" && !requestData) 이런 방식에서

if (httpMethod === "GET" ) 같은 방식으로 바꾸고,

스프링 컨트롤러를

@GetMapping과 @PostMapping 밖에 없던 구조에서

@PutMapping, @DeleteMapping 을 추가한 구조로 바꿔 해결하였다.

---

# 트러블슈팅 39

### 문제:

달력의 세부 아이템을 보여주는 컴포넌트를 구현하고 있었다.

이때 아이템 값을 서버에서 fetch로 받아올때, 모든 값을 공용 fetch Handler 로 받아오기 때문에

별도 로직 변경없이 사용할 수 있다.

그런데 아이템 응답 객체 값에 접근하려하니 undefined 런타임 에러가 발생했다.

### 모색:

컴포넌트가 렌더링되는 순간부터 순서에 따라 차근차근 살펴본뒤 원인을 파악했다.

### 원인:

모든 fetch의 응답 값은 fetchData 라는 상태값에 담아두는데,

이 fetchData에 담기는 객체 구조가 도메인별로 달랐고,

각 컴포넌트마다 fetchData에 접근하는 속성이 달랐던 것이 원인이었다.

그래서

직전path와 다른 path로 라우팅이 되어 다른 컴포넌트로 렌더링 될때,

기존 fetchData { } 속성엔 없는 다른 속성으로 접근하려할때 문제가 생겼다.

예를들어

달력 응답이 담길땐 fetchData { }의 내부 속성 이름은 calendarList 인데,

아이템 응답이 담길땐 fetchData { }의 내부 속성 이름은 calendar 이고

그 이외 속성의 종류나 개수도 달랐다.

그러다 보니

새 컴포넌트의 reutrn의 jsx값에 {calendar.id} 가 있는데, 직전에 저장되어있는 fetchData 에는 'calendar' 라는 속성 값이 없었고

그렇다면 Render phase에서 Commit phase단계로 가기 직전에 undefined 런타임 에러가 발생하는 것이다.

물론 fetch 완료후 상태값이 업데이트된 다음에는 각 컴포넌트에 맞는 객체 구조로 완성되기때문에 문제가 없지만,

fetch 전에 한번 일시적으로 컴포넌트를 호출하는 과정에서만 위 문제가 일어났다.

### 해결:

해결1 =>

위 문제 자체의 해결은 쉬웠다.

jsx의 return문에 간단한 조건부 렌더링 또는 if문을 통해 코드 한줄 정도의 방어 코드만 넣어주면 되었다.

왜냐하면 fetch 전에 한번 일시적으로 컴포넌트를 호출하는 과정에서만 문제가 발생하기 때문이었다.

예를들면

```
if (calendarResponseDto) {
        return ( .. )

```

이렇게 업데이트 전 렌더링이될때, return 내부에서 undefined 속성 접근 방지를 위해 방어 if문을 추가하여 해결하였다.

이것은

`const calendarResponseDto = fetchData.data.calendarResponseDto;`

위처럼 fetchData와 data가 객체일때 calendarResponseDto라는 속성이 없다해도 오류가 나지않고 undefined 를 반환하는 성질과 같이 활용하였다.

해결2 =>

그러나 위 문제와는 별개로,

스프링 백엔드의 http 응답을 일관적인 포맷으로 만들고 싶었다.

따라서 스프링에서는 ResponseEntity와 커스텀 ResponseData 객체를 활용해 응답 메시지를 구성하도록 변경했고,

리액트에서는 fetchData에 접근하는 로직을 변경하여 해결하였다.

---

# 트러블슈팅 40

### 문제:

이전 문제를 트러블 슈팅을 하면서 console.log() 를 출력해봤는데,

console.log()에서 컴포넌트 실행이 2번 출력된 것이 의문이었다.(Strict Mode와 관계가 없었다.)

만약 return 값 jsx의 undefined로 인한 문제였다면 처음 가상 DOM 생성 후 Commint 단계에서 런타임 오류가 나야했다

따라서 console.log()에 1번만 출력되고 바로 런타임 오류가 나야했지만, 콘솔은 2번이 출력된 것이 의문이었다.

### 모색:

렌더링 순서 체크, 혹시 모르는 state 변화 체크, 컴포넌트의 console.log() 를 출력, 인터넷 검색 등을 하며 이유를 탐색해보았다.

### 원인:

Concurrent Rendering의 특성이 원인이었다.

즉 React 18부터 ReactDOM.createRoot() 사용 시 createRoot()는 Concurrent Rendering을 기본으로 활성화한다고 한다.

Concurrent Rendering의 특성은 렌더링을 "사전 시뮬레이션"처럼 수행할 수 있어서

첫 번째 렌더링 결과를 커밋하지 않고 버리고 다시 실행할 수 있다.

만약 첫 번째 렌더에서 오류가 발생하면

더 나은 렌더링 결과를 만들기 위해 첫번째 Render Phase Fiber 노드를 버리고 다시 시도하는 경우가 있었기 때문에

console.log가 두 번 출력되었던 것이다.

2번째에도 undefined를 반환하면 에러가 발생한다.

### 해결:

Concurrent Rendering의 특성을 학습해 의문을 해결하였다.

---

# 트러블슈팅 41

### 문제:

스프링 백엔드의 return 값을 ResponseEntity객체로 바꾸도록 테스트하는 도중 의문이 생겼다.

스프링에서 임의로 상태코드를 150으로 적은뒤 return 하였고,

예상 결과는 fetch Response객체에 150코드가 담기는 것이었다.

그러나 실제로는 Response객체에 500 코드가 담겼다.

이런 현상이 의문이었다.

### 모색:

스프링의 출력창을 확인해보았지만 오류 로그없이 깨끗했다.

리액트에서 fetch로 Response 객체를 받았다는 것은 스프링의 응답은 정상적으로 받았다는 것인데

어떻게 서버 내부 오류일때 발생하는 500 코드가 나올 수 있는지 의문이었고 조금더 검색해보았다.

### 원인:

서블릿 컨테이너(Tomcat) 단계에서 비정상적인 상태코드로 감지하고 500 응답을 만들어 내보내준 것이 원인이었다.

Tomcat은 RFC 9110에 정의되지 않은 코드(100~599 범위 밖이거나 미등록 코드)를 감지하면 내부적으로 500 Internal Server Error로 교체한다고 한다.

즉 스프링 자체에서는 150 상태 코드를 ResponseEntity에 담아 설정만 하기때문에 정상적인 과정이라 인식해서 오류 로그가 나지않았지만

외부 서블릿 컨테이너(Tomcat) 컨테이너 단계에서, 이런 상태코드를 비정상적인 코드라고 감지하여 HTTP 응답 메시지 코드를 500 Internal Server Error로 교체해 내보낸것이다.

따라서 서버 내부 오류는 없어보이는데 500 응답이 받아진 현상이 설명되었다.

### 해결:

스프링 컨테이너의 작동 방식을 학습해 의문을 해결하였다.

---

# 트러블슈팅 42

### 문제:

리액트에서 `const res = await fetch()`를 했을때,

만약 스프링 서버가 꺼져있다면 원칙상 `res`에 Response 객체가 할당되면 안된다.

왜냐하면 Response 객체는 서버에서 응답이 도착했을때만 생기는 객체이기 때문이다.

그러나

```
 if (!res.ok) {
    throw res;
}

```

이 코드가 작동해서 `catch`문에 `res` 객체가 잡혀버렸다.

그 말은 서버가 꺼져서 fetch 응답을 아예 못받았음에도 Reponse 객체가 할당돼버렸다는 것이다.

이런 현상이 의문이었다.

### 모색:

proxy 설정과 관련이 있을것같아서 탐색해보았다.

### 원인:

React 개발 서버에서 `"proxy": "http://localhost:8080"` 를 설정하면,

서버 응답이 없을때 http 응답을 대신 만들어내는 것이 원인이었다.

즉 CRA, Vite 개발 환경에서 proxy 설정은 하면 React 개발 서버가 프록시 서버 역할을 수행한다.

그리고 fetch 요청을 하면 Spring 백엔드로 전달한다.

만약 백엔드가 꺼져 있으면 리액트 개발 서버가 자체적으로 HTTP 500 같은 응답메시지를 자동으로 반환해주는 것이다.

따라서 Response 객체가 생성된 것이다.

혹시나 배포 환경에서도 이렇게 작동할까봐 염려했지만 괜찮았다.

왜냐하면 React 빌드 결과물이 Spring이나 Nginx 같은 서버에 정적 파일로 제공되고,

모든 API 요청은 직접 백엔드(Spring) 로 가기 때문에

백엔드가 꺼져 있으면 네트워크 레벨 오류 (ECONNREFUSED)가 발생하기 때문이었다.

이런 경우 fetch에 따른 Response 객체는 없으며 예상대로 TypeError를 던지게 되어 문제가 없다.

### 해결:

proxy 설정시 React 개발 서버의 작동 방법을 학습해 의문을 해결하였다.

---

# 트러블슈팅 43

### 문제:

기능 구현 중에 `"pathname === ..."` 처럼 동적 url 패턴을 확인할 방법이 다시 필요해졌다.

그런데 이전에 생성했던 정규 표현식 변수는, 성능을 좋게하기위해 지역 스코프로 만들어둬서 재사용할 수 없었다.

### 모색:

다시 정규표현식을 생성하려했지만 중복되는 것같았고,

전역스코프에 두려고 했지만 불필요할때도 렌더링될때 매번 새로 생성되기때문에 다른 방법을 찾아보았다.

### 원인:

정규표현식 객체를 생성해둔 스코프가 다르거나, 생성이 중복되는 문제가 원인이었다.

### 해결:

컴포넌트 최상위에서 `useMemo()` 활용해 해결하였다.

즉 여러가지 정규표현식을 담은 객체{ }를 `useMemo()`로 마운트시 한번만 만들어두는 방법으로 해결하였다.

---

# 트러블슈팅 44

### 문제:

우연히 CalendarItemCreate 페이지에서 앵커태그를 눌렀는데 calendarResponseDto가 null 이라며 런타임에러가 발생했다.

### 모색:

리렌더링 순서를 살펴봤다.

### 원인:

해당 컴포넌트에서 `calendarResponseDto` 객체는 상태 값으로 존재한 것이 아니었다.

이전 페이지에서 `navigator state` 기능으로 일시적으로 넘겨준 값이라는게 원인이었다.

그래서 그 페이지 자체가 리렌더링되고 가상DOM을 만들면서 null 값에 접근하려하니 런타임 오류가 발생한 것이다.

### 해결:

해결1=>

첫 마운트가 될때 아예 상태값으로 가질수있도록 아래처럼 `location.state...` 값을 초기값으로 상태값을 할당하였다.

`const [calendarResponseDto] = useState(location.state.calendarResponseDto);`

그러나 그래도 오류가 났다. 이유를 살펴보니, 리렌더링 시에 변수에는 상태 값이 잘 들어있겠지만 없는 state에 접근하는 코드 자체가 적혀있었기때문에 여전히 오류가 났다.

해결2=>

옵셔널 체이닝 문법으로 해결하였다.

`const [calendarResponseDto] = useState(location.state?.calendarResponseDto);`

이렇게 하면 접근하는 코드 자체가 방어된다.

undefined를 반환하긴하겠지만 그것은 첫 마운트시 초기값일때에만 해당되어서 문제가 없었다.

즉 앵커 기능 작동시 재마운트가 아니라 그저 리렌더링일 뿐이기 때문에, 기존 Fiber 노드가 폐기되지않고 재사용되는 것이고 따라서 상태 값도 기존값이 보존되어 객체 참조도 원활했다.

---

# 트러블슈팅 45

### 문제:

이전 트러블 슈팅을 해결했지만 간과한것이 있었다. 바로 뒤로가기 였다.

그런데 예상과 다르게, 뒤로가기의 경우에 에러가 나지않았어서 눈치채지 못했다.

왜 작동이 잘됐던건지 의문이었다.

### 모색:

앵커 태그 문제를 해결할땐 마운트가 아닌 리렌더링이기 때문에 상태값으로 유지하도록하여 해결이 됐다.

그러나 뒤로가기 시에는 언마운트 => 재마운트 라서 상태값은 아예 초기값으로 돌아가버리고, `location.state` 값은 비어있기때문에 에러가 나야한다고 생각했다.

브라우저의 뒤로가기에 대해서 학습해보았다.

### 원인:

잘 됐던 원인은 브라우저의 히스토리 스택에서 그 시점의 state 값도 같이 저장해두었기 때문이다.

navigator의 state 값은 브라우저 History API 내부 메모리에 저장되고,

뒤로가기 발생 시 브라우저는 History API에 저장된 URL과 state 객체를 React Router에 전달하는 것이다.

### 해결:

브라우저의 히스토리 스택에 대해 학습해 의문을 해결하였다.

---

# 트러블슈팅 46

### 문제:

뒤로가기가 섞여있는 상황일때 다시 문제가 발생했다.

임의의 페이지에서 버튼으로 #해쉬가 생기고, 다른 path로 갔다가 다시 뒤로가기를 했을때 런타임 에러가 발생하거나 데이터가 제대로 표시되지 않았다.

### 모색:

차근차근 렌더링 순서를 살펴보았다.

### 원인:

근본적인 원인은 데이터를 상태값으로 저장해두지 않고 props나 location.state에 의존하던 것이다.

예를들어

#해쉬가 붙은 상태로 라우팅이 돼버려서 새 데이터를 위한 fetch() 가 트리거되지 않거나,

#해쉬가 붙은 상태로 history 스택의 state 값을 찾으려할때 undefined 인 경우들이 표면적인 원인이었고

근본 원인은 상태값으로 값을 갖고있지 않았던 것이다.

### 해결:

Recoil 라이브러리로 전역 상태 관리를 두어 해결하였다.

(현재 리액트 19와는 호환되지 않는다고하는 문제가있어 리액트 18버전으로 다운그레이드하였다.)

즉 이 문제는 컴포넌트가 언마운트가 되더라도 상태값이 유지되면 해결될 문제여서,

처음엔 Context를 활용하려했지만 타겟은 부모 자식관계가아니라 형제 관계인 점 등의 복잡한 상황이 있었다.

따라서 해결 방법을 더 찾아보니 Recoil 을 이용하면 전역적으로 상태 관리를 할수있었고,

해당 atom 단위로만 리렌더링이 되어서 최적화가 가능했기 때문에 Recoil을 사용하였다.

---

# 트러블슈팅 47

### 문제:

뒤로가기시 런타임에러는 해결했지만

url에 나오는 id와, 실제 paint되는 데이터의 id값이 다를 경우의 수가 있었다.

그 경우 사용자에게 혼란을 줄수있는 문제가 발생했다.

### 모색:

모든 history 스택에 pushState로 데이터를 저장하여 모든 url과 맞는 객체로 동기화하도록 하려했으나 과정이 복잡하고 비효율적이었다.

만약 그렇게 한다고해도 데이터를 삭제한뒤 뒤로가기로 이전 url에 도달했을때 값이 표시가 되어버려 혼란을 줄 경우의 수도 있었다.

### 원인:

전역 상태값인 Recoil은 가장 최근에 fetch한 상태값을 유지하고 있었기때문에,

뒤로가기시 history 스택에 따라 url이 변경되더라도 paint는 최신 데이터만 그려지고있던 것이 원인이었다.

### 해결:

해당 컴포넌트가 렌더링될때마다 별도로 데이터를 fetch 해오는 구조로 바꾸어 해결하였다.

Recoil을 없애고 fetchHandler와 각 컴포넌트의 상태값, 조건문 렌더링, useLayoutEffect 의 구조를 개편하였다.

이렇게하면 정상적인 로직일때도, 다양한 id의 url로 뒤로가기할 때도, 삭제된 값으로 뒤로가기할때에도 항상 최신값을 보장하기때문에

접근이 안전하였고 사용자 혼란도 없앨 수 있었다.

---

# 트러블슈팅 48

### 문제:

useLayoutEffect 내부에서 async 익명 함수를 사용하려했는데 런타임 에러가 발생했다.

### 모색:

런타임 에러를 검색해보았다. promise 객체와 관련이 있는듯했다.

### 원인:

useLayoutEffect나 useEffect 함수 내부에서 `return`을 하게되면 리액트는 그것을 clean up 함수로 인식하고, 이는 동기 함수만 가능한 규칙이 원인이었다.

즉 자바스크립트 엔진이 async 비동기 함수를 만나면 바로 promise 객체를 반환하는데,

그 반환이 effect 함수 내부에서 발생하여 리액트는 clean up 함수로 여기고 따라서 동기 함수여야한다는 에러가 발생한 것이다.

### 해결:

useLayoutEffect 콜백 자체에서는 함수 정의만하고 어떤 값도 return 하지 않도록 만들어 해결하였다.

---

# 트러블슈팅 49

### 문제:

스프링 시큐리티를 활용해 Oauth 구글 로그인을 구현하는 상황이었다.

리액트 fetch로 /api 요청 후 로그인 로직이 실행됐을때, 인증되지 않은 상태면 CORS 오류가 발생했다.

### 모색:

요청과 응답의 흐름을 하나하나씩 단계별로 따라가면서 분석해보았다.

### 원인:

브라우저가 자바스크립트에 대해 기본 작동시키는 CORS 보안 정책 때문이었다.

CORS 검사는 브라우저에서 JS로 네트워크를 호출할 때 이뤄진다.

즉 자바스크립트의 `fetch()` 를 사용할때, 요청 Origin(도메인)과 서버의 Origin이 다르다면 기본적으로 CORS 정책 위반으로 통신을 막기 때문이었다.

예를들어 스프링 시큐리티에서는 인증이 안됐할때 기본적으로 `anyRequest().authenticated()` 을 사용하고, 이로 인해 허용한 요청 이외엔 전부 인증 과정을 거친다.

이때 리액트에서 fetch 로 요청이 됐다면 CORS 검사가 진행된다. 그 요청은 스프링 컨트롤러에 도달하기전에 시큐리티 필터가 가로채고 인증 로직을 시작한다.

이때 시큐리티는 기본적으로 구글 로그인 페이지로 리다이렉트시키며 액세스 토큰을 주고받는 통신들이 시작된다.

이 과정에서 개발 Origin(localhost)와 구글 서버 Origin(google.com)은 서로 다른 Origin 이며,

localhost라는 Origin에 대해 구글 서버에서 CORS 허용을 해놓지 않았고, 브라우저가 그것을 감지해 오류가 난것이다.

### 해결:

리액트에서 OAuth 인증이 `fetch()`로 부터 시작하지않고 `href = "http://localhost:8080/oauth2/authorization/google"` 처럼

브라우저 자체 네비게이션을 통해 이동하도록 하여 문제를 해결하였다.

---

# 트러블슈팅 50

### 문제:

인증 실패의 경우 401 코드를 받아 리액트에서 공통적인 에러 처리가 가능하도록 추가로 구성해야했다.

### 모색:

`anyRequest().authenticated()`로 인한 인증 과정을 좀더 상세히 알아보았다.

### 원인:

인증이 실패했을때 작동하는 시큐리티의 기본 로직(리다이렉트)이 근본적인 원인이었다.

### 해결:

`public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint { ... }` 를 통해 커스터마이징하여 해결하였다.

```java
@Override
public void commence(HttpServletRequest request,
                     HttpServletResponse response,
                     AuthenticationException authException)
        throws IOException, ServletException {

    HttpStatus status = HttpStatus.UNAUTHORIZED;

    // 기본 에러 응답 포맷 구성
    Map<String, Object> errorBody = new LinkedHashMap<>();
    errorBody.put("timestamp", ZonedDateTime.now().format(DateTimeFormatter.ISO_OFFSET_DATE_TIME));
    errorBody.put("status", status.value());
    errorBody.put("error", status.getReasonPhrase());
    errorBody.put("message", "로그인이 필요합니다.");
    errorBody.put("path", request.getRequestURI());

    // 응답 설정
    response.setStatus(status.value());
    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
    response.setCharacterEncoding("UTF-8");

    objectMapper.writeValue(response.getOutputStream(), errorBody);
}

```

위와 같이 `CustomAuthenticationEntryPoint` 객체 내부의 `commence` 메서드를 오버라이딩해,

인증 실패시 401 JSON 기본 에러 객체 구조로 통일된 http 응답 메시지를 보내주도록 만들어 해결하였다.

---

# 트러블슈팅 51

### 문제:

리액트에서 현재 로그인 상태를 알고싶었지만,

매번 렌더링할때마다 fetch()로 백엔드 인증 로직이 작동하도록 하는것은 비효율적이라는 생각이 들었다.

### 모색:

한번만 인증 상태를 체크하고, 그 인증 상태를 생명주기동안 유지하는 방법을 찾아보았다.

### 원인:

일반 `useState()` 상태 값을 사용할때는 라우팅, 뒤로가기 등으로 언마운트되어 상태가 초기화되는 것이 원인이었다.

### 해결:

상위 컴포넌트의 `useLayoutEffect`로 해당 /api 요청으로 인증 상태를 알 수 있도록 만들고, Recoil을 활용하였다.

즉 스프링에 현재 인증이 되어있는지 확인하는 컨트롤러를 만들고,

리액트에서는 응답 값에 따른 데이터를 Recoil에 담아 인증 상태를 전역적으로 관리하도록하여 해결하였다.


---

# 트러블 슈팅 52

### 문제:

오아스 로그인 진행시 팝업이 따로 뜨게하고, 그 안에서 로그인을 진행하도록 구현했다.

그런데 로그인 완료 후 팝업이 닫히면, 기존 웹페이지 이미지가 깨지는 현상이 발생했다.

### 모색:

브라우저 개발자 도구 등에서 이미지가 잘 불러와지는지 등을 확인해보았는데 그건 문제가 없었다.

### 원인:

모달 조작 방식이 DOM 요소를 추출한후 new bootstrap.Modal( ) 로 부트스트랩 모달 인스턴스를 만들어서

show()와 hide() 메서드로 조작하는 방식이었는데, 이때 여러 인스턴스가 생겨 모달의 css 효과가 충돌한것이 원인이었다.

### 해결:

팝업이 닫히고, 모달을 따로 조작하기보다 부모 웹페이지를 새로고침 하도록 처리하였고, UX상으로도 자연스러워 이 방식으로 해결하였다.

---

# 트러블 슈팅 53

### 문제:

에러 발생시 클라이언트의 /home/error 로 navigate 되어 처리하도록 구현되어 있었다.

그런데 종종 에러는 발생했지만 /home/error로 navigate되지 않아서 무한 로딩이 렌더링 되는 의도치않은 현상이 발생했다.

### 모색:

문제가 발생하는 조건의 상황을 재현하여 작동 순서를 파악해보았지만 별다른 문제가 없어보였다.

### 원인:

검색 결과 결국 개발의 Strict 모드 등으로 에러가 빠르게 연속 2번 발생하여 생긴 문제였다.

즉 에러가 발생했을때 트리거 코드가 setCatchFlag((prev) => !prev) 처럼 작성되어있었는데,

strict 모드로 인해 트리거가 빠르게 2번 이뤄지다보니 같은 값으로 인식되어 [catchFlag] 의존성 배열이 작동안한 것이 가장 가능성 큰 원인이었다.

### 해결:

setCatchFlag 값을 setCatchFlag(() => Date.now()) 처럼 바꿔서 항상 다른 값으로 인식되도록 만들어 해결되었다.

---

# 트러블 슈팅 54

### 문제:

401 에러 발생시 모달창이 열리도록 useRef 를 활용해 구현중이었다.

그런데 모달창 닫기 버튼을 누르면 부트스트랩 모달의 기본 효과인 까만배경 backdrop 효과가 지워지지 않는 현상이 발생했다.

### 모색:

모달 인스턴스와 관련된 검색을 해봤다.

### 원인:

fetchError 컴포넌트에 전달되어 열리는 모달 인스턴스 개수가 여러개인 것이 원인이었다.

모달 효과들이 섞여 충돌한것이었다.

### 해결:

if (finalErrorObject.errorStatus === 401) {

```
        if (!loginModalInstance.current) {
            loginModalInstance.current = new window.bootstrap.Modal(loginModal.current);
        }
        loginModalInstance.current.show();
    }

```

상위 컴포넌트에서 useRef를 정의하며 모달 인스턴스를 1개로 유지해 안정적으로 작동하도록 해결하였다.

---

# 트러블 슈팅 55

### 문제:

오아스 로그인 구현중 provider를 구글, 카카오, 네이버 3가지 종류로 하도록 만들고 있었다.

그런데 회원가입을 할때는 DB의 Member 테이블로 통합하도록 되어있었다.

이때 해당 오아스 Member의 고유한 식별자를 추출해야했다.

따라서 각 provider 미다 고유 식별자인 provider id를(구글의 'sub' 등을) unique 제약으로 설정하려했는데,

이것은 해당 provider에서만(구글에서만) 유일한 것이지 다른 provider(카카오, 네이버 등) 에서 유일한 식별자는 아니었다.

따라서 unique 설정 컬럼을 정해야했다.

### 모색:

오아스 provider의 unique 설정 관련 문제를 찾아보았다.

### 원인:

provider 식별자는 각 provider 내에서만 유일한 식별자였고, 모두를 통합하는 식별자가 아니었던것이 원인이다.

### 해결:

@Table(

uniqueConstraints = @UniqueConstraint(columnNames = {"provider", "providerId"})

)

스프링 Member 엔티티에 provider와 provider id 컬럼 2가지를 복합유니크로 설정하여 해결되었다.

---

# 트러블 슈팅 56

### 문제:

MySQL을 사용하기위한 DB 클라이언트로 디비버(DBeaver)프로그램을 사용중이었다.

그리고 스프링 앱을 시작하고나서, 디비버로 MySQL 테이블을 select로 조회해봤지만

이상하게도 DB에 테이블이 생성되지 않았다. 테이블 자동 생성인 ddl-auto 설정이 create임에도 불구하고 sql 로그도 찍히지않았다.

### 모색:

우선 DB관련 설정이 되어있는 스프링 yml 파일을 살펴보았다.

sql:

init:

mode: always

위 설정과

defer-datasource-initialization: true

위 설정을 확인해보았지만 이것들은 이 문제와 관련이 없었다.

그런데 디비버에서 우연히 커밋버튼을 눌러보니 그제서야 테이블이 보였다.

원인:

### 원인1:

디비버의 커밋 설정이 오토커밋이아니라 수동커밋이었던 것이 원인이었다.

### 원인2:

MySQL 엔진의 기본 격리 단계가 "REPEATABLE READ" 단계인것이 원인이었다.

"REPEATABLE READ" 단계에서는, 동일한 트랜잭션이 시작됐을때 그 시점의 데이터 스냅샷을 유지하는것이 기본이다.

즉 같은 트랜잭션 내에서 Select 결과가 항상 동일하도록 설계되어있다.

또한 디비버와 스프링은 같은 DB에 대해 별개의 클라이언트이기 때문에 각자 별개의 세션으로 작동한다.

이때 스프링 입장에서는 DB에 테이블을 생성했지만,

디비버 입장에서는 DB에 대해 트랜잭션이 시작됐을 시점의 스냅샷을(테이블이 없던 초기 상태를) 유지하고 있었기 때문에 테이블이 보이지 않았던것이다.

또한 만약 어떤 세션에서 DML, DDL을 시도하려하는데,

다른 세션에서 같은 테이블에 대한 트랜젝션이 감지되면 기본적으로 데이터 안전을 위해 lock이 걸린다.

이때는 그 lock이 풀릴때까지 해당 세션은 멈춰서 대기하는 경우도 생긴다.

### 해결:

"REPEATABLE READ" 단계와 세션, 트랜잭션 스냅샷에 대한 개념을 숙지하고,

디비버 커밋 설정을 오토커밋으로 바꿔 해결하였다.

---

# 트러블 슈팅 57

### 문제:

달력과(Calendar와) 회원(Memer) 엔티티를 DB 연관관계로 설정하고,

Calendar 컨트롤러를 리팩토링하여 해당 회원만 달력 조회가 가능하도록 만들었다.

그리고 달력의 세부아이템(CalendarItem)에는 굳이 회원과(Memer와) 직접적인 연관관계를 맺지 않아도 된다고 생각했다.

왜냐하면 해당 회원만 해당 달력이 조회되고, 그것을 클릭해야 세부항목으로 넘어가기 때문이었다.

그러나 조금더 생각해보니 url로 직접 접근하는 경우는 예외였다.

### 모색:

리액트 라우터 path를 다른 방식으로 바꾸는 방법 등을 고려했다.

그러나 단순히 path만 바꾼다고해도, 포스트맨이라는 도구를 이용하면 의도치않은 저장 및 조회가 발생할 수 있었다.

### 원인:

세부아이템(CalendarItem) 컨트롤러에는 회원 인증 로직을 생략한 것이 문제였다.

그러다보니 브라우저에서 url로 직접 접근하는 경우에,

해당 fetch로 직접 진입하고 그러면 해당 스프링 컨트롤러로 직접 요청이 들어가기 때문에 회원 인증이 이뤄지지않았다.

### 해결:

(@AuthenticationPrincipal OAuth2User oauth2User)

Long currentMemberId = authService.getOAuthCurrentMemberId(oauth2User);

이처럼 세부아이템(CalendarItem) 컨트롤러와 서비스 메서드에도 회원 인증 로직을 구성하여 해결하였다.

---

# 트러블 슈팅 58

### 문제:

회원 기능을 구현하며 메서드를 리팩토링 하였다.

calendarRepository.deleteByIdAndMemberId(calendarId, currentMemberId); 처럼 삭제 메서드를 바꾼후,

테스트를 해보니 전과달리 삭제가 되지않는 오류가 발생했다.

### 모색:

리포지터리 메서드 네이밍 오류인것같아

deleteByIdAndMemberId를 deleteByIdAndMember_Id로 바꿔봤지만 여전히 똑같았다.

### 원인:

@Transactional 어노테이션이 없던것이 원인이었다.

그런데 의문인건 처음엔 @Transactional이 없어도 잘 작동됐었다.

그 이유는 JpaRepository 가 기본으로 제공하는 deleteById, delete, deleteAll 같은 메서드들은

명시적으로 @Transactional 를 설정하지않아도 내부적으로 이미 @Transactional 를 포함하고 있기때문에 잘 작동했던 것이다.

### 해결:

서비스 계층 클래스에 @Transactional(readOnly = true) 을 설정하고, 쓰기 메서드에는 @Transactional를 붙여 해결하였다.

---

# 트러블 슈팅 59

### 문제:

포스트맨으로 테스트를 하던 중이었다.

일부러 멤버의 달력이 아닌 다른 달력id 삭제를 요청을 했는데,

예상과 달리 204 응답이 뜨며 정상처리 된듯 보였다.

그러나 DB에서 확인해보면 삭제되지 않았고 따라서 보안은 정상적으로 작동한것이다.

이런 현상이 의문이었다.

### 모색:

jpa 리포지터리의 delete 메서드 기본 작동 방식을 살펴보았다.

### 원인:

calendarRepository.deleteByIdAndMemberId(calendarId, currentMemberId);

위 리포지터리 메서드에서

조건에 맞는 row가 없더라도, delete는 정상적으로 수행되며 응답이 정상적으로 진행되고,

정상 응답을 마치 삭제된듯 착각한 것이 원인이었다.

### 해결:

long deletedCount = calendarRepository.deleteByIdAndMemberId(calendarId, currentMemberId);

if (deletedCount == 0) {

throw new CalendarNotFoundException(ExceptionMessage.Calendar.CALENDAR_NOT_FOUND_ERROR);

}

이렇게 삭제된 row 수가 0이면, 확실하게 클라이언트에서 인지할 수 있도록 오류를 내보내도록 바꿔 해결하였다.

이런 방식으로 하면

find 메서드로 찾은 후에 delete 처리를 하는 경우보다(DB쿼리가 2번나가는 경우보다) 성능상 이점이 있었다.

---

# 트러블 슈팅 60

### 문제:

포스트맨으로 세부 아이템의 수정, 삭제 테스트를 하던 중 의도치않은 상황이 발생했다.

물론 멤버가 해당 아이템id를 가지고 있어야만 업데이트와 삭제가 이뤄지기때문에 보안상의 문제는 없지만,

요청 url에 적혀있는 달력id는 실제 요청엔 아무 영향을 주지 않았기 때문에 의미없는 글자처럼 작용하였고 혼동을 줄수도 있었다.

### 모색:

서비스 메서드에서 해당 달력을 가지고있는지 조회하는 메서드만 하나 추가하면 해결 가능하였다.

그러나 이렇게 하면 가능성이 적은 예외 상황만을 위해서,

대부분 정상 작동의 DB 쿼리는 1개 더 생성되는 비효율적인 성능 문제가 발생할 것이었다.

### 원인:

해당 달력이 존재하는지 검사하는 로직이 없다면,

요청 url에 적혀있는 달력id는 실제 의미가 없는 글자가 되는것이 원인이었다.

그렇다고 존재 검사 로직을 넣는다면,

대부분의 정상 작동 DB 쿼리는 불필요하게 1번 더 날아가는 비효율적인 성능 문제가 발생하는 것이 원인이었다.

따라서 성능을 택하느냐, url에 포함된 달력id의 의미를 살리느냐를 선택해야했다.

### 해결:

성능 개선을 선택해 해결하였다.

즉 스프링 컨트롤러 에서는

@PutMapping("/calendar/{calendarId}/item/{calendarItemId}/update")

이것을

@PutMapping("/calendar/item/{calendarItemId}/update")

이렇게 바꿨고,

리액트에서도 해당 업데이트 api 요청 url을

await fetchHandler(`/api/calendar/${calendarId}/item/${calendarItemResponseDto.id}/update`, "PUT", updateRequestData);

여기서

await fetchHandler(`/api/calendar/item/${calendarItemResponseDto.id}/update`, "PUT", updateRequestData);

이렇게 바꾸었다. 아이템 삭제 url도 똑같은 원리로 리팩토링하였다.

따라서 로직상 url에 불필요한 정보인 {calendarId} 를 생략해 해결하였다.

---

# 트러블 슈팅 61

### 문제:

메인 로그인 버튼을 기존 모달을 활용해 구현하던 중 배경이 까매지기만 하는 현상이 발생했다.

### 모색:

저번에 비슷한 문제가 있었어서, 해당 함수 부분을 살펴봤다.

### 원인:

저번과 마찬가지로 serviceSection의 display값이 아직 none인 상태에서 모달 css 효과만 작동한게 원인이었다.

### 해결:

예전에 startBtnHandlerInRef 를 useRef로 만들어서 serviceSection DOM요소를 전달해준것과 같이,

상위 컴포넌트 App.js에 useRef를 정의하고 공유할 useRef 변수를 props로 나눈 후

serviceSection 객체를 current에 담아 공유하여 비슷한 원리로 해결하였다.

---

# 트러블 슈팅 62

### 문제:

jwt 토큰 방식 적용 전에 훈련을 위해서 세션, 쿠키 방식 인증을 구현하고 있었다.

또한 시큐리티의 csrf 토큰 방식을 쿠키 방식으로 구현하고 있었다.

이때 클라이언트에서 쿠키의 csrf 토큰을 불러오도록 js의 document.cookie 코드를 사용했는데,

원하는대로 csrf 토큰 본문이 추출되는게 아니라 쿠키의 전체 문자열이 추출되었다.

### 모색:

console.log( ) 로 확인해보고, 적절한 파싱 방식을 검색해보았다.

### 원인:

csrf 토큰 본문만 추출되지않는 방식때문에 직접 파싱을 해야하는 것이 원인이었다.

### 해결:

js-cookie 라는 라이브러리를 찾아 해결하였다.

import Cookies from 'js-cookie'; 로 리액트에서 import 받은후

const token = Cookies.get('XSRF-TOKEN'); 과 같이 활용하여 직접 파싱 코드를 작성하지않고 해결하였다.

---

# 트러블 슈팅 63

### 문제:

jwt 토큰 방식 적용 전에 훈련을 위해서 세션, 쿠키 방식 인증을 구현하고 있었다.

또한 시큐리티의 csrf 토큰 방식을 쿠키 방식으로 구현하고 있었다.

이때 클라이언트에서 쿠키의 csrf 토큰을 불러오도록 js의 document.cookie 코드를 사용했는데,

원하는대로 csrf 토큰 본문이 추출되는게 아니라 쿠키의 전체 문자열이 추출되었다.

### 모색:

console.log( ) 로 확인해보고, 적절한 파싱 방식을 검색해보았다.

### 원인:

csrf 토큰 본문만 추출되지않는 방식때문에 직접 파싱을 해야하는 것이 원인이었다.

### 해결:

js-cookie 라는 라이브러리를 찾아해결하였다.

import Cookies from 'js-cookie'; 로 리액트에서 import 받은후

const token = Cookies.get('XSRF-TOKEN'); 과 같이 활용하여 직접 파싱 코드를 작성하지않고 해결하였다.

---

# 트러블 슈팅 64

### 문제:

jwt 토큰 방식 적용 전에 훈련을 위해서 세션, 쿠키 방식 인증을 구현하고 있었다.

또한 시큐리티의 csrf 토큰 방식을 쿠키 방식으로 구현하고 있었다.

이때 클라이언트에서 쿠키의 csrf 토큰을 불러오도록 js의 document.cookie 코드를 사용했는데,

원하는대로 csrf 토큰 본문이 추출되는게 아니라 쿠키의 전체 문자열이 추출되었다.

### 모색:

console.log( ) 로 확인해보고, 적절한 파싱 방식을 검색해보았다.

### 원인:

csrf 토큰 본문만 추출되지않는 방식때문에 직접 파싱을 해야하는 것이 원인이었다.

### 해결:

js-cookie 라는 라이브러리를 찾아해결하였다.

import Cookies from 'js-cookie'; 로 리액트에서 import 받은후

const token = Cookies.get('XSRF-TOKEN'); 과 같이 활용하여 직접 파싱 코드를 작성하지않고 해결하였다.

---

# 트러블 슈팅 65

### 문제1:

CSRF 기능을 위해 스프링의 CookieCsrfTokenRepository 방식을 설정하였다.

리액트에서도 "XSRF-TOKEN" 쿠키 값을 저장하고 fetch에 headers를 추가해

'X-XSRF-TOKEN' 값을 요청 http 메시지에 붙이도록 준비를 끝냈다.

그러나 예상과 달리 요청 이후, 개발자도구로 확인해보니 쿠키에 CSRF 토큰이 저장되지 않았다.

따라서 요청시 토큰값이 인증되지않아 forbidden 에러가 떴다.

### 모색1:

요청을 한번 보내고나서 다시 개발자도구를 확인해보았는데,

그때는 CSRF 토큰이 쿠키에 저장되어있었다.

이와 관련하여 CSRF 토큰이 쿠키에 저장되는 과정에 대해 좀더 알아보았다.

### 원인1:

알아본 결과 POST, PUT, DELETE 같이 상태를 변경하는 요청에 대해서만 CSRF 토큰을 자동으로 쿠키에 저장해주는(응답에 Set-Cookie 해주는)것이라고 판단하였다.

즉 단순 GET 요청일때는 서버에서 CSRF 토큰을 쿠키에 내려주지 않는다고 판단하였다.

### 해결1:

```java
@GetMapping("/auth/csrf-token")
public ResponseEntity<ResponseData<Void>> getCsrfToken(CsrfToken csrfToken) {

    // 이로 인해 csrf 토큰은 자동으로 set cookie된다. (보안상 응답 body에는 토큰을 담지않는다.)
    csrfToken.getToken();

    return ResponseEntity
            .status(HttpStatus.OK)
            .body(
                    ResponseData.<Void>builder()
                            .statusCode(HttpStatus.OK.value())
                            .data(null)
                            .build()
            );
}

```

따라서 위와같이 서버 컨트롤러에 CSRF 토큰 전용 엔드포인트를 만들어 해결하였다.

메서드 파라미터를 통해 csrfToken을 주입받은후, csrfToken.getToken(); 을 통해 토큰을 사용하듯 접근하면

그때는 자동으로 Set-Cookie가 진행된다고 판단하였고,

이후 리액트 코드에서 초기 마운트시 useLayoutEffect 에서 해당 요청으로 CSRF토큰을 Cookie로 저장 받도록 만들었다.

---

# 트러블 슈팅 66

### 문제2:

이전 문제를 해결함으로써, CSRF 토큰은 첫 Post 요청 전에 미리 Cookie에 정상적으로 저장되었다.

그런데 희안한것은 그 상태에서 Post 요청을 했음에도 forbidden 오류가 발생했다.

### 모색2:

CSRF 토큰 발급, 인증 로직에 대해 검색하며 원인을 더 파헤쳐보았다.

시도(1): 헤더 네임의 문제인가싶어

headers: {'X-XSRF-TOKEN': token },

headers: {'X-CSRF-TOKEN': token }

위처럼 여러가지 헤더 네임을 적용해보았지만 원인이 아니었다.

시도(2): js-cookie 라이브러리의 파싱 방식 문제인가싶어

const token = Cookies.get('XSRF-TOKEN');

console.log("token:", token) 으로 헤더에 추가되는 값과, 실제 http 메시지의 토큰값을 확인해보았지만 원인이 아니었다.

시도(3): 브라우저의 Cross-Origin과 Same Site 정책 문제인가 싶었다.

즉 Set-Cookie 할때 따로 설정하지않으면 기본 설정은 SameSite=Lax로 설정되며,

SameSite=Lax는 Cross-Origin에서 발생하는 POST, PUT, DELETE 등의 요청 시에는 해당 쿠키를 서버로 전송하는 것을 차단하는 기능이다.

이런 경우 개발자도구의 http 요청 메시지에는 "XSRF-TOKEN" cookie와 "X-XSRF-TOKEN" 헤더가 명시된것이 확인되더라도

막상 스프링의 서버로 전달되었을때 스프링에서 해당 쿠키 값을 무시하도록 브라우저가 처리할 수 있다고 하였다.

의아했던건 리액트에서 "proxy": "[http://localhost:8080](http://localhost:8080/)" 설정을 했음에도 Cross Origin으로 인식했다는건가 싶었다.

따라서 이것이 진짜인지 확인하기위해 yml에 DEBUG 로그를 추가해서 http 요청 메시지의 스펙을 확인해보았을땐 정상적으로 추가되어있어 혼란스러웠다.

또한 만약 이것이 사실이라면 csrf 기능을 키기전에 로그인 후("JSSESSION" 쿠키가 저장된후) GET, POST, PUT, DELETE 요청이 전부 잘 작동했던 것이 의아했다.

Cross Origin 때문에 쿠키 전달을 막는다면 "JSSESSION" 쿠키도 막혀야하기 때문이다.

그에 대해 다시 찾아보니 개발환경에서 "JSSESSION" 같은 쿠키는 Cross Origin이라도 완화된 Lax 정책으로 정상 작동한다는 설명이 있었다.

혼란스러워서 다시 여러 원인을 찾아보다가,

org.springframework.security.web.csrf: TRACE

org.springframework.security.web.authentication: DEBUG

org.springframework.security.web.access: DEBUG

위와 같이 시큐리티의 디버그 로그를 확인할수있는 명령어를 찾아 확인해보았다.

그 결과

```
... TRACE 29428 --- [nio-8080-exec-4] s.s.w.c.CsrfTokenRequestAttributeHandler : Wrote a CSRF token to the following request attributes: [_csrf, org.springframework.security.web.csrf.CsrfToken]
... TRACE 29428 --- [nio-8080-exec-4] .w.c.XorCsrfTokenRequestAttributeHandler : Not returning the CSRF token since its Base64-decoded length (27) is not equal to (72)
... DEBUG 29428 --- [nio-8080-exec-4] o.s.security.web.csrf.CsrfFilter         : Invalid CSRF token found for http://localhost:8080/api/home
... DEBUG 29428 --- [nio-8080-exec-4] o.s.s.w.access.AccessDeniedHandlerImpl   : Responding with 403 status code

```

위와 같은 스프링 로그를 발견하였고,

"Not returning the CSRF token since its Base64-decoded length (27) is not equal to (72)" 이 로그가 진짜 원인을 찾은 핵심이었다.

### 원인2:

진짜 원인은 바로 시큐리티의 토큰 저장 방식과, 토큰을 다루는 핸들러의 종류가 달랐던 것이 원인이었다.

시큐리티 버전 6.x 이상 최신 버전에서는 CSRF 핸들러가 기존 'CsrfTokenRequestAttributeHandler' 에서 'XorCsrfTokenRequestAttributeHandler' 로 변경된 것이다.

변경된 이유는 기존 토큰 보안성 강화를 위해 저장 방식을 바꿨고, 그 토큰을 다룰수 있는 핸들러가 새로 생긴것이다.

즉 기존 토큰 발급 방식은 UUID 방식이었는데, 여기에 더 복잡한 마스킹 로직을 추가하여 기존과는 다른 형식으로 토큰을 저장하도록 바뀌었고,

그 토큰을 다루기 위해서는 기존 CSRF 핸들러가 아닌  'XorCsrfTokenRequestAttributeHandler' 방식을 핸들러로 설정에 명시해야했다.

토큰 저장 방식은 'CookieCsrfTokenRepository' 방식으로(기존 방식처럼 UUID로) 토큰을 만들고 저장하였지만

핸들러는 'XorCsrfTokenRequestAttributeHandler' 방식으로 설정되어 있다보니,

기존 UUID 토큰을 호환성 없는 핸들러가 다루다가 인코딩 오류가 나서 토큰 검증이 실패한 것이다.

### 해결2:

```java
http
    .csrf(csrf -> {
        // 호환성 문제로 최신 XOR 핸들러를 UUID 토큰 기반 레거시 검증 핸들러로 교체

        // 핸들러 객체를 생성
        CsrfTokenRequestAttributeHandler requestHandler = new CsrfTokenRequestAttributeHandler();
        requestHandler.setCsrfRequestAttributeName(null); // XOR 검사 비활성화

        csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(requestHandler);
    })

```

이처럼 기존 토큰 방식과 호환되는 안정적인 핸들러로 교체하여 토큰 검증 문제를 해결하였다.

또한 새로고침 시 방식의 차이도 해결되었다.

이전에는 새로고침으로 GET 요청시 CSRF 토큰이 쿠키에 저장되지 않고 이후 첫 Post 같은 요청시에만 (데이터 변경 위험이 있는 요청시에만) CSRF 토큰이 쿠키에 저장되었다.

이는 핸들러가 GET 요청시 기존 UUID 토큰이 유효한 토큰이 아니라고 판단하여 Set-Cookie를 누락한 것이다. 다만 Post 요청시에는 새 토큰이 필요하겠다고 판단하여 새 마스킹 토큰을 발급한 것이다.

교체 이후에는 새로고침으로 GET 요청시에도 바로 CSRF 토큰이 잘 발급되도록 해결되었다.

---

# 트러블 슈팅 67

### 문제:

serviceSection.jsx의 상위 컴포넌트인 MainHeader.jsx 안에서 로그아웃 컴포넌트를 구현하고 있었다.

fetch를 사용하고싶었는데, 새로 만들자니 이미 serviceSection.jsx에 완성되어있는 공통 error 처리 로직을 또 다시 구현해야하는 번거로움이 있었다.

### 모색:

이전에 비슷한 상황이 몇번 있었고,

그에 대한 해결 방법인 useRef와 props방식이 떠올라 이를 활용하기로 하였다.

### 원인:

함수를 재사용하려는 위치가 하위 컴포넌트가 아닌 상위 컴포넌트여서,

간단하게 props로 전달하는 방식을 쓰지못한다는 것이 원인이었다.

### 해결:

상위 컴포넌트 App.js에 `const fetchHandlerInRef = useRef(() => {});` 와 같이 useRef를 활용하고,

서로 이어줄 형제컴포넌트들에게 props로 내려줘 함수를 공유해 해결하였다.

---

# 트러블 슈팅 68

### 문제:

로그아웃 버튼을 클릭하면 마치 로그아웃 된듯 잠깐 '로그인'으로 바뀌었다가,

다시 새로고침될떄는 '로그아웃'이 표시되는 등 로그아웃 처리가 되지않는 문제 발생했다.

### 모색:

```jsx
<a id="logoutBtn"
    onClick={ async () => {
        await fetchHandlerInRef.current("/api/logout", "POST");
        setLogin(defaultLoginAtomData);
        window.location.href = "/";

    }}
>로그아웃</a>

```

로그아웃 onClick은 이와 같이 구현 되어있었다.

혹시 렌더링 순서 문제인가 싶어 serviceSection.jsx의 fetchHandler 와 useLayoutEffect 관련 로직을 살펴봤지만 문제를 찾지못했다.

그러다가 setLogin(...)과 새로고침 부분을 주석처리를 하고 다시 함수를 실행하며 원인을 찾아보았다.

### 원인:

fetchHandler 안에서 분기되는 if 조건문이 원인이었다.

즉

`const fetchHandler = useCallback(async (apiPathname, httpMethod, requestData) => {...}`

위와같이 fetchHandler 메서드 파라미터가 구성되어있었고,

메서드 내부에서 해당 요청을 분기처리 하는 조건문은

`else if ((httpMethod === "POST" || httpMethod === "PUT") && requestData) {..}`

위와같이 '&& requestData'로 AND 처리되어 작성되어있던 것이 원인이었다.

즉 이 상황은 POST 요청이면서도 requestData가 필요없는 경우여서 인자에 requestData를 생략하여 실행하였고,

따라서 requestData가 기본적으로 undefined 로 처리되어 저 분기문을 통과하지 못하였으며

로그아웃 fetch 로직 자체를 건너뛰게되어 fetch없이 함수가 끝난것이 원인이었다.

### 해결:

`else if (httpMethod === "POST" || httpMethod === "PUT") {`

...

`body: requestData ? JSON.stringify(requestData) : undefined`

...

위처럼 조건문을 바꿨고,

JSON.stringify(requestData) 부분에서 혹시 모를 null에 의한 오류 방지를 위해 조건연산자를 활용해 해결하였다.

---

# 트러블 슈팅 69

### 문제:

Mysql DB에서 직접 INSERT 쿼리로 가상의 멤버, 가상의 아이템들을 넣어 테스트를 하고있었다.

이미 이전에 직접 브라우저 테스트도 끝냈고, 포스트맨 테스트도 끝냈기때문에 이상이 없을거라 생각했다.

그런데 DB쿼리 후 브라우저 url로 직접 접근하니, 예상치 못하게 다른 멤버가 삽입한 항목이 보이는 현상이 나타났다.

### 모색:

테스트에 사용한 달력 id와, 아이템 id, 멤버 id를 살피면서 따져보았다.

### 원인:

원래 검증 절차는 어플리케이션 단계에서 이뤄지는데,

DB 단계에서 테스트 값을 직접 추가하다보니, 달력id가 서로 겹쳤고

이런 경우는 권한 로직을 우회한 케이스라는 것이 원인이었다.

### 해결:

이렇듯 DB에서 직접 값을 조작하여 테스트하는 경우는 어플리케이션의 권한 로직을 우회하는 경우가 많을 것이라 생각해 리팩토링 하지않았다.

또는 DB에서 기존 달력이나 아이템의 멤버가 존재하면 다른 멤버는 그 id를 소유할 수 없도록 하는 등 강제로 DB에 엄격한 제한을 거는 방법도 있지만,

운영환경에서는 어플리케이션 단계에서 권한을 검사하도록 하고, 성능 이슈도 있는것으로 생각해 이상이 없다고 판단하였다.

---

# 트러블 슈팅 70

### 문제:

Form login 기능을 구현하고 테스트하던중 자꾸 401 코드가 발생하였다.

### 모색:

스프링 시큐리티에서 폼로그인은 기본적으로 username과 password를 파싱할때

application/x-www-form-urlencoded 방식으로 작동한다는 것을 알게되었다.

따라서 리액트 fetch에서 요청 타입을 "Content-Type": "application/x-www-form-urlencoded" 로 설정하고,

데이터를 요청메시지에 담는 방식을

```jsx
const formData = new URLSearchParams();
formData.append("userId", userId);
formData.append("password", password);

...
body: formData

```

이렇게 바꿔서 fetch의 body: formData로 전달하였다.

그러나 그럼에도 자꾸 로그인 실패가 발생했다.

따라서 userId와 password가 제대로 전달되지 확인하기위해 console.log로 요청할때 값을 직접 확인해보았다.

### 원인:

클로저 현상이 원인이었다.

즉 fetchHandler 함수는 useCallback으로 감싸져있었고, 그래서 초기 입력 값인 빈값("")만 담겨서 전달되던 것이 원인이었다.

### 해결:

```jsx
formData.append("userId", requestData?.userId);
formData.append("password", requestData?.password);

```

이렇게 fetchHandler 매개변수에서 값을 읽어와 최신 입력값으로 갱신해 문제를 해결하였다.

---

# 트러블 슈팅 71

### 문제:

Form 회원가입(SignUp.jsx)과 Form 로그인(SignIn.jsx) 컴포넌트의 fetch 핸들러를 공용 핸들러와는 별도로 만들고 있었다.

왜냐하면 중복된 회원을 확인하는 GET 메서드에는 평소 GET과 달리 추가 요청 데이터가 필요했고,

로그인 POST 메서드에는 평소 POST와 달리 시큐리티 기본 작동방식인 "application/x-www-form-urlencoded" 로 설정해줘야했기 때문이다.

그러나 다시 만들다보니, fetch 결과에 따른 가입이나 로그인 처리 구현이 복잡했고 특히 에러처리를 새로운 구조로 재구현해야하는 까다로운 문제가 생겼다.

### 모색:

FetchError.jsx 는 에러객체를 props로 받기만하면 그 뒤는 알아서 처리되도록 구성되어 있었기때문에 SignUp, SignIn 컴포넌트에서

에러 객체만 fetchError.jsx 로 전달해 처리하려는 생각이 먼저 들었지만 형제 컴포넌트여서 새롭게 무언가를 덧붙여야했다.

기존 구조를 재활용하면서 더 단순하게 만들고 싶었다.

따라서 그냥 별도 핸들러를 없애고 에러처리가 구현되어있는 공용 fetch 핸들러를 props로 받아 활용하도록 시도했다.

### 원인:

GET은 평소 GET과 달리 추가 요청 데이터가 존재했고,

POST는 평소 POST와 다른 header 타입과 body 데이터 타입이 존재했던것이 원인이었다.

### 해결:

아래와같이 공용 fetch핸들러의 GET과 POST 분기 속에 또다른 분기를 하나씩 추가했고,

이를 props로 받아 재사용하도록 해결하였다.

```jsx
if (httpMethod === "GET") {

    if (requestData) {
      apiPathname = `${apiPathname}?userId=${encodeURIComponent(requestData)}`;
    } ...
}

...

else if (httpMethod === "POST" || httpMethod === "PUT") {

    const formLoginApi = "/api/formLogin";

    if (apiPathname === formLoginApi) {

      const formData = new URLSearchParams();
      formData.append("userId", requestData?.userId);
      formData.append("password", requestData?.password);
      ...
    }
}

```

---

# 트러블 슈팅 72

### 문제:

Form 회원가입을 할때 리액트에서 입력값 검증을 한것처럼,

서버 단계에서도 Dto에 @Pattern을 활용해 검증 로직을 추가하였다.

그런데 검증 실패 케이스에서 에러 코드로 400을 예상했는데 401이 응답돼서 혼란스러웠다.

### 모색:

스프링의 에러 로그를 확인해보았고 아래 2가지 로그가 있었다.

[... WARN ... MethodArgumentNotValidException: Validation failed ...]

[... DEBUG ... .AnonymousAuthenticationFilter  : Set SecurityContextHolder to anonymous SecurityContext]

### 원인:

원래는 MethodArgumentNotValidException 로 입력값 유효성 검증이 실패하여 기본적으로 400 코드가 응답될 것이지만,

응답 단계에서 시큐리티가 MethodArgumentNotValidException을

AuthenticationException 계열로 착각하고 “인증 실패”로 처리하여 401 코드로 바뀐것이 원인이었다.

### 해결:

스프링에 GlobalExceptionHandler 를 명시적으로 추가하여 해결하였다.

즉 전역 예외 핸들러에서 MethodArgumentNotValidException 예외를 직접 잡아처리했고,

Security는 더 이상 이 예외를 가로채지 않게되어 해결하였다.

---

# 트러블 슈팅 73

### 문제:

Form 회원가입시 email은 선택사항이다. 따라서

email을 아예 입력하지 않으면 자동으로 통과되고,

한글자라도 적게되면 이메일 정규식 형태를 맞춰야 통과하도록 구현했다.

그런데 리액트에서 email 기본 상태 값을 ""로 설정해놓은 상태라서,

이렇게 되면 DB의 email 컬럼에는 빈문자열이 저장되고 이것은 NULL 보다 불명확하고 데이터 낭비일 수 있겠다는 생각을 했다.

### 모색:

"" 라는 빈 문자열도 통과하지 못하도록 바꿀 수 있었지만,

이미 리액트에서의 구현은 ""가 가능하도록 맞춰져있어서 까다로웠다.

### 원인:

원인은 ""라는 빈문자열이 비효율적으로 DB에 그대로 저장되는 것이었다.

### 해결:

MemberFormRegisterDto에서 엔티티로 변환하는 과정에서 toEntity 메서드를 활용한다.

이때 이메서드에서 빈문자열("")을 Null로 변환하도록 아래처럼 삼항연산자로 바꿔 해결하였다.

```java
public static Member toEntity(MemberFormRegisterDto memberFormRegisterDto){
    return                ...
            .email( (memberFormRegisterDto.getEmail() == null ||
                    memberFormRegisterDto.getEmail().isEmpty() ) ? null : memberFormRegisterDto.getEmail() )
           ...
}

```


---

# 트러블 슈팅 74

### 문제:
jwt 토큰 발급 구현중 의문이 2가지 있었다.

첫째 의문 =>
jwt의 setSubject로, String userId = authentication.getName(); 을 반환받아 사용하도록 구현하고있었는데,
폼회원과 오아스회원을 동시 구현했기때문에 authentication 객체는 CustomUserDetails 일수도, CustomOAuth2User 일수도 있었다.
이 경우 authentication.getName(); 은 어떻게 작동하는가가 의문이었다.

둘째 의문 =>
String userId = authentication.getName(); 이때 userId 변수는 반드시 유일값이어야 한다는 정보를 얻었다.
그러나 이때 유일값이어야 하는 이유 또한 의문이었다. 또한 유일값이 아니어서 로직 변경이 필요했다. 



### 모색:
정보를 찾아 첫째 의문을 해결하였다.
authentication 안의 Principal이 CustomOAuth2User 타입이면,
authentication.getName()은 내부적으로 getName() 커스텀 메서드가 호출되고,
CustomUserDetails 타입이면 내부적으로 getUsername() 커스텀 메서드가 호출되는 구조였다.

둘째 의문 또한 해결하였다.
저 변수가 유일값이어야 하는 그 이유는, jwt는 세션 기반과 달리 무상태성(stateless) 기반의 인증 방식이었고,
이에 따라 인증 후 세션 기반과 달리 authentication 객체를 jwt토큰을 매개로 직접 서버에 설정해줘야했기 때문이었다.
즉 세션 기반에서 세션id가 사용자의 유일한 식별자여서 해당 사용자만의 정보나 권한을 알아낼 수 있던 것처럼,
jwt 기반에서 인증 매개는 jwt 토큰이었고 따라서 jwt 토큰이 사용자의 유일한 식별자여야했다.
따라서 jwt 생성시 setSubject에 사용자만의 유일 식별자를 심어놔야했던 것이다.

그렇다면 유일한 식별자를 찾아내야했고,
마침 커스터마이징으로 member 엔티티가 CustomOAuth2User와 CustomUserDetails객체에 주입되는 구조로 만들어놨기때문에
member 엔티티의 id를 식별자로 활용하기로 하였다.


### 원인:
그렇게 해야하는 이유에 대해 잘 몰랐던 것이 원인이었고,
엔티티의 구조상 userId 하나로는 유니크값이 불충족 됐던것이 원인이었다.
즉 authentication.getName();은 해당 멤버 엔티티의 userId 필드가 return 되는 상태였으나,
userId는 provder와 함께 결합되어야 복합키로 유니크 값이 충족되는 구조였다.
이 구조로 만든 이유는 오아스로그인 유저들을 하나의 Member 테이블에 통합 저장하기 위함이었다.
즉 오아스 유저는 provider id가 userId가 되는데, provider가 같을땐 상관없지만 다른 경우 userId가 만에하나 겹칠수 있는 상황이 생기기때문이었다.



### 해결:
정보를 찾아 의문을 해결하고,
member 엔티티가 커스터마이징으로 OAuth2User와 UserDetails 객체에 주입되는 구조로 만들어놨기때문에
member 엔티티의 id를 식별자로 활용하며 해결하였다.

즉 
CustomOAuth2User 에서는 @Override
    public String getName() {
        return String.valueOf(member.getId());
    } 로 메서드를 바꿨고 

CustomUserDetails 에서는

    @Override
    public String getUsername() {
        return String.valueOf(member.getId());
    }

로 메서드를 바꿔, 유일 식별자를 얻어 jwt토큰에 setSubject 하도록바꿔 해결하였다.
또한  jwt 발급에 활용하기위한 메서드로, CustomUserDetailsService에 loadUserByMemberId 메서드를 추가하여 해결하였다.




---

# 트러블 슈팅 75

### 문제:
세션, csrf 토큰 방식을 jwt 방식으로 바꾸고, 테스트를 하는 상황이었다.
일부러 이미 삭제된 달력 주소로 접근하였다. '해당 가계부가 존재하지않는다' 고 표시되는것에서 그쳐야하는데
로그아웃까지 되어버리면서 로그인 모달창까지 나타나는 문제가 발생했다.


### 모색:
로그아웃+모달창 표시까지 된다는것은 fetchHandler에서 401 응답을 받았다는 의미였다.
console.log를 천천히 살펴보며 순서를 따라가보고, 스프링의 로그도 검색해보았다.

이때 로그의 순서를 보니 ResponseStatusExceptionResolver ->
AnonymousAuthenticationFilter -> CustomAuthenticationEntryPoint 순서로 호출되었고,
이것은 에러 디스패치(포워드)에 의한 재진입의 강한 증거로 보였다.


### 원인:
이미 404 Not Found 상태의 스프링 기본 에러 처리가 되도록 구현 되어있었지만,
그 과정에서 시큐리티 필터체인이 에러 처리를 가로채며 미인증 상태로 인식하였고,
다시 401상태로 바꿔서 에러 응답을 내보낸 것이 원인이었다.

즉 상세하게 보자면
스프링 부트의 4xx/5xx의 기본적인 에러 처리는 BasicErrorController가 작동하고 내부적으로 /error 라는 맵핑포인트로 요청을 포워드하며 처리하는데,
이 요청은 새로운 서블릿 디스패치로 간주되고 이 때 시큐리티 필터 체인 전체가 재실행된다.

이 재실행 시점에 SecurityContext 인증은 비어있다.
왜냐하면 Stateless(무상태) 환경인 JWT 방식에서는 SecurityContext가 요청 간에 유지되지 않기때문이다.

또한 /error 요청은 인증 헤더(JWT 토큰) 를 미포함하고, 설령 포함하고 있더라도 필터 체인이 재실행되는 시점에 토큰 검증이 제대로 되지 않을 수 있다.

따라서 필터 체인은 인증되지 않은 상태로 보호된 /error 맵핑 포인트에 접근했다고 판단하여 CustomAuthenticationEntryPoint를 호출하고,
미인증 응답으로 401 Unauthorized 응답을 기존 404 Not Found응답에 덮어씌워 클라이언트에게 보낸것이다.



### 해결:
시큐리티 설정에 .requestMatchers("/error", "/error/**").permitAll() 를 추가하여 해결하였다.
이 경로는 컨트롤러 처리 중 발생한 예외를 웹 서버 규격에 맞게 응답으로 변환하는 것이고,
인증이 필요한 맵핑포인트는 보안이 정상적으로 작동하고있으므로 안전하였다.

---


# 트러블 슈팅 76

### 문제:
http의 환경을 https로 바꾸고나서 오아스 로그인 테스트를 해보니,
기존 팝업창에서 접속이 안됐다.

### 모색:
그 순간 오아스 구현할때 설정했던 구글 클라우드의 설정에서 승인된 리디렉션 URI 설정 등이 떠올랐다.

### 원인:
https프로토콜과 포트 8443으로 변경된 것은 url이 변경되었다는 것이고 
따라서 기존 url 설정을 변경해야했던 것이 원인이었다.

### 해결:
오아스와 관련된 모든 url설정들을 변경하여 해결하였다.
url 설정 뿐만아니라 스프링의 응답메시지에서 리디렉션 시켜주는 url 코드와,
리액트의 오아스 팝업창 관련 url 코드 등도 전부 바꿔 해결하였다.


---


# 트러블 슈팅 77

### 문제:
리프레시 토큰을 redis로 구현할 준비중에, 잠시 도커 컨테이너로 테스트를 하는 중이었다.
Dockerfile을 이용해 원본 .jar 파일 빌드, 포트 설정을 하여 docker image를 만들고,
docker compose 로 해당 docker image 기반의 docker container를 실행하도록 하였다.
그러나 localhost 8080 으로 접속되지가 않았다.

### 모색:
docker ps -a 명령어로, 컨테이너 히스토리를 살펴봤는데, jar가 실행중에 오류가 나서 자동으로 꺼져있었다.
따라서 docker logs myContainer 명령어로 해당 컨테이너의 로그를 상세히 살펴봤고,
[org.hibernate.exception.JDBCConnectionException: unable to obtain isolated JDBC connection [Communications link failure ...]]
오류 로그가 보였다.

### 원인:
원본 프로젝트 yml 파일의 mysql DB 설정이 'localhost'로 하드코딩 되어있었고,
도커 가상환경엔 mysql이 존재하지않아 DB 연결에 실패한 것이 원인이었다.

즉 Dockerfile을 통해 원본 프로젝트를 build하여 최종 실행파일인 .jar를 만들어서 이를 활용하여
도커 이미지를 만들었고, 이 이미지를 동적으로 실행한 인스턴스인 도커 컨테이너를 도커 컴포즈안의 서비스 1개로 등록하여 실행하였다.

그러면 도커 컨테이너들은 각각 리눅스 기반 가상 환경에서 마치 독립된 1개의 컴퓨터처럼 작동하고,
그곳에서 .jar가 실행된것인데 이 .jar 속의 yml 파일 설정엔 mysql DB가 'localhost' 라는 이름으로 하드코딩 되어있었으므로
해당 리눅스 컴퓨터에서는 'localhost' 라는 것을 자기자신으로 인식하며 그 환경안에서 mysql 서버를 찾은 것이다.
하지만 mysql 서버는 해당 환경에 존재하지 않았기때문에 DB 연결이 실패하며 오류가 난것이다.


### 해결:
도커 컴포즈의 services 속성에 mysql 데이터베이스 컨테이너를 1개 더 추가하여 해결하였다.

services 는 그 하위에 각각의 컨테이너를 별칭으로 등록해놓을 수 있고,
이 compose를 실행하면 모든 컨테이너들이 한번에 실행되며 각 컨테이너들은 각각 독립된 가상 ip 주소를 갖는다.
그러나 내부적으로는 각 DNS 서버에 모든 services 별칭들이 등록되어있어서 모든 ip 주소에 손쉽게 연결이 가능한 원리로 문제가 해결되었다.


---


# 트러블 슈팅 78 

### 문제:
redis를 도커로 구현하였고, 폼로그인 리프레시 토큰과 재발급 로직까지 구현하였다.
테스트 도중 일부러 개발자 도구의 쿠키에서 리프레시 토큰을 수동으로 삭제하고, 로그아웃 버튼을 눌렀고
로그아웃 처리는 잘 되었다. 그러나 잠깐이지만 화면에 jwt 관련 오류메시지가 예상치 않게 보였다.


### 모색:
에러가 발생한 구체적인 상황을 알고있었기때문에, 리프레시 토큰이 없을때의 logout 로직을 천천히 따라가보았다.
리액트에서 콘솔로 토큰들을 출력해봤지만 정상적이었고,
백엔드 맵핑포인트에서도 access token과 refresh token이 요청메시지에서 null일때의 로직을 분리처리 해놨기때문에 토큰문자열의 문제는 아니라고 생각했다.


### 원인:
리액트의 LogoutBtn.jsx의 로그아웃 onClick 핸들러에서,
공용 fetchHandler로 로그아웃 요청을 보내기전에 이미 쿠키에서 삭제한 것이 원인이었다.
그러나 fetchHandler 안에서는 처음에 local storage에서 액세스 토큰을 추출하여 모든 요청에 재사용하도록 구현되어 있었다.
그러다보니, accessToken 변수는 undefined 였고 이것이 서버로 전송됐을때 파싱 오류가 난것이었다.

의문점은 서버에서 이미 토큰값이 null인지 분기 처리를 해놓았는데 어떻게 이것을 통과했냐는 것이다.
원인은 
 String authorizationHeader = request.getHeader("Authorization");
        if (authorizationHeader != null && authorizationHeader.startsWith("Bearer "))

이렇게 Authorization와 Bearer 까지만 체크하는 것이 원인이었다.
리액트 에서는 "Authorization": `Bearer ${accessToken}`, 이렇게 전송되도록 하니까 if문 조건에 맞아 통과한 것이다.



### 해결:
리액트 로그아웃 핸들러 내부의 코드 순서를 바꿔 해결하였다.
그렇게 fetch에 액세스 토큰과 리프레시 토큰 모두를 담아 보내고,
이에 따라 서버에서는 redis에 저장되는 토큰 무효화에 더욱 유리해졌다.
이후 서버에 대한 요청이 성공하든 실패하든 그 밑으로 코드가 흘러가 액세스토큰 또한 삭제되며 문제를 해결하였다.



---


# 트러블슈팅 79

### 문제:
jwt 액세스 토큰과 리프레시 토큰 발급 후 구현과 테스트까지 마친 상태였다.
로그아웃까지 된 상태에서 우연히 뒤로가기를 하다보니 달력 생성 url( /create) 에서,
"로그인이 필요합니다" 라는 에러화면이 표시되어야하는데 예상과 달리 달력 생성 페이지가 그대로 보였다.


### 모색:
전역 recoil의 login상태 분기에 따라 jsx 내용이 return 되게 하든지, 아닌지로 처리하려 하였다.
그러나 그렇게하면 화면에 표시만 안될뿐이지, 다른 컴포넌트처럼 자동으로 통합에러화면으로 가지는 못했다.
그래서 정식 인증 맵핑포인트인 ( /auth/me )로 fetch하는 로직을 만들어 해결하려했지만,
그러면 달력을 생성하러 들어갈때마다 매번 무거운 인증 처리 로직이 작동하여 성능 과부하가 예상되었다.


### 원인:
달력 생성 컴포넌트는 마운트시 바로 최신 값을 확인할 상황이 없었기 때문에 첫 마운트될때 fetch 통신 로직을 생략했던 것이 원인이었다.
즉 이미 다른 컴포넌트 에서는, 첫 마운트가 되면 바로 자동으로 최신 값 갱신을 위해 useLayoutEffect에서 fetch핸들러가 작동하며
미인증 상태 또는 fetch 통신 오류가 확인되고, 따라서 공용 fetchHandler의 통합 에러처리로 정상적인 에러 화면이 표시되는 상태였지만, 
달력 생성 컴포넌트는 달력 생성시에만 fetch가 필요했기때문에, 마운트시에 fetch 통신으로 미인증 상태가 확인될 길이 없던 것이 원인이었다.


### 해결:
스프링에 아래와 같이 무겁지 않은
fetch 통신 맵핑포인트( /auth/ping )를 만들어 인증 상태와 fetch 상태가 잘 확인되도록 하였고
이것을 useLayoutEffect 내에서 첫 마운트시 호출되도록 만들어 해결하였다.

// 시큐리티 필터로 인증 상태만 체크하기 위한 맵핑포인트 
    @GetMapping("/auth/ping")
    public ResponseEntity<ResponseData<Map<String, String>>> getAuthFetchStatus() {

        Map<String, String> authFetchSuccessData = Map.of(
                "message", "통신 성공"
        );

        return ResponseEntity
                .ok(ResponseData.<Map<String, String>>builder()
                        .statusCode(HttpStatus.OK.value())
                        .data(authFetchSuccessData)
                        .build());
    }



---

# 트러블슈팅 80

### 문제:
트러블슈팅 79를 해결하다가, 다른 맵핑포인트의 직접 접근 케이스를 방어하지 않은것이 떠올랐다.
즉 오아스 팝업창 로그인 인증 성공후, 리다이렉트되는 전용 리액트 맵핑포인트 라우터 ( /oauth/point ) 를 구현하였는데,
이를 직접 url로 접근하는 경우의 방어 로직을 구현해야했다.


### 모색:
시간이 부족하여 챗지피티에게 물어보며, 간단한 if문을 추가하는것으로 도움을 받았다.

즉
useLayoutEffect(() => {

        // URL에서 쿼리 파라미터를 추출.
        const params = new URLSearchParams(window.location.search);
        const accessToken = params.get('accessToken');

	
	[이곳에 if문 추가하기]
     

        // 부모 창에 토큰과 함께 메시지를 전송.
        if (window.opener && !window.opener.closed) {
            if (accessToken) {

                window.opener.postMessage({
                    type: "OAUTH_SUCCESS",
                    accessToken: accessToken
                }, "https://localhost:3000");
            }

        }
        // 팝업 창 닫기
        window.close();
    }, []);


기존 위의 [이곳에 if문 추가하기] 부분에

if (!accessToken) {
	window.location.replace("/");
	return;         
  }

이 코드만 추가하는 조언을 받았다.

그러나 다시 여러 케이스를 생각해보니,
이렇게 되면 만약 팝업창에서 accessToken이 빈값이었을때 바로 팝업창이 리다이렉트되며 return 으로 코드가 끝나버리게 된다.
따라서 오아스 로그인 팝업창은 닫히지않고 홈화면으로 이동해버리는 의도치 않는 상황이 발생했다.



### 원인:
url로 직접 접근했을때, 팝업창에서 정상 로직대로 진행됐을때,
accessToken이 빈값인 경우일때 등 모든 경우에서 방어가 이뤄지는 코드가 아닌것이 원인이었다. 



### 해결:
아래와 같이,
모든 경우의 수에서 안전하게 작동할 수 있는 조건문 if (!window.opener)를 추가하는 방법이 떠올라서 해결하였다. 

useLayoutEffect(() => {

        // URL에서 쿼리 파라미터를 추출.
        const params = new URLSearchParams(window.location.search);
        const accessToken = params.get('accessToken');


        if (!accessToken) {

            if (!window.opener) {

                // url로 해당 라우터로 직접 접근한 경우는 리다이렉트 처리한다.
                window.location.replace("/");
                return;
            }
        }

        // 부모 창에 토큰과 함께 메시지를 전송.
        if (window.opener && !window.opener.closed) {
            if (accessToken) {

                window.opener.postMessage({
                    type: "OAUTH_SUCCESS",
                    accessToken: accessToken
                }, "https://localhost:3000");
            }

        }
        // 팝업 창 닫기
        window.close();
    }, []);

---
<br><br><br><br><br>
