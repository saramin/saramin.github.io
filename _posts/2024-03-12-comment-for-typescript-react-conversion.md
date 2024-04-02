---
title: React + TypeScript 전환기 (Feat. msw)
author: 지성봉
---

<style>
	  .fig-caption {display:block; margin-top:0px; padding:5px; font-size:0.85em; font-style: italic; color: #555}
</style>

안녕하세요. 사람인 FE개발팀 지성봉입니다.

사람인 FE 개발팀에서는 기존의 사람인 서비스를 점진적으로 FE 분리 전환을 진행 중에 있는데요, 최근 사람인 서비스 중 신입·인턴 채용달력 모바일 서비스(이하 채용달력)를 React + TypeScript(이하 TS)로 전환하게 되었습니다.

React + TS로의 전환은 제 개인적으로도 제법 작지 않은 도전이기도 했습니다.   
지금까지는 Vue로 개발해 왔었고, 표준 기술로 선택 한 React, TS, React Query 등은 프로덕트 레벨에 적용하는 첫 사례에 해당했거든요.

특히 학습한 것을 조금씩 서비스에 반영하며 점진적으로 개선해 가는 것이 아니라, 이번처럼 제한 된 시간 안에  그 동안 학습해 온 것을 한 순간에 쏟아내어 온통 새로운 기술 스택으로 서비스를 완성해 가야 할 때는 상당히 적지 않은 부담을 느끼며 새로운 기술 스택에 적절한 설계와 방법론을 고민해야 하기 때문에 제법 도전이 될 수 밖에 없는 것 같습니다.

아직은 많은 시행착오를 겪지 못했고 새롭게 익힌 기술 스택에 대해 경험과 지식이 충분하게 쌓여있지 않기에 미흡한 부분도 있고 이것저것 시도해보고 있는 것들도 더러 있기 때문에, 이 글에서 소개하는 것들이 어쩌면 다소 적절하지 않거나 미흡한 방법을 수도 있습니다. 혹 그러한 것들이 눈에 뜨인다면 학습 직후의 것이라 아직은 다듬어지지 못한 거친 돌이라 너그러이 생각해주시면 좋을 것 같습니다.
## TS 도입
FE 개발팀에서는 작년 KPI의 일환으로 팀 내 기술 표준을 정립하면서 TS를 도입하기로 결정했었는데요, 채용 달력이 TS를 도입하기로 결정하고 학습 이후에 적용한 첫 케이스이기도 했습니다. 

개인적으로 팀 개발 프로세스에서 다음의 사항들을 문제라고 인식했었는데요.

- 팀 내 개발 프로세스에서 테스트가 작성되고 있지 않고, 테스트를 도입하기에는 아직 역량이 충분하지 않다.
- 코드 리뷰 단계에서 예외 처리 누락이나 오탈자로 인한 오류가 종종 발견되고, 자주 발견되는 만큼 코드 리뷰에서 검증해야 할 사항이 많아진다.
- 컴포넌트, 함수 등에 대한 문서화가 이루어지지 않아 코드 레벨에서 사용법을 확인하거나 스토리북에서 사용할 컴포넌트를 찾아다니느라 낮은 생산성이 지속된다.

이러한 문제를 당장 해결하는데에는 TS가 가장 좋은 대안이라고 판단했기 때문에, TS 도입에 적극 지지하는 입장이기도 했습니다.
### Type 관리 전략

채용 달력 서비스에서는 몇 번의 시행착오를 거쳐 type을 크게 두 가지 카테고리로 나누어 관리하도록 작성되었는데요.

컴포넌트 props 및 컴포넌트에 종속되어 있는 types는 각 컴포넌트 디렉토리 안에 위치하여 local types로 관리하고, 그 외 Router, API fetcher, React Query, utils, 3rd party 등에 필요한 types는 types 디렉토리 안에 위치하여 global types로 관리하도록 작성되었습니다.

<figure>
     <img src="/img/type-directory-strategy.png" alt="프로젝트 디렉토리 스크린샷" />
	    <figcaption class="fig-caption">좌: Badge 컴포넌트 디렉토리, 우: types 디렉토리</figcaption>
</figure>

초기에는 type 정의를 각 컴포넌트 및 모듈 파일에 직접 작성하고 있었는데, 다음의 상황들을 만나게 되었습니다.

- import 되는 컴포넌트 및 모듈 개수 만큼 type import문이 작성되면서 import문이 화면 절반 이상을 차지하는 상황이 발생한다.
- 문서화를 위해 props type에 작성된 JSDoc 어노테이션으로 인해 type 정의 부분이 화면 한 페이지를 차지할 만큼 길어지는 상황이 발생한다.
- 단일 파일에 여러 개의 함수 등이 작성되는 경우, 연관된 type과 코드를 가까이 배치시키면 경계를 명확히 하기 위한 추가 장치가 필요해지는 문제가 발생하고, type 정의와 코드 모음을 분리하여 배치시키면 수정 시 종종 위치 찾기를 위해 스크롤 이동으로 인한 스위칭 비용이 높아지는 문제가 발생한다.

우선 첫 번째 문제를 해결하기 위해 모든 type을 global type으로 변경해 봤는데

이번에는 storybook에서 정의된 props들을 표현하지 못하는 문제가 발생했습니다. 
storybook github issue에서 local types만 지원하고 있다는 내용이 발견되어 컴포넌트에 한하여 local로 사용하게 되었습니다.

{% raw %}

```typescript
import type { HTMLAttributes } from "react";

let VALID_HEADING_LEVEL: readonly [1, 2, 3, 4, 5, 6];

declare global {
  type HeadingLevel = (typeof VALID_HEADING_LEVEL)[number];
}

export interface HeadingProps extends HTMLAttributes<HTMLHeadingElement> {
  /**
    * the level of heading
    */
  level: HeadingLevel;
}
```

{% endraw %}

<p class="fig-caption">
    Heading 컴포넌트에 정의한 type 사례.<br> 컴포넌트를 벗어나 다른 곳에서 쓰일 소지가 있는 것들은 global로 정의 됩니다.
</p>

두 번째와 세번째 문제를 해결하기 위해 타입 정의 부분만 별도 파일로 분리하여 types 디렉토리에서 관리하고 있는데, 스크린샷에서 보이듯 관심사별로 묶어서 타입을 모아두는 방식을 택하고 있습니다. 

{% raw %}

```typescript
// types/services.d.ts
declare interface SaraminApiError {
  message: string;
  code: string;
}

declare interface SaraminApiResponse<T> {
  success: boolean;
  error: SaraminApiError | null;
  data: T;
}

// src/services/CalendarApis.ts
export const getThemes = (
  params?: object,
): Promise<SaraminApiResponse<ThemesResponse>> => { ... };
 
// src/hooks/useCalendarApis.ts
export const useGetThemes = (
  params?: object,
  config?: Omit<
    UseQueryOptions<
      SaraminApiResponse<ThemesResponse>,
      AxiosError<SaraminApiResponse<null>>,
      ThemesTransform
    >,
    "queryKey"
  >,
) => { ... }
```

{% endraw %}

<p class="fig-caption">
    API 응답 type 사례. 이렇게 정의해두면 어느 API Fetcher, React Query hooks 등에서든 import 없이 바로 사용할 수도 있고, 인텔리센스에서도 즉시 확인 할 수 있었습니다.
</p>

컴포넌트도 types 디렉토리에 위치시켜 관리하도록 시도해보았는데, 컴포넌트별로 작성하게 되면 types 내의 파일이 방대해지도 하고 일부 파일은 3~4줄로 작성이 그치는 경우도 있었고, 아토믹 시스템에서의 수준별로 묶어도 봤지만 특정 컴포넌트에서만 요구되는 literal type이나 네이밍이 겹치는 것을 피하기 위한 네이밍 규격을 추가해야 하는 등의 고려사항이 더 발생하여 결국 아직까지는 썩 와닿는 좋은 방법을 찾지 못해 결국 컴포넌트에 필요한 types는 컴포넌트 디렉토리에서 관리하도록 구성되었습니다.

### TS를 도입하면서 어려웠던 점

TS를 도입하기 전에는 다소 어렵지 않을까 막연한 부담이 있었는데요, 막상 적용을 시도해보니 IDE와 린트가 워낙 잘 받쳐줘서 생각보다 어려움은 없었습니다.

내부에서도 TS관련된 린트를 타이트하게 잡으면 TS에 제법 많은 시간을 할애하게 되서 일정에 영향을 줄 수 있을 거라는 조언도 있었는데, 운이 좋아서일지는 모르겠지만 의외로 TS도입으로 인해서 일정이 영향을 받는 일은 없었습니다.

떠도는 이야기로 TS를 도입하게 되면 type을 정의하느라 더 많은 키보드 타이핑을 해야 하고 그래서 더 많은 시간이 소비된다는 이야기들이 지금도 있는 것으로 아는데, 오히려 자잘한 오타로 인한 오류나 이벤트 핸들러 속에 숨어 발견 되기 어려운 오류를 방지하기 위해 눈을 부릅뜨고 코드를 리뷰해야 하거나 뒤늦게 오류를 해결하느라 소비하는 시간 등을 따지면 오히려 더 많은 시간이 절약되지 않았나 싶습니다.

다만 TS를 도입하면서 어려웠다고 느꼈던 건 외부 라이브러리를 사용할 때였는데요.

<figure>
     <img src="/img/what-the.png" alt="제법 많은 양의 오류 안내 설명 화면 캡쳐" />
	    <figcaption class="fig-caption">타입을 정의해 가는 중, 처음에는 무척이나 당황하게하는 어마무시한 양으로 느껴지는 오류 안내 메세지였…</figcaption>
</figure>


특히 React Query를 사용하면서 Custom hook을 만들어 적용할 때 요구되는 type을 작성하느라 진땀을 뺐던 것 같습니다 😂 TypeScript를 적용하던 초기에는 type이 일치하지 않다는 메세지를 올바르게 해석해내지 못하고 헤메이느라 시간을 좀 더 잡아먹기도 했는데 프로젝트 후반부에 들어서는 어느 정도 익숙해졌는지 라이브러리에 정의 된 type을 추적해가며 해결할 수 있게 되었습니다.

<figure>
     <img src="/img/dts-for-tanstack.png" alt="라이브러리의 type 정의 파일을 찾아서 이해할 수 있게 되었다" />
</figure>

이러한 것만 아니면 TS를 도입하는데 초기에는 큰 어려움은 없다고 생각하는데요, 막연히 어려울 것이라는 걱정 때문에 미루고 있는 분이 계시다면 도입 시도를 적극 추천합니다.

### TS도입으로 얻은 좋은 점
개인적으로 TS도입 이전에 비해 더 만족감을 느끼는 부분들을 몇 가지 소개해보자면
- 코드리뷰에서 불필요한 리소스가 조금 더 줄었다

    <figure>
       <img src="/img/code-review-space.png" alt="띄어쓰기 오류에 대한 코드 리뷰 화면 캡쳐" />
	     <figcaption class="fig-caption">띄어쓰기로 인한 오류. 눈을 크게 뜨고 봐야 한다</figcaption>
    </figure>	
		
    정적 타입을 사용하지 않던 이전 환경에서 코드리뷰를 하면 혹시나 숨어 있을지 모를 누락된 예외 처리, 오탈자로 인한 오류까지도 함께 검사해야 하거나 꼼꼼히 살펴보지 않으면 리뷰어 조차도 놓치기 쉽기 때문에 제법 많은 리소스가 발생되었는데 정적 검사가 이를 대신하여 경고를 띄워 리뷰 전 단계에서 해결 할 수 있게 되어 더 본질적인 것에 집중하여 리뷰가 가능해졌습니다.
		
    <figure>
       <img src="/img/code-review-null-exception.png" alt="[API가 예외를 내려주지 않으면 발견하기 쉽지 않은 오류 코드 리뷰 화면 캡쳐" />
	     <figcaption class="fig-caption">API가 완성되기 전까지는 알아차리기 힘들지도 모를 그런 오류. 하지만 TS였다면?</figcaption>
    </figure>	
		
    때로는 컴포넌트를 먼저 개발하고 이후에 API fetcher를 개발하면서 가공된 데이터에서 예외가 발생하거나 타입의 불일치로 일어나는 문제를 실제로 구동해볼 때에야 확인할 수 있거나 리뷰 시 관련 파일을 열어두고 비교해야 했었는데 TS에서 미리 API 응답 데이터의 타입을 정의해두고 시작하면서 이러한 문제 역시 사전에 방지할 수 있게 되면서 리소스가 좀 더 절약될 수 있었습니다.
		
-   개발 속도가 더 빨라졌다

    이전에는 컴포넌트를 가져다 쓰려면 컴포넌트의 필수 prop이 무엇인지 혹은 사용 가능한 값을 확인 하기 위해 화면 한 쪽에 컴포넌트 코드를 열어두거나 storybook을 띄워두고 찾아다니거나 해야 했지만, TS도입 이후로는 IDE의 인텔리센스 지원으로 손 쉽게 처리할 수 있게 되어 체감되는 개발 속도 향상이 제법 높게 느껴졌습니다.
		
    <figure>
       <img src="/img/vscode-intelisense.jpeg" alt="[Visual Studio Code intelisense 화면 캡쳐" />
	     <figcaption class="fig-caption">인텔리센스에서 필수 속성의 누락 및 사용 가능한 속성을 알려주고, 누락된 속성 추가 기능으로 필수 속성을 한 번에 삽입할 수 있습니다.</figcaption>
    </figure>
				
-   가독성이 좋아졌다

    단순히 작성된 코드를 알아보기 쉬워진 것이 아니라, 컴포넌트나 함수를 어떻게 사용해야 할지 이해가 더 쉬워졌습니다.

    간혹 overloading을 사용하여 함수를 작성할 경우가 있는데, 특히 파라미터 순서에 따라 타입이 다른 경우에는 파라미터 이름만으로 가독성을 지원하는데 한계가 있는 경우가 있었는데, TS를 통한 overloading 표현으로 IDE가 적절하게 지원해주면서 함수를 이해시키기 위한 문서를 추가로 작성하거나 문서가 없는 함수를 이해하기 위해 코드 내부를 들여다보아야 하는 일을 줄일 수 있게 되었습니다.
		
    {% raw %}

    ```typescript
    /**
      * date format의 문자열, 날짜 객체로도 비교 가능하고,
      * 주어진 연월 또는 연월일로도 비교 가능한 함수를 만들다보니,
      * 시간이 흐른 뒤, 이게 뭐하던 건지 파악하려면 작성한 나도 코드를 따라가며 해석해야 했다.
      */
    function isSameDay(other, year, month, date) {
      if (typeof year === `string`) {
        return other.setHours(0, 0, 0, 0) === new Date(year).setHours(0, 0, 0, 0);
      }
      if (year instanceof Date) {
        return other.setHours(0, 0, 0, 0) === year.setHours(0, 0, 0, 0);
      }
      if (typeof year === `number` && typeof month === `number`) {
        return (
          other.setHours(0, 0, 0, 0) === new Date(year, month, date || 1).setHours(0, 0, 0, 0)
        );
      return false;
    }
    
    /**
      * type과 함께 overload로 표현하면서 평안이 찾아왔다
      */
    function isSameDay(other: Date, date: string | Date): boolean;
    function isSameDay(other: Date, year: number, month: number): boolean;
    function isSameDay(other: Date, year: number, month: number, date: number): boolean;
    function isSameDay(
      other: Date,
      year: number | string | Date,
      month?: number,
      date?: number,
    ) { ... }
    ```
    
    {% endraw %}

    이 외에도 리팩토링이 매우 수월해진 점, 설계가 용이해진 점 등 여러 가지 좋은 점들이 있었는데요 지면상 여기까지만… 😅
		아직 TS 도입을 고려만하고 할까말까 고민중이라면 어서 도입하고 평안을 얻으시길 기원합니다.

## MSW 도입
채용 달력에서는 API를 mocking하기 위해서 msw(mock service worker)를 사용하고 있습니다.    
msw는 이름에서 볼 수 있듯 service worker를 이용하여 모의(mock)를 도와주는 라이브러리로, service worker가 HTTP 요청을 가로채 등록해 둔 모의 응답을 전달해주는 기능을 합니다.

<figure>
     <img src="/img/msw-on-dev-tools-network.png" alt="크롬 개발자도구 네트워크 탭 화면 캡쳐" />
	   <figcaption class="fig-caption">msw가 네트워크를 가로채 실제 API처럼 작동해줍니다.</figcaption>
</figure>

Storybook과도 통합 가능하고, request에 따라서 응답을 변경할 수도 있기 때문에 프론트엔드 개발에서는 매우 유용한 라이브러리입니다. 

참고로, 프로젝트에 msw를 도입한 2월 기준으로 msw-storybook-addon이 아직 msw 1.x 버전만 지원하고 있기 때문에 msw 역시 1.x 버전을 사용해야 합니다. 그렇지 않으면 저처럼 2.x로 모든 설정을 끝낸 후에 다시 1.x로 다운그레이드 하는 불상사가 발생하게 될거에요 🤣
적용 방법 및 Storybook과의 통합 방법은 [msw 공식 문서](https://v1.mswjs.io/docs/)와 [storybook addon 문서](https://storybook.js.org/addons/msw-storybook-addon/)에 잘 정리되어 있어 그대로 따라하기만 해도 되기 때문에 굳이 여기에서까지 작성하는 것은 피하겠습니다.

채용달력에서는 이미 만들어진 API를 사용하는 부분도 있고, 앞으로 수정 될 응답을 사용해야 하는 경우도 있었는데요, msw를 통해 실제 API를 호출하여 작동하는 것과 같은 효과를 가질 수 있기 때문에 저희처럼 API에서 응답되는 실제 데이터를 미리 넣어두고 사용하거나 앞으로 생성 될 혹은 변경 될 규격에 맞춰서 데이터를 넣어두고 개발하면 API 개발자의 개발 완료 시점을 기다리지 않아도 되게 되었습니다. ~~(아, 물론 이제는 API가 먼저 나와야 할 수 있어요 같은 핑계를 못대게 되는 읍! 읍!…)~~

{% raw %}

```typescript
const recruitSchedule: SaraminApiResponse<ScheduleRecruitsResponse> = {
  success: true,
  error: null,
  data: {
    list: {
      "20240119": [
        {
          company_nm: "사람인",
          title: "사람인 FE개발팀 신입 채용",
          ...,
          company_logo_url: "",
          meta_tag: "서울 구로구 · 무관", // 추가될 데이터 
        },
        {
          company_nm: "사람인",
          title: "사람인 FE개발팀 신입 채용 2",
          ...,
          company_logo_url: "",
          meta_tag: "", // 빈 문자열로 올지도 모르니까 일단...
        },
        {
          company_nm: "사람인",
          title: "사람인 FE개발팀 신입 채용 3",
          ...,
          company_logo_url: "",
          meta_tag: "", // 혹시 property를 빼고 오는 경우가 있을 수도 있으니...
        },
```

{% endraw %}

개인적으로 msw 도입에의 가장 큰 효과는 API의 실패 케이스를 미리 대응할 수 있는 점일 것 같습니다. API 응답에 따른 처리가 필요한 기능을 만들 경우, 성공 케이스에만 대응하고 실패 케이스를 놓치게 되거나 혹은 실패 상황을 만들기 어려워서 상상 속에서 개발하는 상황을 종종 보아왔는데요. msw를 도입하면서 응답이 늦는 경우, 특정 HTTP status를 대응해야 하는 경우, 오류 상황을 재현하는 스토리를 작성해야 하는 경우 등을 모두 과거에 비해 훨씬 더 손 쉽게 만들어내어 개발 할 수 있게  있게 되었습니다.

{% raw %}

```typescript
import { rest } from "msw";
import { ..., recruitSchedule } from "@mocks/data/calendar";

const handlers = [
  ...,
  // request를 가로채 parameter에 따라 다른 데이터를 줄 수도 있다
  rest.get(`/calendar/recruits/schedule`, (req, res, ctx) => {
    const monthParam = req.url.searchParams.get("month");
    const start = req.url.searchParams.get("startDate");
    const end = req.url.searchParams.get("endDate");
    const getMonthlyResult = (month: string) =>
      Object.entries(recruitSchedule.data.list).filter(([date]) =>
        date.includes(month),
      ) || [];
    const getWeeklyResult = (startDate: string, endDate: string) =>
      Object.entries(recruitSchedule.data.list).filter(
        ([date]) => date >= startDate && date <= endDate,
      ) || [];
    if (!monthParam && !(start && end)) return res(ctx.status(500), ...);
	
    return res(
      ctx.status(200),
      ctx.json({
        ...recruitSchedule,
        data: {
          ...recruitSchedule.data,
          list: Object.fromEntries(
            monthParam ? getMonthlyResult(monthParam) : getWeeklyResult(start, end),
          ),
        },
      }),
    );
  }),
  ...
];
```

{% endraw %}

## HOC 패턴 적용
혹시 사람인 신입·인턴 채용달력 페이지를 보셨나요? 여러 페이지로 구성되어 있지만 화면 구성은 결국 주간/월간뷰의 차이를 제외하고는 나머지 구성은 항상 동일하게 되어 있는데요.

<figure>
     <img src="/img/calendar-screenshot.jpeg" alt="사람인 채용달력 주간뷰 및 월간뷰 화면 캡쳐" />
</figure>

월간 달력이나 주간 달력이냐를 제외하면 나머지 부분은 동일한 형태를 유지합니다. 
대략 코드 형태로 보자면

{% raw %}

```typescript
// 월간뷰 페이지 구조 
fetchDatas...

handle user interaction...

<Themes themes={...} onClickXXX={ ... } />
<DateController date={...} onClickXXX={...} />
<NavigationTabBar date={...} subscribeList={...} />
<MonthlyCalendar  date={...} scheduleRecruitList={...} />
<Toolbar onClickXXX={...} />
<Recruitment scheduleRecruitmentMap={...} />
<Notice />

// 주간뷰 페이지 구조

fetchDatas...

handle user interaction...

handle weeklyCalendar interaction...

<Themes themes={...} onClickXXX={ ... } />
<DateController date={...} onClickXXX={...} />
<NavigationTabBar date={...} subscribeList={...} />
<WeeklyCalendar date={...} scheduleRecruitList={...} />
<Toolbar onClickXXX={...} />
<Recruitment scheduleRecruitmentMap={...} />
<MonthController />
<Notice />
```

{% endraw %}

<p class="fig-caption">
동일한 로직과 동일한 컴포넌트가 겹치는 부분이 많았다. 이걸 페이지마다 다 따로 작성하는게 과연 적절한가?
</p>

이러한 형태가 되었는데 처음에는 월간 템플릿과 주간 템플릿을 각각 작성하고, 주간 템플릿에만 요구되는 추가 UI 기능들을 작성하는 방향을 떠올렸습니다.

하지만 이내 나머지 영역이 모두 동일한 비지니스 로직을 가지고 동일한 기능을 가지는데 굳이 분리해서 관리해야 할 이유가 있을까하는 의문이 들기 시작했습니다. 동일한 부분을 각각 작성하는 순간 이후 유지보수가 발생 할 경우 양 파일 모두에 반영하고 코드 리뷰 시 리뷰어도 양쪽의 로직이 동일하게 적용되었는지 혹 빠뜨린 부분은 없는지도 리뷰해야 하기 때문에 오히려 품질을 유지하는 것을 더 어렵게 할 것이라는 생각이 들었고 변경 사항을 추적하는 것 역시 다소 복잡해질 소지가 있다는 판단이 들었습니다.

재사용 문제를 해결하기 위한 장치로 hook이 있지만, 이 경우에서는 하나의 hook이 서로 다른 관심사를 가지는 여러 컴포넌트를 반환하게 하는 형태가 되고 오히려 유지보수를 더 어렵게 만들 소지가 높을 것 같아 해결 방법에서 제외하였고, 이전에 글로만 봐왔던 High Order Component(HOC) 패턴을 사용하기로 했습니다.

{% raw %}

```typescript
// src/hocs/withCalendarCommon.tsx
const CalendarCommon = <P extends ComponentProp>(Component) => {
  return (props: P) => {
    fetchDatas...
	  
    handle use interaction...
	  
    return (
      <>
        <Themes themes={...} onClickXXX={ ... } />
        <DateController date={...} onClickXXX={...} />
        <NavigationTabBar date={...} subscribeList={...} />
        <Component {...props} />
        <Toolbar onClickXXX={...} />
        <Recruitment scheduleRecruitmentMap={...} />
        <MonthController />
        <Notice />
      </>
    );
  };
}
```
{% endraw %}

{% raw %}

```typescript
// src/pages/theme.tsx

import MonthlyTemplate from "@components/Templates/Monthly";
import WeeklyTemplate from "@components/Templates/Weekly";
import withCalendarCommon from "@hocs/withCalendarCommon";

const ThemePage = () => {
  const { week } = useParams();
  // something to do only on theme page ...

  const Component = week
    ? withCalendarCommon(WeeklyTemplate)
    : withCalendarCommon(MonthlyTemplate);

  return <Component />;
};

export default ThemePage;
```

{% endraw %}

<p class="fig-caption">
상이한 부분만 템플릿화 하여 동일한 로직을 사용하는 부분은 HOC 적용
</p>


이러한 식으로 동일한 비지니스 로직과 동일한 UI 기능 및 동일하게 사용되는 컴포넌트를 더 이상 두 벌에서 관리하는 문제를 해결할 수 있을 것으로 보여졌습니다.

다만 HOC의 사용이 많아질 경우에는 props 추척이 복잡해질 수 있기 때문에 HOC가 중첩되는 일이 없도록 규칙을 정리하고 현재는 여기에만 사용하고 있습니다.

## 마무리하며

완전히 새로운 기술 스택을 이용하여 프로덕트를 만드는 건 처음 프론트엔드 개발자로 발걸음을 떼었던 때와 유사한, 물론 그 때보다는 덜했지만, 지금까지 학습한 것만으로 충분히 해결 될 수 있을까? 새로운 것 옆에 새로운 것 옆에 새로운 것들의 조합에서 문제가 발생했을 때 빠르게 원인을 찾아서 해결할 수 있을까? 등 묵직한 부담을 어깨에 올려두고 낯섦과 마주해보았습니다. 

일부 어려울 것이라고 막연한 생각들로 새로운 것을 잘 할 수 있을까하는 막연했던 부담들은 우려했던 것 보다 더 많이 어렵지 않게 다가와 부담을 지워낼 수 있었고, 뒤로 갈 수록 ‘아 이렇게 하면 더 좋았겠구나’ 싶은 부분도 조금씩 보이기 시작하고 있습니다.

이제 한 걸음 더 나아가 더 좋은 방법론을 고민해보고 더 딥다이빙할 시간을 맞이할 준비를 해봐야 겠습니다.

긴 글 읽어주셔서 감사합니다.
