---
title: React Query 도입 후기
author: 김선미
---

안녕하세요. 사람인 FE개발2팀 김선미입니다.

'멘토링매치'는 회사나 직무 등 여러가지 궁금한 사항을 멘티와 멘토의 입장으로 일대일 음성 대화를 통해 경험과 정보를 나눌수 있는 서비스입니다.

이 글에서는 런칭 이후 진행된 리팩토링 작업중 react-query 도입과정에 대한 소개를 해볼까합니다.


## 리액트 쿼리란?

React-Query는 데이터를 불러오고 캐싱하며, 서버 데이터와의 동기화 및 업데이트 하는 작업을 개발자가 쉽고 간단하게 할 수 있도록 도와주는 라이브러리입니다. 즉, '비동기 로직을 쉽게 다룰수 있게 해준다' 라고 이해하면 될 것 같습니다.



## 장단점 및 핵심기능

- 데이터 캐싱 기능
- 동일한 데이터에 대한 중복 요청을 제거
- 백그라운드에서 "오래된" 데이터 업데이트
- 데이터 업데이트를 최대한 신속하게 반영
- 페이지네이션 및 데이터 지연 로드와 같은 성능 최적화
- 서버 상태의 메모리 및 가비지 수집 관리
- 구조적 공유로 쿼리 결과 메모하기

참고로 위에 있는 기능중 데이터 캐싱 기능을 잘 사용하기 위해서는 리패치가 일어나는 조건과 옵션 두가지에 대한 이해도가 필요합니다.

**refetch가 일어나는 조건**

- refetchOnWindowFocus : 윈도우에 포커스 된 경우
- refetchOnMount : 마운트 될때
- refetchOnReconnect : 재연결 될 때

기본적으로 React Query는 위의 세가지 기능의 기본값은 모두 `true` 상태입니다. 이외에도 queryKey와 함께 State값을 같이 넘겨준 경우 State값이 변경된다면 `refetch`가 일어나게 됩니다.


**staleTime**

`fresh`한 데이터가 `stale`한 데이터로 변화되는 시간을 말합니다. 즉, 데이터의 유통기한이라고 생각하면 될것 같습니다. `staleTime`에 대한 기본 옵션은 0이고, 이 외에도 다른 옵션을 설정하지 않았다면 호출 즉시 `stale`한 상태로 변하게 되고, `refetch`가 일어나는 조건과 일치하게되면 데이터가 패치됩니다.


**cacheTime**

데이터가 `inactive`상태일 때를 기준으로 캐싱된 상태로 남아있는 시간입니다.

`staleTime`과는 별개로 기준 시점으로부터 데이터의 삭제가 결정되고, `cacheTime`이 지나면 가비지 콜렉터로 수집됩니다.



## 실제 사용해보기

멘토링 서비스를 개선하면서 관심가던 기능은 캐싱말고도 페이지네이션 기능이었는데, 마침 담당하고 있던 화면이 검색 조건이 다양한 더보기 버튼이 있는 리스트 화면이었기 때문에 적용해보면 아주 좋을것 같아 도입을 결정하게 되었습니다.


### 설치

`npm i @tanstack/react-query @tanstack/react-query-devtools`


### \_app.jsx 세팅

```react
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

// React Query 기본 옵션 설정
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      refetchOnWindowFocus: false, // 윈도우가 다시 포커스되었을때 데이터를 호출할 것인지
      retry: 0, // API 요청 실패시 재시도 하는 옵션 (설정값 만큼 재시도)
    },
  },
});

export default function MyApp({ Component, pageProps }) {
   return (
        <>
            <QueryClientProvider client={queryClient}>
                <Component {...pageProps} />
                <ReactQueryDevtools initialIsOpen={false} />
            </QueryClientProvider>
        </>
	);
}
```


## GET 방식의 데이터 호출시, useQuery

useQuery는 React Query에서 제공하는 GET방식의 데이터를 호출시 사용하는 함수입니다.

멘토링 서비스에서는 직무나 경력등의 코드들을 조회하기 위한 공통 코드에 대한 정보 들을 호출하여 사용하고 있는데, 이 데이터들은 한번 불러온 후 오랫동안 어딘가에 저장해놓고 사용할 데이터(변화되지 않을 데이터)였기때문에 `staleTime`이라는 옵션을 `Infinity`로 주면 꽤 유용할것 같아 사용해보기로 했습니다.


### 기본적인 사용방법

사용 방법은 아래 코드와 같고, 데이터 호출시 유용한 여러 상태들의 값을 가지고 올 수 있습니다.

```react
import { useQuery } from '@tanstack/react-query';

const { data, isLoading, isError, ... } = useQuery(queryKey, queryFn, options)
```

[useQuery Docs](https://tanstack.com/query/v4/docs/react/reference/useQuery)


### 실제 적용한 코드

useQuery로 데이터를 호출한뒤 React Query의 개발자도구를 열어 보면 항상 `fresh`한 상태인걸 알수있습니다. 리패치의 전제조건은 데이터가 `stale`한 상태여야 하기 때문에 현 상태에서는 `cacheTime`이 지나더라도 `refetch`를 하지 않습니다.

```react
// 공통 컴포넌트 내부 (useSample.js)
import { useQuery } from '@tanstack/react-query'
const { data: cmmCode } = useQuery(['cmm-code'], fetchCmmCode, {
  staleTime: Infinity,
});
return {  ..., cmmCode }

// 실제 공통 코드를 필요로 하는 컴포넌트 내부에서 사용
import useSample from '@/hook/useSample'
export default function MentorPage() {
    const { cmmCode } = useSample();

    return (
    	<>
        	{cmmCode && <FilterGroup />}
        </>
    )
}
```


## 무한 스크롤 더보기, useInfiniteQuery

`useInfinityQuery ` 역시 React Query에서 제공하는 GET방식의 데이터를 호출시 사용하는 함수이며,  페이징 기능 구현시 사용합니다. 


### 기본적인 사용 방법

사용방법은 useQuery와 거의 비슷하지만 차이점을 정리하자면 아래와 같습니다.

- 두 개의 배열 속성을 포함하는 객체 {pageParams: [], pages: []}로 반환
- 이전/다음 페이지를 가져오기 위한 `fetchPreviousPage`, `fetchNextPage` 함수
- `hasNextPage`, `hasPreviousPage`, `isFetchingNextPage`, `isFetchingPreviousPage` 등의 속성으로 이전/다음 페이지가 있는지를 확인하고 이전/다음 페이지를 가져올때 확인 할 수 있습니다.

```react
import { useInfiniteQuery } from '@tanstack/react-query';

useInfiniteQuery({
  queryKey,
  queryFn: ({ pageParam = 1 }) => fetchPage(pageParam),
  ...options,
  getNextPageParam: (lastPage, allPages) => lastPage.nextCursor,
  getPreviousPageParam: (firstPage, allPages) => firstPage.prevCursor,
})
```

[useInfiniteQuery Docs](https://tanstack.com/query/v4/docs/react/reference/useInfiniteQuery)


그리고 `hasNextPage`라는 값을  잘 사용하기 위한 `pageParam`과 `getNextPageParam` 에 대해 잠시 정리하자면 아래와 같습니다.


__pageParam__

`useInfinityQuery`를 사용하게 되면 기본적으로 `pageParam`이라는 값을 제공해주고, 첫 페이지는 기본적으로  0 이라는 값이 제공되고, 이후에는 개발자가 계산하여 사용해야합니다.


__getNextPageParam__

요청이 완료된 후 최신 응답 값으로 몇 페이지 인지 계산해서 반환해주는 옵션입니다. 값을 반환을 해주지 않게되면 `hasNextPage`는 계속해서 `undefined` 상태이므로 활용을 할 수 없게 됩니다. 

다음 페이지가 있다면 pageParam + 1을 해주고, 없다면 undefined를 반환해주면 됩니다. 그럼 `hasNextPage`의 값은 `true` 또는 `false`가 됩니다.


### 실제 적용한 코드

검색 키워드, 직무, 기업, 경력 등의 필터 및 정렬 등 다양한 조건들이 있는 멘토 리스트 화면입니다.

항상 최신의 데이터를 보여주면 어떨까 하다가 생각해보니 검색 조건에 따라 변경될 상황도 꽤나 많았기 때문에 `staleTime`은 1분 정도로만 주었습니다.

그리고 저의 경우엔 API호출시 서버에서 내려주는 결과값중 count와 pageSize의 값을 활용하여 nextPage 의 여부를 판단하였고 undefined 또는 pageParam + 1을 미리 반환해주어 `getNextPageParam` 함수에서는 받은 값을 그대로 돌려주기만 했습니다. 

이후 hasNextPage의 상태값에 따라 UI를 컨트롤 할 수 있었습니다.

```react
import { useInfiniteQuery } from '@tanstack/react-query';
import { getMentorList } from '@/apis/search';

export default function MentorPage() {
    /* 멘토 리스트 조회 */
    const fetchMentorList = async ({ pageParam = 1 }) => {
        const res = await getMentorList({ ...params, page: pageParam });

        if (res.status === 200) {
            const { count, mentors } = res.data.result;
            const isLast = count / params.pageSize <= pageParam;

            return {
                count: count,
                items: mentors,
                nextPage: isLast ? undefined : pageParam + 1,
            };
        }
    };

    const { isLoading, data, hasNextPage, fetchNextPage, isFetchingNextPage } =
        useInfiniteQuery([queryKey, params], fetchMentorList, {
        staleTime: 60 * 1000,
        getNextPageParam: (lastPage) => lastPage.nextPage,
    });

    return (
        <>
            // 리스트 하단 더보기 버튼
            {hasNextPage && <ListMoreButton />}
        </>
    )
}
```

## 업데이트, useMutation

useQuery가 GET방식의 데이터를 조회하는거라면, POST, PUT, DELETE 등 데이터를 업데이트하기 위한 방법으로는 useMutation을 사용합니다.


### 기본적인 사용방법

사용방법은 useQuery, useInfinityQuery와 동일하고, 업데이트를 하는 방법으로는 아래 2가지가 있습니다.

```react
const { ..., data, } = useMutation({ ..., mutationFn })
```

[useMutation Docs](https://tanstack.com/query/v4/docs/react/reference/useMutation)


**invalidateQueries**

기존에 캐싱되어 있는 데이터를 무효화하는 함수입니다. 기존의 데이터를 무효화하게 되면 `refetch`가 일어나면서 업데이트 됩니다. 이때 조심해야 할 것은 리스트의 갯수가 많다면 리패치하는 시간이 소요되기때문에 사용자는 업데이트 되는데 느리다고 생각할 수 있습니다.


**실제 적용한 코드**

멘토링 서비스에는 멘토와 멘티가 음성대화를 하면서 간단하게 메모를 작성 할 수 있는 기능이 있는데, 이후 작성한 메모들을 선택해서 수정할 수 있게 구현된 화면이 있습니다. 다른 리스트 화면들에 비해 리스트의 갯수가 많지 않을것 같아 적용해보았습니다.

mutate를 실행하여 성공하면(메모를 성공적으로 수정하면) `invalidateQuries`로 기존 데이터를 무효화 시켜 메모 리스트를 리패치하여 업데이트 하는 로직입니다.

```react
import { useMutation } from '@tanstack/react-query';

const { mutate } = useMutation(updateMemo, {
  onSuccess: () => queryClient.invalidateQuries(['memo-list'])
})
```


**setQueryData**

`invalidateQuries`가 데이터 자체를 무효화 시켜 리패치를 일어나게 하는거라면, `setQueryData`는 쿼리 데이터를 수동으로 설정할 수 있습니다. 리패치를 일으켜 네트워크 요청을 기다릴 필요 없이 내가 원하는 데이터로 수동으로 변경 할 수 있다라는 얘기입니다. 서버와 실시간 동기화하는것은 아니지만 사용자 입장에선 내가 원하는 화면이 이벤트 실행 직후 빠르게 반영되는것으로 보여지기 때문에 아주 좋은 경험을 줄 수 있습니다.

### 실제 적용한 코드

메모 리스트 화면 말고도 멘토 리스트, 기업 리스트 등 다양한 리스트 화면이 존재하는데 대부분 북마크 기능을 가지고 있고 리스트의 갯수도 아주 많습니다. 그래서 부분적으로 수동 업데이트가 가능한 `setQueryData`를 활용해보기로 했습니다.

마찬가지로 mutate를 실행시켜 성공하면(북마크를 성공적으로 추가/삭제하면) `getQueryData`로 기존 데이터를 가져와서 부분적으로 업데이트를 한 새로운 데이터를 만든후 `setQueryData`로 재설정해줍니다. 그럼 리패치 없이 변화된 데이터가 즉시 반영되게 됩니다.

```react
/* 실제 데이터를 가져와서 부분 업데이트 */
const update = (data, variables, lists) => {
    const { pages, pageParams } = lists;
    let page = pages[data.pageKey];
    const key = page.lists.findIndex(
        (list) => list.mentorMemIdx === variables.idx
    );

    page.lists[key].likeMentor = !page.lists[key].likeMentor;

    return {
        pageParams,
        pages,
    };
};

/* mutate 성공시 수동 업데이트 */
const { mutate } = useMutation(toggle, {
    onSuccess: (data, variables) => {
        const lists = queryClient.getQueryData(["mentor-list"]);
        return queryClient.setQueryData(
            ["mentor-list"],
            update(data, variables, lists)
        );
    },
});
```

## 마무리

기존에는 데이터를 호출시 결과값을 담아놓을 상태값을 만들고 필요에 따라 데이터가 존재하는지 판단을 하여 isLoading 등의 상태값 또한 만들어줘어야 했습니다. 그리고 useEffect를 통해 패치를 하는등 귀찮은 작업들을 일일이 해줬어야 했는데 React Query를 도입하면서 코드수가 줄어들고 깔끔하게 정리된것 같아 좋았고, 사용법 또한 아주 간단해서 도입하는데 까다롭진 않았던것 같습니다.

그리고 `onSuccess`, `isLoading` , `isError` 등의 다양한 플래그들을 지원해줘서 필요할때마다 유용하게 쓸수 있었습니다.

도입해보니 장점이 꽤 많은것 같아 좀 더 깊게 학습해서 잘 사용해보면 좋을것 같습니다.

감사합니다.
