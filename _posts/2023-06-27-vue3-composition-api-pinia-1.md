---
title: Vue3, Composition API와 Pinia를 이용한 상태관리 (1)
layout: post
comments: true
social-share: true
show-avatar: true
author: 노혜민
---

안녕하세요. 사람인 개발팀 노혜민입니다.

이번 포스팅에서는 Vue3의 `Composition API`와 `Pinia`를 활용한 상태관리 경험을 공유하고자 합니다.

Vue3 릴리즈 이후 Vue.js를 활용한 프로젝트 진행 기회가 없어 Vue3를 실제 업무에 적용할 기회가 없었으나 인재풀 프로젝트 개편과 함께 Vue3를 실무에 적용하게 되었습니다.  프로젝트를 진행하며 Vue3의 주요 기능들을 적극적으로 활용해보기 위해 노력했고  상태관리에는 `Pinia`를 사용하는 새로운 도전을 하기도 했습니다.  

험난했던 프로젝트 개발 과정 중에 `Pinia`와 관련된 내용은 추후 기술 블로그에 공유되면 좋겠다는 생각을 했고 이렇게 포스팅을 쓰게 되었습니다!

총 4가지의 소주제를 순차적으로 이야기해보고자 합니다.

01. Vue3의 새로운 기능: Composition API
02. 상태관리 도구: Pinia
03. 적용 결과: 인재풀에서의 Pinia
04. 느낀점: 이론적 장단점과 내가 느낀 실제

컴포넌트 구성 방식을 좌우하는 주요 기능인 `Composition API`와 `<script setup>`에 대해 설명하고 본격적으로 `Pinia`가 무엇인지  마지막으로 실제 사람인 인재풀에서는 어떤 부분에 어떻게 적용되었고 그 과정을 통해 느낀점을 이야기해보겠습니다.

이론적인 설명과 실제 프로젝트 적용기까지 꽤나 긴 글이 될 것으로 보여 총 2개의 시리즈로 포스팅을 작성할 예정입니다.

***
# 01. Vue3의 새로운 기능: Composition API
## 1-1. Composition API
`Composition API`는 Vue3에서 도입된 새로운 API로 Vue.js 애플리케이션의 코드 구성과 재사용성을 향상시키는 기능을 제공합니다.
`Composition API`의 추가로 Vue.js는 총 두가지 방식으로 코드 구성이 가능해졌습니다. 
기존에 활용되던 `Options API`도 여전히 사용가능하기 때문에 선호하는 방식을 선택해 코드를 구성할 수 있습니다.

간단한 예시로 `Composition API`에 대해 살펴봅시다!  
아래는 `Composition API`방식으로 작성한 간단한 Count 증가 버튼 이벤트 예시입니다.
```vue
<template>
  <div>
    <button @click="increment">Increment</button>
    <p>Count: {{ count }}</p>
  </div>
</template>

<script>
import { ref } from 'vue';

export default {
  setup() {
    // ref를 사용하여 반응형 데이터를 정의합니다.
    const count = ref(0);

    // 이벤트 핸들러 함수를 정의합니다.
    const increment = () => {
      count.value++;
    };

    // 반응형 데이터와 이벤트 핸들러를 반환합니다.
    return {
      count,
      increment
    };
  }
};
</script>
```
`Composition API` 방식은 `setup` 함수를 사용해 컴포넌트의 로직을 구성합니다.   

`setup` 함수는 **반응형 데이터와 이벤트 핸들러를 별도로 정의, 객체로 반환**하고 반환된 객체의 속성들은 탬플릿에서 사용 가능합니다.  **반응형 데이터와 로직을 분리해 관리 가능한 구조**이므로 **재사용성**이 뛰어나고  구조적인 코드 작성이 가능해 **가독성이 향상**된다는 특징이 있습니다.   

위의 예시만으로도 `Options API`와의 차이가 느껴질 겁니다.  
`Options API`에서 정의하던 `data`, `methods`, `computed`, `watch` 등의 속성들이 사라졌기 때문입니다.

위의 Count 예시를 `Options API` 방식으로 구현하면 아래와 같습니다.
```vue
<script>
export default {
  data() {
    return {
      count: 0
    };
  },
  methods: {
    increment() {
      this.count++;
    }
  }
};
</script>
```
`Composition API` 방식이 `Options API` 방식보다 재사용성 측면에서 유리하고 더욱 직관적이며 유연하다는것을 볼 수 있습니다.

항상 `Composition API` 방식만이 좋은것은 아닙니다.  
`Composition API`는 Vue3에서 도입된 새로운 개념이기 때문에 기존 `Options API`에 익숙한 개발자들은 학습을 위한 시간 투자가 필요하며  새로운 개념인 만큼 학습 리소스, 커뮤니티 지원이 부족할 수 있습니다.  또한 Vue2와의 하위 호환성이 없어 기존 Vue2 사용 프로젝트에서의 Vue3 전환이 필요한 경우 많은 리소스를 필요로 합니다.  

이러한 단점도 있기 때문에 프로젝트 상황을 고려해 적용했을 경우 장점을 효율적으로 사용할 수 있을것입니다.

## 1-2. script setup
`<script setup>`은 `Composition API`와 마찬가지로 Vue3에 추가된 신규 기능입니다.

싱글 파일 컴포넌트(SFC, Single-File Components)내에서 Composition API를 사용하기 위해 필요한 문법으로 Vue 컴포넌트의 `setup` 함수를 **더 간단하게** 작성할 수 있도록 도와주는 기능을 합니다. Vue.js 공식문서에서도 SFC와 Composition API를 함께 사용하는 경우 `<script setup>`을 사용하라고 권장하고 있습니다.   
  
아래는 위에서 보았던 `Composition API` 예시에 `<script setup>`방식을 적용한 예시입니다.
```vue
<template>
  <div>
    <button @click="increment">Increment</button>
    <p>Count: {{ count }}</p>
  </div>
</template>

<script setup> // script setup 적용
import { ref } from 'vue';

const count = ref(0);

const increment = () => {
  count.value++;
};
</script>
```
별도의 `setup`함수 정의와 객체 반환이 생략되고 단순히 `<script setup>`을 사용하여 컴포넌트 로직을 구성합니다.   

`Composition API` 적용만으로도 코드의 가독성을 향상시키는 효과를 얻을 수 있지만,  `<script setup>` 방식을 함께 적용하면 **더욱 간결한 코드 작성이 가능해집니다.**

`<script setup>`방식은 코드가 간결해진다는 큰 장점이 있지만, **Vue3 이상의 버전에서만 사용할 수 있고 일부 기능은 제한될 수 있다는 단점도 있습니다.**

제한되는 기능을 대체할 수 있는 방법들이 있긴 하지만 프로젝트 적용 전 이런 제한 사항과 대체 방안을 사전에 숙지할 필요가 있습니다. 
또한 `Composition API`와 마찬가지로 새로운 개념인 만큼 학습 리소스, 커뮤니티 지원이 부족할 수 있어 이 부분도 적용 선택 시 고려되어야 합니다.

## 1-3. script setup 제한 사항과 대체안
`<script setup>`을 사용할 경우 일부 기능에 제한 사항이 발생하게 됩니다.  
하지만 제한되는 기능을 대체할 수 있는 방안이 있기 때문에 크게 문제가 되지는 않습니다.

### 1. this 키워드 사용 불가
`<script setup>`은 컴포넌트 옵션 객체를 사용하지 않기 대문에 `this`키워드로 컴포넌트 인스턴스에 접근이 불가합니다.  대신 `ctx` 객체를 통해 컴포넌트 인스턴스에 접근할 수 있고 `ref`나 `toRef`를 사용해 데이터를 반응형으로 감싸 `value` 속성을 사용해 데이터에 접근할 수 있습니다.


### 2. data, computed, methods 등의 속성은 사용할 수 없습니다.
대신, ref, computed 함수, defineEmits, defineExpose 등의 기능을 사용하여 대체할 수 있습니다.  이때 컴포넌트 옵션들의 의존성이 순서에 의해 결정되기 때문에 옵션 정의 시에 아래 예시와 같은 순서를 따릅니다.

```vue
<script setup>
import { ref, computed, watch } from 'vue';

// 1. props 정의: 컴포넌트에 전달되는 프롭스
const props = defineProps({
  // props 옵션들...
});

// 2. 데이터 선언: ref, reactive 등을 사용하여 컴포넌트의 반응형 데이터 선언
const count = ref(0);

// 3. 컴포지션 함수: 컴포넌트의 로직을 재사용 가능한 컴포지션 함수로 추출, 재사용 가능
const doubleCount = computed(() => {
  return count.value * 2;
});

// 4. 컴포넌트 옵션: 컴포넌트 옵션을 직접 사용하지 않고 선언된 변수와 함수로 로직을 작성
// computed 옵션 대신 computed 함수, watch 속성 대신 watch 함수 ...
watch(count, (newValue, oldValue) => {
  console.log('Count changed:', newValue, oldValue);
});

// 5. 기타 코드: 필요에 따른 기타 로직, 함수 정의
// 필요한 추가 로직이나 함수 등...
</script>
```

### 3. 생명주기 훅을 직접 사용할 수 없습니다.
Composition API에서 사용되는 생명주기 훅에 대한 대체 함수와 그 기능을 간단히 설명한 표입니다.

| 옵션 API 생명주기 훅     | Composition API 대체 함수 | 설명                                  |
|-------------------|-----------------------|-------------------------------------|
| `beforeCreate`    | -                     | Composition API에서는 대체 함수가 없습니다.     |
| `created`         | `setup` 함수 내의 로직      | 컴포넌트의 인스턴스가 생성되고 초기화된 직후 실행됩니다.     |
| `beforeMount`     | `onBeforeMount`       | 컴포넌트가 DOM에 마운트되기 직전에 실행됩니다.         |
| `mounted`         | `onMounted`           | 컴포넌트가 DOM에 마운트된 직후 실행됩니다.           |
| `beforeUpdate`    | `onBeforeUpdate`      | 컴포넌트가 업데이트되기 직전에 실행됩니다.             |
| `updated`         | `onUpdated`           | 컴포넌트가 업데이트된 직후 실행됩니다.               |
| `beforeUnmount`   | `onBeforeUnmount`     | 컴포넌트가 언마운트되기 직전에 실행됩니다.             |
| `unmounted`       | `onUnmounted`         | 컴포넌트가 언마운트된 직후 실행됩니다.               |
| `errorCaptured`   | `onErrorCaptured`     | 컴포넌트 또는 자식 컴포넌트에서 발생한 에러를 처리합니다.    |
| `renderTracked`   | `onRenderTracked`     | 렌더링이 추적될 때 실행됩니다.                   |
| `renderTriggered` | `onRenderTriggered`   | 렌더링이 트리거될 때 실행됩니다.                  |
| `activated`       | `onActivated`         | `<keep-alive>` 컴포넌트가 활성화될 때 실행됩니다.  |
| `deactivated`     | `onDeactivated`       | `<keep-alive>` 컴포넌트가 비활성화될 때 실행됩니다. |

`Composition API`에서는 이러한 생명주기 훅을 대체하는 함수들을 제공하여 비슷한 동작을 구현할 수 있습니다.  
각 대체 함수는 `on` 접두사와 해당 생명주기 이름을 가지고 있습니다. 
이러한 대체 함수들을 사용하여 컴포넌트의 특정 시점에서 원하는 로직을 실행할 수 있습니다.

대체안들만 보았을 때 이게 어떻게 적용될지 감이 안올 수 있으나 다음 포스팅에 이어질 실제 적용 예시를 보면 조금 더 쉽게 이해될 수 있습니다!

*** 

인재풀에서는 Vue3의 주요 특징들을 최대한 활용해보고자 `Coposition API`, `<script setup>`로 컴포넌트를 구성했고  이런 구조와 함께 사용했을 때 더욱 이점을 갖는 `Pinia`를 상태관리 도구로 선택했습니다.   저희도 `Pinia` 사용이 처음이었기 때문에 낯설기도 했고 `Vuex`에 비해 참고할 수 있는 자료가 많지 않아 여러 시행착오를 겪기도 했습니다.  

`Pinia`가 무엇인지 어떤 특징을 갖는지 먼저 이야기해보겠습니다.  

# 02. 상태관리 도구: Pinia
`Pinia`는 2020년 7월 Vue.js 3의 beta 버전과 함께 출시 된 상태 관리 라이브러리입니다.  
가볍고 직관적이며 `TypeScript`와의 호환성이 뛰어나고 성능 최적화에 용이하다는 특징을 갖고 있습니다.

Vue3의 `Composition API`를 기반으로 하고 있어 Vue3와 함께 사용할 경우 `Vuex`와 비교해 여러 이점을 갖는 라이브러리입니다.

실제로 Vue의 창시자인 Evan You가 `Vuex`와 `Pinia`에 대한 비교를 제시하며 `Pinia` 사용을 추천하기도 했습니다.  (참고: 2020년 4월 Vue.js 공식 Stack 채널)

## 02-1. Pinia의 주요 특징
`Pinia`의 주요 특징은 다음과 같습니다.

### 1. 가볍고 직관적인 API ⭐
`Pinia`는 Vuex보다 더 가벼우며, 직관적인 API를 제공합니다.  
Vue.js 3의 Composition API와 함께 사용되어, 개발자가 상태 관리를 더욱 쉽게 할 수 있도록 도와줍니다.

### 2. TypeScript와의 호환성
`Pinia`는 TypeScript와 완전히 호환됩니다. 명시적인 타이핑을 사용하여 안정성과 코드 가독성을 높여줍니다.

### 3. 컴포넌트별 상태 관리 ⭐
`Pinia`는 전역 상태 뿐만 아니라, 개별 컴포넌트에서만 필요한 로컬 상태도 관리할 수 있습니다.  
컴포넌트 간 상태 공유를 더욱 쉽게 할 수 있습니다.

### 4. 성능 최적화 ⭐
`Pinia`는 Vue.js 3의 `reactive` 함수를 사용하여 상태를 관리합니다.  
이를 통해 상태 변화를 실시간으로 추적하고, 반응형으로 처리합니다. 따라서 성능 최적화에 용이합니다.

### 5. 유연한 getter, mutation, action API ⭐
`Pinia`는 Vuex와 유사한 개념인 getter, mutation, action과 같은 API를 제공합니다.  
이를 통해 각 상태에 대한 접근과 변경을 보다 유연하게 할 수 있습니다.

`Pinia`는 이러한 특징으로 Vue.js 애플리케이션에서 상태 관리를 보다 편리하게 할 수 있도록 도와줍니다.

저희 서비스인 인재풀의 경우 TypeScript를 활용하고 있지 않아 2번의 장점은 실제로 실감하지 못했으나  그 외의 `Pinia`에서 설명하는 주요한 장점들은 실제로 체감하며 작업을 진행했습니다. 개인적으로 코드 구조가 Vuex보다 직관적이어서 작업시에 1, 5번의 장점이 가장 크게 와닿았습니다!

## 2-2. Vuex와 Pinia 비교
기존 Vue 사용자들은 `Vuex`에 익숙해져 있어 `Pinia`를 선택하는게 망설여질 수 있을거라고 생각합니다.

저희도 상태관리 라이브러리 선택에 있어 여러 고민이 있었으나 이번 프로젝트의 구조상  `Pinia`의 직관적인 API 디자인과 성능상의 이점이 특히 유용할것이라 판단해 적용하게 되었습니다.

무조건 둘 중 하나가 좋다고 말하기 어렵기 떄문에 `Vuex`와 `Pinia`의 차이점에 대해 확인하고  적용하려는 프로젝트의 특성에 적합한 라이브러리를 선택하시길 추천합니다.

`Vuex`와 `Pinia`의 차이점은 아래와 같습니다.

### 1. API 디자인

- `Vuex`: 객체 기반으로 동작하며, 상태, 게터, 뮤테이션, 액션 등이 각각 별도의 객체로 정의됩니다.
- `Pinia`: 함수 기반으로 동작하며, 상태, 게터, 뮤테이션, 액션 등이 모두 단일 객체 내부에 정의됩니다.

### 2. 타입스크립트 지원
- `Vuex`: 기본적으로 타입스크립트를 지원하지 않습니다. 타입스크립트를 사용하려면 별도의 라이브러리(vuex-module-decorators)를 사용해야 합니다.
- `Pinia`: 기본적으로 타입스크립트를 지원합니다.

### 3. 모듈 구성
- `Vuex`: 모듈 단위로 구성됩니다. 모듈마다 상태, 게터, 뮤테이션, 액션 등이 각각 별도의 객체로 정의됩니다.
- `Pinia`: 전역 상태와 로컬 상태를 모두 관리할 수 있습니다. 컴포넌트 단위로 상태를 구성할 수 있으며, 필요한 경우 모듈로 구성할 수도 있습니다.

### 4. 성능
- `Pinia`는 상태를 Proxy 객체로 래핑하여 Vue.js 3의 반응성 시스템을 이용해 상태 변경을 감지하므로,   반응성 시스템의 성능을 최대한 활용할 수 있습니다.
- `Vuex`는 일반 객체를 사용하므로 반응성 시스템의 성능을 최적화할 수 없습니다.

### 5. 파일 크기
- `Pinia`는 `Vuex`보다 더 가벼우며, 필요한 기능만 사용하면 더 적은 파일 크기로 상태 관리를 할 수 있습니다.

결론적으로 두 라이브러리 모두 다른 장점을 갖고 있습니다.   
`Vuex`는 Vue.js의 공식 상태 관리 라이브러리로서 안정적이고 많은 사용자들이 사용하고 있다는 장점이 있고 `Pinia`는 `Vuex`보다 가볍고 직관적이며 타입스크립트를 지원하고 컴포넌트 단위로 상태를 구성할 수 있어 유연하다는 장점을 갖고 있습니다.  

각각의 이점이 있기 때문에 프로젝트의 특성에 따라 적합한 라이브러리를 사용하는게 좋다고 생각합니다.

***  

지금까지 `Composition API`, `Pinia`에 대해 생소한 독자를 위해 이론적 설명을 먼저 진행했습니다.  다음 포스팅에서는 실제 프로젝트 적용 예시를 살펴보고 인재풀 서비스에서 어떻게 활용하며 장점을 극대화했는지 이야기해보겠습니다 :)

긴 글 읽어주셔서 감사합니다!
