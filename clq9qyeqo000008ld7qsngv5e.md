---
title: "React Suspense 알아보기"
datePublished: Mon May 16 2022 03:34:25 GMT+0000 (Coordinated Universal Time)
cuid: clq9qyeqo000008ld7qsngv5e
slug: react-suspense-overview
canonical: https://velog.io/@xiniha/React-Suspense-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0
tags: javascript, reactjs

---

React 18이 정식으로 출시되었는데요, React 18은 Suspense 사용에 있어서 큰 변화를 가져왔고, 따라서 이번 글에서는 React 18에서 변한 사항들을 포함하여 Suspense의 전체적인 내용들에 대해 알아보려고 합니다.

## 전통적 방식의 비동기 데이터 기반 렌더링

React에서 컴포넌트를 작성하다 보면, 비동기 데이터에 의존하여 UI를 그리는 컴포넌트를 심심치 않게 작성하게 됩니다. 일반적인 React의 컴포넌트 렌더링은 동기적으로 이뤄져야 하기 때문에, 비동기 컴포넌트 렌더링은 별도의 처리가 필요합니다. 이런 경우에 가장 전통적으로 사용되는 구현법은 다음과 같이 비동기 데이터의 로딩 상태를 컴포넌트 내에서 관리하며 로딩 상태에 따라 적절한 UI를 그려 주는 방식입니다.

```javascript
// Component.jsx
const Component = () => {
  const [data, setData] = useState(null)
  
  useEffect(() => {
    fetch(API_URL)
      .then(res => res.json())
      .then(setData)
  }, [])
  
  if (!data) return <>Loading...</>
  else return <>Data: {JSON.stringify(data.results)}</>
}
```

```javascript
// App.jsx
const App = () => {
  return <Component />
}
```

위 코드에서 비동기 상태를 컨트롤하는 부분을 별도 Hook으로 분리하면 다음과 같이 사용할 수 있습니다.

```javascript
// api.js
export const useFetch = (url) => {
  const [data, setData] = useState(null)
  
  useEffect(() => {
    fetch(url)
      .then(res => res.json())
      .then(setData)
  }, [url])
  
  return data
}
```

[전체 코드 - StackBlitz](https://stackblitz.com/edit/vitejs-vite-ra2uhk)

Hook에서 로딩이나 에러 상태를 별도로 관리한다던가, 캐싱을 적용한다던가, 아니면 아예 직접 Hook을 만드는 대신 React Query 같은 라이브러리를 사용하시는 분들도 계시겠지만, 결국 개념적으로 위 코드와 유사한 형태를 띄게 될 것입니다.

위 코드는 대체로 잘 작동하지만, 몇 가지 문제가 있습니다.

* 개별 컴포넌트가 각각 로딩 상태를 표현하는 방식을 정의합니다. (이는 상황에 따라 UI 일관성을 떨어뜨릴 수 있습니다)
    
* 개별 컴포넌트가 모두 각각의 로딩 상태를 가지게 됩니다. 따라서 로딩 상태를 나타내는 스피너 등의 UI가 과도하게 많아질 수 있습니다.
    
* 컴포넌트 렌더링이 완료된 후 컴포넌트가 DOM에 Mount되어 `useEffect()`가 실행되는 시점에서야 데이터를 가져오기 시작합니다.
    
* React가 비동기 렌더링에 대해 이해할 수 없으며, 따라서 SSR 등의 상황에 대응하려면 별도의 작업이 필요합니다.
    

이 문제들은 앱이 '동작하도록' 만드는 데에는 그다지 문제가 되지 않지만, 엄연히 앱의 퀄리티에 영향을 주는 문제들이고, 높은 퀄리티의 앱을 개발하기 위해서는 반드시 해결해야 하는 문제들입니다. 비동기 데이터를 최초 렌더링 시에 모두 가져온다던가 하는 방식으로 문제를 회피할 수도 있지만, 데이터를 가져오는 코드와 사용하는 코드가 분리되면서 유지보수가 힘들어지는 문제가 새로 발생하는 등 회피책으로서의 한계가 뚜렷합니다. 이런 상황에서 React 팀이 이런 문제를 근본적으로 해결하기 위해 제안한 것이 바로 Suspense입니다.

## Suspense: 비동기 렌더링을 위한 새로운 표현 방식

Suspense를 사용하면, 컴포넌트의 렌더링이 비동기로 이루어질 수 있습니다. 즉 위에서 언급했던 문제들을 근본적으로 해결할 수 있다는 뜻이죠. 한번 코드를 살펴볼까요?

```javascript
// Component.jsx
const Component = () => {
  const data = useFetchAsync(API_URL) // useFetchAsync()의 구현은 아래에서 다루겠습니다.
  
  return <>Data: {JSON.stringify(data.results)}</>
}
```

```javascript
// App.jsx
const App = () => {
  return (
    <Suspense fallback={'Loading...'}>
      <Component />
    </Suspense>
  )
}
```

코드에 생긴 한눈에 확인할 수 있는 변화는 다음과 같습니다.

* `Component` 코드에서 비동기 상태 처리 관련 코드가 사라졌습니다.
    
* `Component`를 사용하는 `App` 측에서 Suspense를 활용하여 로딩 상태를 처리하고 있습니다.
    

즉, 위에서 언급했던 네 개의 문제 중 첫 번째 문제 (로딩 상태 표현 방식을 개별 컴포넌트가 정의한다) 가 해결되었습니다! 두 번째 문제인 "개별 컴포넌트가 모두 각각의 로딩 상태를 가지게 된다" 역시 Suspense를 활용하여 해결이 가능한데요, 한번 살펴볼까요?

```javascript
const App = () => {
  return (
    <Suspense fallback={'Loading...'}>
      <Component />
      <Component />
    </Suspense>
  )
}
```

원래 방식대로 컴포넌트 내부적으로 로딩 상태를 관리하고 로딩 컴포넌트를 표시했다면, 위 상황에서 두 개의 `<Component />`가 각각의 로딩 컴포넌트를 표시했을 것입니다. 그러나 Suspense를 사용하여 **두 로딩 상태를 하나로 묶었기 때문**에, 하나의 로딩 컴포넌트로 두 로딩 상태를 모두 표시할 수 있게 되었습니다. 마치 `Promise.all()` 같기도 하죠?

위에서 언급했던 네 가지 문제들 중 나머지 두 가지 문제 역시 Suspense에 의해 해결되는데요, 이에 대해 알아보기 위해서 Suspense가 내부적으로 어떻게 동작하는지를 좀 더 자세히 알아보도록 하겠습니다.

## Suspense의 내부 동작

위에서 Suspense에 대해서 소개할 때 "Suspense를 사용하면 컴포넌트 렌더링이 비동기로 이루어질 수 있다"고 언급했었는데요, 이게 정확히 무슨 뜻일까요? `useFetchAsync()`의 구현을 살펴보면서 이야기를 이어나가 봅시다.

```javascript
// api.js

// 실제 코드에서는 Context를 활용하여 관리하는 것이 권장됩니다.
const cache = new Map()

export const useFetchAsync = (url) => {
  const state = cache.get(url)
  switch (state?.status) {
    case undefined: {
      const promise = new Promise((resolve, reject) => {
        fetch(url)
          .then(res => res.json())
          .then(data => {
            cache.set(url, {
              status: 'ready',
              data
            })
            resolve(url)
          })
          .catch(error => {
            cache.set(url, {
              status: 'errored',
              error
            })
            reject(error)
          })
      })
      cache.set(url, {
        status: 'pending',
        promise
      })
      throw promise
    }
    case 'pending': throw state.promise
    case 'ready': return state.data
    case 'errored': throw state.error
  }
}
```

[전체 코드 - StackBlitz](https://stackblitz.com/edit/vitejs-vite-aor9oq)

코드가 좀 복잡한데, 한번 찬찬히 살펴보겠습니다.

* `cache`는 각 URL별 로딩 상태를 추적하기 위해 사용하는 Map으로, 키로 URL을 사용하고, 값으로 해당 URL의 데이터 로딩 상태를 나타내는 객체를 사용할 것입니다. 이 객체는 데이터 로딩을 대기 중인 상태, 데이터가 로딩된 상태, 로딩 중 에러가 발생한 상태를 나타냅니다. Promise와 1대 1로 대응된다고 보셔도 되겠네요.
    
* `useFetchAsync()` Hook 코드를 살펴봅시다. 이 Hook이 실행되면 위의 `cache`에서 파라미터로 받은 URL에 대한 로딩 상태를 가져오고, 로딩 상태가 어떤지에 따라서 적절한 동작을 합니다.
    
    * `cache` 내에 파라미터로 받은 URL에 대한 로딩 상태가 없다면, 데이터를 로딩하는 Promise를 생성하고, 그 Promise를 담은 `'pending'` 상태의 객체를 만들어서 `cache`에 넣고, 그 Promise를 `throw`합니다(!!!)
        
        * 생성한 Promise는 데이터 로딩이 완료되면 로딩된 데이터를 넣어 `‘ready'` 상태의 객체를 만들고, `cache`에 담는 일을 합니다.
            
        * 만약 에러가 발생한다면 발생한 에러를 넣어 `‘errored‘` 상태의 객체를 만들고, `cache`에 담습니다.
            
    * `cache` 내 객체의 `status`가 `‘pending‘`이라면, 객체 내의 Promise를 `throw`합니다.
        
    * `cache` 내 객체의 `status`가 `‘ready‘`라면, 객체 내의 데이터를 반환합니다.
        
    * `cache` 내 객체의 `status`가 `‘errored‘`라면, 객체 내의 에러를 `throw`합니다.
        

아마 위 코드에서 가장 낯설게 다가오는 부분은 바로 Promise를 `throw`하는 부분일 텐데요, 아마 많은 분들이 `throw`를 에러를 던질 때에나 사용하셨겠지만, 사실 `throw`와 `try-catch`는 모든 값을 대상으로 사용할 수 있습니다. React는 이 특성을 이용하여, 컴포넌트 실행 중 Promise가 `throw`하는 동작을 **컴포넌트 렌더링의 중단을 표현하기 위해** 사용합니다!

> 비록 Suspense 최초 출시 후 Promise를 `throw`하는 표현법이 꽤나 오랫동안 사용되었지만, 공식적으로 이 표현법은 아직 확정된 디자인이 아니며, 추후 변경될 수 있고, 아마 변경될 것입니다.

컴포넌트의 렌더링이 중단되면 무슨 일이 일어날까요? 컴포넌트가 속한 가장 가까운 부모의 Suspense가 "중단 상태"에 빠지며, `children` 대신 프로퍼티로 지정한 `fallback` 항목을 렌더링하게 됩니다. 이후 아까 렌더링을 중단시킬 때 `throw`한 Promise가 `resolve`되면, 해당 컴포넌트의 렌더링을 다시 시도하게 됩니다. 아까 던진 Promise의 동작을 생각해 보면, Promise가 `resolve`될 때 `cache`의 내용을 결과값으로 입데이트했었고, `useFetchAsync()` Hook의 구현 상 캐시가 결과값과 함께 업데이트되면 해당 값을 반환하는 방식으로 작동하고, 그렇게 나머지 컴포넌트 렌더링도 완료됩니다.

동작 방식을 살펴보면 아시겠지만, Suspense를 사용하면 기존에 데이터를 가져오던 방식의 두 번째 문제인 "컴포넌트가 DOM에 마운트된 후에야 데이터를 가져오기 시작한다"를 컴포넌트 렌더링 과정 중에 데이터를 가져오기 시작함으로써 해결할 수 있게 됩니다. 물론 이를 좀 더 개선시키기 위해서는 컴포넌트가 필요로 하는 데이터를 미리 분석하여 가져온다던가 하는 최적화를 적용해야 하지만요. (이를 실현해 주는 프레임워크로는 Relay가 있습니다)

## `startTransition()`: 시급하지 않은 상태 업데이트 실행하기

기존 방식의 네 번째 문제점이던 SSR 지원의 어려움을 Suspense가 어떻게 해결해주는지를 논하기 전에, React 18에 새로 추가된 `startTransition()` API와, 이 API가 Suspense와 함께 사용되었을 때 사용자 경험 측면에서 어떤 개선 사항을 이끌어낼 수 있는지 살펴 보도록 하겠습니다.

React 18에서 추가된 가장 대표적인 API로 `startTransition()`을 들 수 있을 것 같은데요, `startTransition()`이 하는 일은 바로 "시급하지 않은 상태 업데이트를 여유 있는 상황에 실행하도록 에약하기"입니다. "시급하지 않은 상태 업데이트"라니, 한번에 와닿지는 않죠? 한번 예제 코드와 함께 알아보도록 하겠습니다.

```javascript
const fib = (num) => {
  if (num <= 2) return 1
  else return fib(num - 2) + fib(num - 1)
}

const App = () => {
  const [num, setNum] = useState(1)
  const [fibNum, setFibNum] = useState(1)
  const [pending, startTransition] = useTransition()
  const result = useMemo(() => fib(fibNum), [fibNum])

  return (
    <>
      <p>
        <button
          onClick={() => {
            setNum((i) => i - 1)
            startTransition(() => setFibNum((i) => i - 1))
          }}
        >
          -
        </button>
        {num}
        <button
          onClick={() => {
            setNum((i) => i + 1)
            startTransition(() => setFibNum((i) => i + 1))
          }}
        >
          +
        </button>
        {pending && '업데이트 중...'}
      </p>
      <p>
        피보나치 수열의 {fibNum}번째 수는 {result}입니다.
      </p>
    </>
  )
}
```

[전체 코드 - StackBlitz](https://stackblitz.com/edit/vitejs-vite-gqwumg)

먼저 `useTransition()`과 `pending`, `startTransition()` 부분을 제외하고 코드를 설명해보자면, 위 코드는 상단에 수 `num`과 `fibNum`을 증감시킬 수 있는 카운터를 표시하고, 그 바로 아래에 피보나치 수열의 `fibNum`번째 수를 표시하는 컴포넌트의 코드입니다. 피보나치 수열을 계산하는 것은 수열의 뒤쪽으로 갈수록 매우 많은 계산을 필요로 하는 작업이 되기 때문에, `useMemo()`를 사용하여 계산랑을 최소화해 준 것을 확인할 수 있습니다.

`useTransition()`을 호출하는 부분을 살펴보자면, 해당 Hook을 호출 시 `pending`이라는 변수와 `startTransition()`이라는 함수를 반환하는 것을 볼 수 있습니다. `pending` 값은 `startTransition()`으로 실행한 상태 업데이트가 진행 중이면 `true`, 아니면 `false` 값을 나타냅니다. 하단 JSX에서 `{pending && '업데이트 중...'}`을 해 둔 부분을 보면, 상태 업데이트가 진행 중일 때 `pending`이 `true`가 되고, 메시지를 표시하게 될 것이라고 예상해볼 수 있습니다.

그렇다면 `startTransition()`은 무슨 상태 업데이트를 일으키기 위해 사용될까요? `startTransition()`을 사용하는 부분을 보시면 모두 `setFibNum()`을 사용하여 `fibNum`을 업데이트하는 동작을 실행하고 있습니다. `fibNum`을 업데이트한다는 건, 즉 피보나치 수열을 다시 계산해서 표시되는 결과가 업데이트된다는 뜻이죠. 이걸 `startTransition()`에 넣음으로써 어떤 차이가 생기게 될까요? `startTransition()` 사용이 만들어 내는 차이를 보여 드리기 위해, `startTransition()`을 사용하지 않은 버전과 동작을 비교해 보겠습니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1702832680291/6390ed01-9d89-4e80-9fc2-af01ec08a9a8.webp align="left")

* `startTransition()`을 사용하지 않은 버전은 버튼을 누르면 아무 동작 없이 앱이 잠시 동안 멈췄다가, 계산이 끝나면 카운터의 숫자와 아래 결과 숫자가 함께 업데이트됩니다.
    
* `startTransition()`을 사용한 버전은 버튼을 누르면 카운터의 숫자가 즉시 업데이트되고, 카운터 옆에 `업데이트 중...` 텍스트가 표시됩니다. 이후 계산이 끝나면 텍스트가 사라지고 아래의 결과 숫자가 업데이트됩니다.
    

위에서 살펴본 것을 정리해보자면, `startTransition()`**을 사용하면 시급하지 않은, 많은 계산을 요구하는 상태 업데이트를 뒤로 미룰 수 있다** 정도로 볼 수 있겠네요! 앱의 사용자에게 "카운터를 조작한 동작이 반영되었다"는 메시지를 전달할 수 있다는 점에서, `startTransition()`을 사용하지 않은 버전보다 상대적으로 부드러운 사용자 경험을 제공한다고 평가할 수 있겠습니다.

## `startTransition()`과 Suspense의 조합

이전 예제에서는 `startTransition()`이 많은 계산을 요구하는 상태 업데이트를 발생시킬 때 어떻게 사용자 경험을 향상시킬 수 있는지를 살펴보았는데요, 이번에는 `startTransition()`과 Suspense가 조합되었을 때 어떠한 방식으로 사용자 경험을 향상시킬 수 있는지 알아보도록 하겠습니다.

```javascript
// DelayedPage.jsx
const cache = new Map()

const DelayedPage = ({ id, delay, children }) => {
  let state = cache.get(id)
  if (!state) {
    state = new Promise((resolve) =>
      setTimeout(() => {
        cache.set(id, true);
        resolve(true)
      }, delay)
    )
    cache.set(id, state)
  }
  if (state !== true) throw state

  useEffect(
    () => () => {
      cache.delete(id)
    },
    [id]
  )

  return <>{children}</>
}
```

```javascript
// App.jsx
const App = () => {
  const [page, setPage] = useState(0)

  const pages = [
    <DelayedPage id="0" delay={500}>
      500ms 지연된 페이지
    </DelayedPage>,
    <DelayedPage id="1" delay={1000}>
      1000ms 지연된 페이지
    </DelayedPage>,
    <DelayedPage id="2" delay={3000}>
      3000ms 지연된 페이지
    </DelayedPage>,
  ]

  return (
    <>
      <nav>
        <button onClick={() => startTransition(() => setPage(0))}>
          첫 번째 페이지
        </button>
        <button onClick={() => startTransition(() => setPage(1))}>
          두 번째 페이지
        </button>
        <button onClick={() => startTransition(() => setPage(2))}>
          세 번째 페이지
        </button>
      </nav>
      <Suspense fallback={'로딩 중...'}>
        {pages[page]}
      </Suspense>
    </>
  )
}        
```

[전체 코드 - StackBlitz](https://stackblitz.com/edit/vitejs-vite-rp76rb)

위 코드는 꽤나 단순하게 구현된 탭바를 포함한 페이지 레이아웃입니다. 각 페이지는 렌더링이 일정 시간 동안 지연된 후 일어나며(비동기 렌더링), 탭바의 버튼을 누름에 따라서 하단에 표시되는 페이지가 바뀌게 됩니다. 이 코드에서 `startTransition()`은 무슨 역할을 하고 있을까요? `startTransition()`이 어떤 부분에 적용되었는지 살펴보면, 바로 현재 표시되는 페이지를 바꾸는 상태 업데이트에 해당되는 코드에 적용되어 있다는 것을 확인할 수 있습니다. 매 페이지 전환은 `<DelayedPage />`가 구현된 방식 특성상 자연스레 렌더링을 중단시키고, Suspense가 `fallback`을 보여주게 만들 것입니다.

...정말 그럴까요?

`startTransition()` **내에서 업데이트한 상태로 인해 시작된 렌더링이 중단된 경우, Suspense는** `fallback`**을 보여주는 대신, 이전에 보여주고 있던 UI를 표시합니다.** 이를 활용하면 여러 비동기 페이지 간에 이동할 때, 로딩 표시를 보여주는 대신 페이지가 좀 더 부드럽게 전환될 수 있도록 구성하는 것이 가능해집니다. 그리고 `startTransition()`의 "시급하지 않게" 상태를 업데이트한다는 특성을 활용하면, 시급한 상태 업데이트와 시급하지 않은 상태 업데이트를 구별하여, 이전 페이지의 콘텐츠와 로딩 상태를 동시에 보여주는 방식으로 구현하는 것이 가능해지죠.

```javascript
// 상태 정의
const [loading, setLoading] = useState(false)

// 업데이트 트리거
<button
  onClick={() => {
    setLoading(true)
    startTransition(() => {
      setPage(0)
      setLoading(false)
    })
  }}>
  첫 번째 페이지
</button>
```

위 코드에서 버튼의 클릭 핸들러를 살펴보면, 먼저 시급한(`startTransition()`에 감싸지지 않은) 상태 업데이트로 `loading`을 `true`로 만들고, 이후 `startTransition()`을 통해 페이지 전환을 트리거한 다음, 이것이 완료되었을 때 `setLoading(false)`가 실행되게 하여 `loading`을 다시 `false`로 만들도록 코드가 구성되었습니다. 이제 `loading` 값을 활용하여 로딩 바를 보여준다던가 하는 방식으로 화면을 구성하면, 사용자에게 좀 더 부드러운 앱 사용 경험을 제공해줄 수 있겠죠?

이제 위에서 이야기했던 기존 비동기 데이터 사용 방법의 마지막 문제점이었던 SSR 환경 지원 문제를 Suspense가 어떻게 해결해주는지 살펴보도록 하겠습니다.

## Suspense를 활용한 Streaming SSR

SSR 환경에서 비동기 데이터에 의존하는 컴포넌트는 항상 골칫거리입니다. 아무 생각 없이 `useEffect()`로 비동기 데이터를 가져오려 했다면 서버 환경에서는 해당 코드가 실행되지 않아 그냥 빈 화면만 보내질 것이고(경우에 따라서는 SSR된 HTML이 아무 의미가 없게 되는 경우도 종종 발생할 수 있겠죠), NextJS나 Remix가 전통적으로 해오던 것처럼 페이지 렌더링 이전에 데이터를 미리 모두 가져와 놓는 방식으로 구성했다가는 데이터 가져오기가 모두 완료되기 전까지 사용자는 빈 화면만 보고 있어야 하는 문제가 발생하겠죠. 이 문제를 해결하려면 어떻게 해야 할까요? 아니, 이 문제를 해결할 수는 있는 걸까요?

Suspense가 React 내에서 비동기 렌더링을 표현할 수 있는 새로운 언어로 추가되면서, React는 비동기 렌더링에 대해 이해할 수 있게 되었습니다. 이는 React가 **Suspense Streaming SSR**이라는 새로운 테크닉을 구현할 수 있도록 해 주었는데요, 간단하게 설명하자면 "Suspense 외곽의 콘텐츠를 먼저 보내주고, Suspense 내의 콘텐츠는 렌더링이 완료되는 대로 클라이언트로 전달해 줘서 렌더링시키자"는 것입니다. 이러면 사용자는 페이지가 클라이언트에서 렌더링된 시점에서야 뒤늦게 데이터를 가져오기 시작하지도 않을 것이고, Suspense 렌더링이 완료될 때까지 하염없이 빈 화면만 보면서 기다릴 필요도 없어지겠죠. 이해를 돕기 위해 Streaming SSR이 실제로 동작하는 화면을 보여드리도록 하겠습니다.

대략 다음과 같은 코드(상세 사항은 생략하였습니다)를 Streaming SSR로 렌더링했을 때,

```javascript
const App = () => {
  return (
    <article>
      <aside>
        <h1>This is a sidebar</h1>
        <p>
          Try clicking the counter even when the main content is still loading!
        </p>
        <Counter />
      </aside>
      <main>
        <CacheProvider>
          <Suspense fallback={"Loading..."}>
            <DataConsumer id="foobar">
              {(data) => (
                <>
                  <p>
                    Data: {data}
                    <br />
                    Try clicking the counter even when the other Suspense is
                    still loading!
                    <br />
                    <Counter />
                  </p>
                  <Suspense fallback={"Nested Loading..."}>
                    <DataConsumer id="fizzbuzz">
                      {(data) => (
                        <>
                          <p>
                            Nested Data: {data}
                            <Counter />
                          </p>
                        </>
                      )}
                    </DataConsumer>
                  </Suspense>
                </>
              )}
            </DataConsumer>
          </Suspense>
        </CacheProvider>
      </main>
    </article>
  );
};
```

다음과 같이 작동하게 됩니다.

![](https://velog.velcdn.com/images/xiniha/post/b1ae067a-6a5b-4462-95b9-65aff4cc4e0e/image.webp align="left")

가장 눈에 띄는 부분은, Suspense 내에서 비동기로 지연되어 렌더링되는 여러 `<DataConsumer />`들이 **즉시 동기적으로 렌더링되는 좌측의 UI가 클라이언트로 전달되는 것을 막지 않고, 비동기 렌더링이 끝나는 대로 클라이언트에 추가적으로 렌더링 결과물을 전달하는 방식으로 렌더링되었다는 것입니다.** 또한, 서버에서 렌더링이 왼료된 컴포넌트가 **클라이언트에 도착하는 즉시 하이드레이션이 완료되어 컴포넌트를 사용할 수 있다는 점**도 확인할 수 있습니다. 기존에 비동기 데이터에 가로막혀, 동기적으로 렌더링될 수 있는 부분들도 비동기 데이터 다운로드가 완료될 때까지 클라이언트에 그려질 수 없었던 문제를 완전히 해결한 것이죠. 스트리밍 SSR 관련해서는 이야기할 게 산더미라 마음 같아서는 신나게 떠들고 싶지만 글의 범위와 분량 조절을 위해(...) 제가 [여러 스트리밍 SSR 구현 예제를 살펴볼 수 있도록 구성한 프로젝트 링크](https://github.com/XiNiHa/streaming-ssr-showcase)만 남기고 넘어가도록 하겠습니다...

## 정리

위에서 언급했던 기존 비동기 렌더링 표현 방식의 문제점 4가지와, 이를 Suspense가 어떻게 해결했는지를 다시 살펴 보겠습니다.

* 개별 컴포넌트가 각각 로딩 상태를 표현하는 방식을 정의합니다. (이는 상황에 따라 UI 일관성을 떨어뜨릴 수 있습니다)
    
    * Suspense를 활용하면 해당 컴포넌트를 사용하는 측에서 로딩 상태를 표현하는 방식을 정의하기에, UI 일관성을 유지하는 데 큰 도움을 줍니다.
        
* 개별 컴포넌트가 모두 각각의 로딩 상태를 가지게 됩니다. 따라서 로딩 상태를 나타내는 스피너 등의 UI가 과도하게 많아질 수 있습니다.
    
    * Suspense를 활용하면 여러 컴포넌트의 로딩 상태를 묶어서 표현할 수 있습니다. 따라서 로딩 상태를 나타내는 UI의 수를 적절하게 조절할 수 있습니다.
        
* 컴포넌트 렌더링이 완료된 후 컴포넌트가 DOM에 Mount되어 `useEffect()`가 실행되는 시점에서야 데이터를 가져오기 시작합니다.
    
    * Suspense를 활용하면 데이터를 가져오는 시점을 DOM Mount 이전으로 당길 수 있습니다.
        
* React가 비동기 렌더링에 대해 이해할 수 없으며, 따라서 SSR 등의 상황에 대응하려면 별도의 작업이 필요합니다.
    
    * Suspense를 활용하면 React가 비동기 렌더링을 이해할 수 있고, 이에 따라 SSR 역시 손쉽게 가능합니다.
        

Suspense는 기존 React의 비동기 렌더링 방식의 여러 문제점들을 해결하는, React에게 있어서 매우 중요한 변화입니다. 여기에 Suspense를 기반으로 하는 [`<Cache />` API](https://github.com/reactwg/react-18/discussions/25)나 Server Components 등의 기능까지 더해지게 되면 더욱 엄청난 변화를 가져오게 될 것입니다. 따라서 이 거대한 변화의 기반인 Suspense를 이해하는 것은 앞으로 React를 사용하는 데에 있어서 큰 도움이 될 것이고, 이 글이 좀 더 많은 분들이 Suspense를 이해하는 데에 도움을 줄 수 있길 바랍니다.