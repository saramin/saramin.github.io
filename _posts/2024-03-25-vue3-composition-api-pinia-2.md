---
title: Vue3, Composition API와 Pinia를 이용한 상태관리 (2)
layout: post
author: 노혜민
---

안녕하세요. 사람인 개발팀 노혜민입니다.

이번 포스팅은 [Vue3, Composition API와 Pinia를 이용한 상태관리 (1)]({{ site.url }}/2023-06-27-vue3-composition-api-pinia-1/){: target="_blank" } 글의 후편입니다. 
이전 포스팅에서 Composition API, Pinia에 대한 이론적인 설명을 다루었다면 ***이번 포스팅에서는 실제로 Pinia를 어떤 방식으로 적용했고 어떤 작업 결과를 냈는지 다루려합니다.*** 
 
글의 목차는 아래와 같습니다.   
03\. 적용 결과: 인재풀에서의 Pinia  
04\. 느낀점: 이론적 장단점과 내가 느낀 실제  

Vue3가 적용된 `인재풀`서비스 런칭으로 약 1년 정도의 시간이 지나면서 당시 프로젝트 작업자 외에 많은 팀원들이 Vue3를 직접 경험할 수 있었습니다. 프로젝트 작업시의 관점 뿐만 아니라 서비스 운영, 유지보수 관점에서 느낀 `Vue3`, `Composition API`, `Pinia`의 장단점에 대해 이야기하고자 합니다.  

컴포넌트 구성 방식을 좌우하는 주요 기능인 `Composition API`와 `<script setup>`에 대해 설명하고 본격적으로 `Pinia`가 무엇인지  마지막으로 실제 사람인 인재풀에서는 어떤 부분에 어떻게 적용되었고 그 과정을 통해 느낀점을 이야기해보겠습니다.

# 3. 적용 결과: 인재풀에서의 Pinia

설명에 앞서 사람인 인재풀 서비스에 대한 이해가 필요합니다. 인재풀은 기업 회원에게 제공되고 있는 서비스로 아래 화면과 같이 좌측 검색 필터를 활용해 원하는 인재를 검색할 수 있는 서비스입니다. 사용자가 원하는 이력서 항목 조건을 설정하면 그에 맞는 인재 결과를 제공해주고 있습니다.
[<img src="{{site.url}}/img/vue3/talentpool_screenshot.png" />]({{site.url}}/img/vue3/talentpool_screenshot.png)  

인재풀 검색 필터 기능은 경력, 지역, 직무, 학력, 연봉, ... 등 총 17가지 항목의 조건 변경이 가능합니다. 다양한 세부 항목이 존재하는 만큼 필요로하는 데이터가 복잡하며 데이터 변경이 잦은 영역입니다. 이러한 기능적 특징 때문에 검색 필터 영역을 단일 컴포넌트로 관리할 경우 코드 복잡도가 올라갈 수 있는 상황이었습니다. 따라서 검색 필터 영역은 `Filter`라는 상위 컴포넌트 아래 항목 단위로 분리된 자식 컴포넌트를 갖도록 구성했습니다.
[<img src="{{site.url}}/img/vue3/talentpool_component.png" />]({{site.url}}/img/vue3/talentpool_component.png)  

인재 검색 기능은 아래와 같이 동작하게 됩니다.
[<img src="{{site.url}}/img/vue3/talentpool_search.png" />]({{site.url}}/img/vue3/talentpool_search.png)

모든 검색 필터 조건 정보를 담은 JSON 형태의 데이터가 인재 검색 API 요청 파라미터 정보로 활용됩니다. 검색 필터의 세부 항목에서 변경이 발생하게 되면 해당 항목 컴포넌트 내부에서 검색 필터 데이터 변경, 검색 리스트 API 재호출이 순차적으로 수행되게 됩니다. 이때 필터 데이터 변경, API 재호출 동작은 각기 다른 컴포넌트 내부 동작에 의해 발생하지만 최종적으로 활용되는 데이터는 검색 필터 조건 정보를 담은 공통의 데이터 입니다.  

활용되는 필터 데이터는 공통의 데이터이기 때문에 어떤 컴포넌트에서 해당 데이터를 참조하던 동일한 상태여야합니다. 활용되는 공통 데이터의 구조가 복잡하다는 점과 각기 다른 컴포넌트에서 공통의 데이터 상태가 공유되는 프로젝트 특성상 상태관리 라이브러리 활용의 장점을 경험하기에 적합했고 `Pinia`라이브러리 적용을 결정하게 되었습니다. (물론 검색 필터 기능 외에 다른 기능을 위해서도 스토어를 정의하며 상태 관리 라이브러리를 활용했습니다.)   

이번 프로젝트에서는 모든 작업자들이 `Pinia`를 처음 접하는 상황이었기에 공식 문서를 적극 활용해 공식 문서에서 제공해주는 기본 구조를 기반으로 *`store`를 생성해주었습니다.   

> 스토어란?   
> 스토어는 컴포넌트 트리에 바인딩되지 않은 상태 및 처리해야 할 일의 로직을 가지는 독립적인 것입니다.  즉, 전역 상태를 호스팅합니다. 항상 존재하고 모두가 읽고 쓸 수 있는 컴포넌트와 비슷합니다. state, getters, actions라는 세 가지 개념이 있으며, 이러한 개념은 컴포넌트의 data, computed, methods와 동일하다고 가정해도 무방합니다.   

실제 프로젝트 적용 예시를 인재풀 프로젝트 일부를 통해 설명하겠습니다.   

인재풀 검색 기능을 위해 정의된 검색 조건 스토어 코드 일부입니다.   
```vue
export const useSearchConditionsStore = defineStore("searchConditions", () => {
      // ref()는 state 속성
      const conditions = ref(deepCopy(defaultFilterState));
 
      // getters
      const getFilters = computed(() => conditions.value);

      // actions
      const updateFilters = (filters) => {};
 
      const resetFilters = () => {};

      return {
          conditions,
          getFilters,
          updateFilters,
          resetFilters
      };
});
```
스토어를 정의하는 문법에는 `옵션 스토어` 방식과 `셋업 스토어`방식이 있습니다. 프로젝트 전반적으로 `Composition API`를 활용하고 있기 때문에 `Composition API`의 `setup` 함수와 유사한 형태인 `셋업 스토어` 방식을 활용해 스토어를 정의해주었습니다.   

이렇게 정의된 스토어의 상태값은 각각의 필터 항목 컴포넌트에서 접근 가능하며 어느 곳에서 접근해도 동일한 상태를 유지하는 공통의 상태 값이 됩니다. 사용 예시로 경력 필터 영역을 살펴보겠습니다. 
```vue
<script setup>
import { storeToRefs } from "pinia";
import { useSearchConditionsStore } from "@/stores/modules/SearchConditions";

const searchCondition = useSearchConditionsStore();
const { getFilters } = storeToRefs(searchCondition);
</script>

<template>
      <div class="talent_filter_item">
           <span class="talent_filter_tit">경력</span>
           <div class="filter_input">
              <div class="InpBox SizeS from_to">
                 <select
                     id="career_min"
                     name="career_min"
                     v-model="getFilters.career_min"
										 >
                        <option :value="''">선택</option>
                        <option :value="0">신입</option>
                        <option :value="n" v-for="n in 20">{{ n }}년 이상</option>
                    </select>
            </div>
            <span class="txt_from_to">~</span>
            <div class="InpBox SizeS from_to">
                <select
                    id="career_max"
                    name="career_max"
                    v-model="getFilters.career_max"
                >
                    <option :value="''">선택</option>
                    <option :value="0">신입</option>
                    <option :value="n" v-for="n in 20">{{ n }}년 이하</option>
                </select>
            </div>
        </div>
    </div>
</template>
```

경력 구간 입력을 위한 셀렉트 박스에 검색조건 상태값을 양방향 바인딩해주면 셀렉트 박스 동작이 발생했을 때 검색 조건 스토어 상태값에 변경을 발생시킬 수 있습니다. 
그리고 변경이 발생할 때 마다 변경된 조건에 맞게 검색 리스트가 재호출 되어야 합니다.  

아래는 검색 조건 상태 변화를 감지해 결과 리스트를 재호출하고 리스트 결과를 렌더해주는 코드 일부입니다.  

```vue
<script setup>
import { ref, watch } from "vue";
import { useSearchConditionsStore } from "@/stores/modules/SearchConditions";

let recommendList = ref(null);

// store 선언
const conditionsStore = useSearchConditionsStore();

// 반응형 상태
const { getFilters } = conditionsStore;

const getSearchTalentList = async () => {
     recommendList.value = await conditionsStore.getSearchList(makeParams());
};

// 필터값 변경 감지
watch(getFilters, () => {
     // API 호출
     getSearchTalentList();
 },
 { deep: true } // 해당 개체의 모든 중첩 속성을 탐색
);
</script>

<template>
     <div class="talent_list_wrap">
          <div class="talent_list">
          <!-- 리스트 or empty case -->
          <List.TalentList v-if="recommendList?.length > 0" :talentList="recommendList" />
          <List.NoResult v-else />
          </div>
     </div>
</template>
```
`watch`함수를 통해 검색 조건 상태의 변경을 감지합니다. 이때 검색 조건의 데이터 구조가 중첩된 속성을 갖고 있기 때문에 내부 값의 변경까지 감지해내기 위해서 deep 속성을 활용합니다.   

간단한 코드 예시로 전체적인 검색조건 변경과 그로 인한 리스트 재호출 과정을 저희 프로젝트에 어떻게 구현했는지 살펴보았습니다. 
상태 관리 라이브러리의 활용으로 각기 다른 컴포넌트에서도 공통 데이터에 손 쉽게 접근 할 수 있었으며 발생하는 이벤트, 데이터의 변화에 의한 이후 로직을 간단하게 구현할 수 있었습니다.  
 
※ 위 예시 코드들은 간단한 설명을 위해 상태 관리 라이브러리 사용과 관련된 기본적인 흐름이 보이는 코드의 일부분만 발췌한 것이라는 점 참고부탁드립니다.
 
# 4.  느낀점: 이론적 장단점과 내가 느낀 실제
`Pinia`의 공식 문서에서 크게 다섯가지 특징이 있다고  말합니다. 앞선 포스팅에서도 `Pinia`의 주요 특징에 대해 언급한 바 있습니다.  
* 가볍고 직관적인 API
* TypeScript와의 호환성
* 컴포넌트별 상태 관리로 상태관리 편리성
* 유연한 상태 관리
* 성능 최적화에 용이함

공식 문서에서 말하는 특징들을 실제 작업을 진행하면서도 느낄 수 있었습니다.  저희 프로젝트에서는 아쉽게도 TypeScript를 활용하고 있지 않아 두번째 특징을 느낄 순 없었지만 그 외의 주요 특징에 대해서는 모두 공감할 수 있었습니다.   

그러나 프로젝트 초기 작업자의 입장에서는 사용상의 단점을 크게 느끼지 못했습니다. 아마  無의 상태에서 나름의 학습 결과들을 가지고 프로젝트를 쌓아 올려간 것이기에 사용상의 장점이 더 크게 와닿았기 때문이라고 생각합니다. 프로젝트 런칭 이후 약 1년 가량의 시간이 지나면서 초기 작업자 외 팀원들도 Vue3, Pinia를 학습하고 사용하게 되었습니다. 
프로젝트 초기 작업자의 관점이 아닌 프로젝트 유지보수 관점에서의 팀원분들의 Vue3 사용 경험을 조사해보았습니다.  
## 4-1. 프로젝트 런칭 후 작업자들의 작업 소감
총 네가지 항목에 대한 작업자들의 의견을 조사했습니다.  	

1. Vue2와 다른 Vue3만의 특징 적용에 어려움이 있었나요?
2. Composition API 방식에 대해 어떻게 생각하시나요?
3. Pinia 상태관리 라이브러리 적용에 대해 어떻게 생각하시나요?
4. Vue3 사용 및 인재풀 프로젝트에 대한 생각  

인재풀과 관련된 작업을 진행한 경험이 있는 7명의 작업자분들이 답변을 해주셨습니다.  

#### 1. Vue2와 다른 Vue3만의 특징 적용에 어려움이 있었나요?
약 70%의 작업자가 '적응하는 것에 큰 어려움이 없었다' 라고 답했으며 각 15%의 작업자가 '적응하는데 시간이 걸렸다', 'Vue2 경험이 없어 잘 모르겠다'라고 답했습니다.

#### 2. Composition API 방식에 대해 어떻게 생각하시나요?
약 70%의 작업자가 'Composition API가 Options API 방식보다 좋다'고 답했으며 나머지 30% 작업자가 '비슷한 것 같다', '잘 모르겠다'고 답했습니다.

#### 3. Pinia 상태관리 라이브러리 적용에 대해 어떻게 생각하시나요?
약 70% 작업자가 'Vuex보다 쉽게 느껴진다'고 답했으며 각 15%의 작업자가 'Pinia가 react의 상태관리 툴과 비슷하다', '잘 모르겠다'고 답했습니다.

#### 4. Vue3 사용 및 인재풀 프로젝트에 대한 생각
설문 응답에 공통적으로 언급된 장단점은 아래와 같습니다.   
#### 장점
- 코드의 가독성이 좋아짐
- `Composition API` 사용으로 인해 코드 작성이 간결해짐
- `SFC` 사용으로 컴포넌트 정의가 명확해짐
- 코드 재사용성 증가
- 데이터 처리의 편리성 (반응형 데이터 처리, 전역 데이터 처리, 양방향 통신)
- `<Teleport>`를 활용한 간편한 모달 기능 구현

#### 단점
- Vue3에 익숙해지기까지 시간이 필요
- Vue3의 최대 이점 중 하나인 Typescript 미사용

단순히 Vue3 사용 뿐만 아니라 Composition API, Pinia 사용으로 느낀점, 새롭게 구성된 인재풀 프로젝트 자체에 대해 느낀점을 응답해주셨습니다.   

이런 공통적인 장단점 외에 프로젝트에 대한 피드백 내용들도 있었습니다.   
#### 개선점
- 불필요한 상태 변경, 동작 발생 부분에 대한 개선 
- 코드 스타일 통일 (`Composition API`, `Options API` 사용 혼재)
- `watch` 사용으로 인한 중복 api 호출 개선
- axios 대신 vue-query 도입

프로젝트 초기 작업자들 또한 새롭게 Vue3를 학습하고 적용했기 때문에 상태관리와 프로젝트 구조에 대한 이해가 부족한 부분이 있지 않았나 싶습니다. 
또한 프로젝트 런칭 이후 새로 적용된 기술에 대한 공유가 부족해 `Composition API` 적용에 어려움이 있어 코드 스타일이 혼재된 것으로 보여 아쉬운 부분입니다. 
프로젝트 런칭 후 시간이 지나면서 저도 같은 부분에 대한 개선이 필요하다고 느꼈기 때문에 개선점들은 차차 해결해 나가도록 하겠습니다.  

위 설문 내용은 Vue3를 사용할 때 뿐만 아니라 프로젝트 내에 신기술 적용에 앞서 고려할 부분에 대해 참고할 자료가 될 수 있을것이라 생각이 듭니다. 
특히 혼재된 코드 스타일 문제가 신기술 적용 전 고려할 주요한 점 한가지를 일깨워주는 것 같습니다. 신기술 적용 후 적용된 기술에 대한 문서화 혹은 기술 전파 과정이 미흡할 때 이와 같은 문제가 발생할 수 있다고 생각합니다. 새로운 기술 도입에 앞서 런칭 후 어떤식으로 기술을 전파할지에 대해서 미리 고려해 둔다면 신기술 적용 후 더 안정적인 프로젝트 유지 보수가 가능할 것 같습니다. 

## 4-2. 개인적 소감
2022년 11월 인재풀 서비스 개편 프로젝트를 계기로 Vue3를 처음 사람인 서비스에 적용하게 되었습니다. 
Vue.js로 구성된 기존 운영 서비스들은 모두 Vue2를 기반으로 하고 있어 그동안 Vue3를 실무에 적용할 기회가 없었습니다. 
Vue3를 처음 적용하는 프로젝트인 만큼 Vue3가 갖는 특장점을 최대한 활용해보자는 작업자들간의 공통된 목표로 `Composition API`, `Pinia`, `Vite`를 적용해 작업을 진행했습니다. 

공식 문서를 제외하고 `Pinia`에 대한 참고 자료가 많지 않아 어려움이 많았지만 `Pinia`를 사용한 상태 관리 경험은 꽤나 의미있었습니다. 
`Vuex`에 익숙해져있었기 때문에 작업 초반에는 낯설게 느껴졌으나 `Pinia` 활용에 충분히 익숙해진 뒤에는 `Pinia`의 장점을 잘 느낄 수 있었습니다. 
`Pinia`의 장점 중 하나인 직관적인 코드 구조덕에 기술을 익히고 활용하기까지 시간이 오래 걸리지 않았고 개인적으로 `Pinia`에 적응한 후에는 `Vuex`를 익히고 적응할 때 보다 작업 효율이 향상 되었다고 느꼈습니다. 
이 부분은 개인적 취향이기 때문에 저와 잘 맞는 라이브러리라고 느꼈습니다. 그리고 `Pinia`의 함수 기반 컴포넌트 단위 구성 또한 개인적으로 `Vuex`보다 협업자들간의 작업 파악에 더 용이하다고 느꼈습니다. 
이론적으로 말하는 실제 장점을 프로젝트 진행 과정 중에 충분히 느낄 수 있었습니다. 아무래도 프로젝트 초기 구축부터 작업에 참여해와서인지 사용상의 단점보다 장점들이 더 크게 와닿았던 듯 합니다. 

이번 Vue3 적용기에는 다루지 못했으나 장점 중 하나로 언급된 `<Teleport>`기능을 통해 모달 기능 구현에 있어 작업 효율이 매우 향상되었습니다. 
간단하면서도 유용한 이 기능에 대해 다음 포스팅에서 다루면 좋을 것 같다는 생각이 듭니다. 

실제 프로젝트보다 매우 간략하게 요약된 예시 코드 안에서 적용 사례를 설명하다보니 부족한 부분이 있을 수 있지만, 그럼에도 불구하고 긴 글 읽어주시며 저희의 Vue3 적용기에 관심 가져주셔서 감사합니다.  

> 참고문헌
> * [Vue.js 공식 문서](https://v3-docs.vuejs-korea.org/guide/introduction.html)
> * [Pinia 공식 문서](https://pinia.vuejs.kr/introduction)
