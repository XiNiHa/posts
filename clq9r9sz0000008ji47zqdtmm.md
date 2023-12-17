---
title: "쉽게 배우는 Relay - 1. Relay 소개"
datePublished: Wed Sep 07 2022 13:43:59 GMT+0000 (Coordinated Universal Time)
cuid: clq9r9sz0000008ji47zqdtmm
slug: relay-tutorial-1-introdution
canonical: https://velog.io/@xiniha/%EC%89%BD%EA%B2%8C-%EB%B0%B0%EC%9A%B0%EB%8A%94-Relay-1-Relay-%EC%86%8C%EA%B0%9C
tags: relay, javascript, reactjs, graphql

---

안녕하세요! 이번에는 짧게 끝내는 글이 아닌 시리즈로 연재하는 튜토리얼을 만들어 보려고 하는데요, Relay의 공식 문서가 너무 어렵고 복잡하다는 수많은 울부짖음을 듣고(...) 한번 한국어로 쉽게 따라갈 수 있는 튜토리얼을 써 봐야겠다고 결심하게 되었습니다. Relay의 보급은 대의를 위해 매우 중요한 이슈이니까요. (?) 그럼 한 번 시작해 볼까요?

## Relay란?

Relay는 Meta(구 Facebook)에서 개발 중인 GraphQL 클라이언트로, Meta에서 개발하는 각종 프론트엔드에서 가장 활발히 사용되고, 가장 권장되는 GraphQL 클라이언트입니다. 실제로 현재 Facebook 웹 버전은 Relay를 매우 적극적으로 사용하고 있으며, 이는 높은 복잡도를 가진 아주 큰 규모의 웹앱에서도 사용하기 매우 적합한 클라이언트라는 점을 엿볼 수 있게 해 줍니다.

Meta에서 개발한 물건답게 React와도 매우 좋은 궁합을 자랑하며, React의 각종 실험 기능들을 선제적으로 테스트해 보는 일종의 테스트 베드이기도 합니다. 가장 대표적인 예시로는 Suspense 기반 데이터 페칭이 있죠! 이외에도 Server Component나 새 `<Cache />` API 등 다양한 범위에 걸쳐 테스트가 이뤄지고 있습니다.

## Relay를 쓰는 이유

### 페이지별로 최적화된 단일 쿼리 생성

Relay는 클라이언트에서의 GraphQL 활용의 정수를 보여 주는 라이브러리로, GraphQL의 잠재력을 최대한으로 끌어낼 수 있도록 도와 주는 각종 기능들로 무장하고 있습니다. 그 중 가장 큰 비중을 차지하는, Relay의 정체성이라고도 할 수 있는 기능이 바로 **페이지별로 최적화된 단일 쿼리를 생성해주는 기능**입니다.

GraphQL을 사용하는 가장 단순한 방법은, 각 컴포넌트 내에서 사용하고자 하는 데이터를 직접 쿼리하는 것입니다.

```javascript
const Profile = ({ userId }) => {
  const [{ data }] = useQuery(graphql`
    query (＄userId: ID!) {
      user(id: ＄userId) {
        name
        profileImageURL
      }
    }
  `, { userId })
  
  if (!data.user) return null
  
  return (
    <section>
      <img src={data.user.profileImageURL} />
      <h2>{data.user.name}</h2>
    </section>
  )
}
```

GraphQL을 사용해 보셨다면 아마 익숙한 형태의 코드일 것입니다. 그런데 이 방식은 매우 큰 문제가 있는데요, 각 컴포넌트가 GraphQL 쿼리를 개별적으로 날리기 때문에, 결과적으로 엄청난 양의 GraphQL 요청이 발생하게 됩니다. 심지어 이 요청들은 동시에 발생하는 것도 아니라 마치 폭포(Waterfall)의 형태처럼 차례대로 발생하게 되며, 이는 최종적으로 모든 데이터를 가져오는 데까지 걸리는 시간을 매우 크게 증가시킵니다. 예시를 보겠습니다.

```javascript
const UserList = () => {
  const [{ data }] = useQuery(graphql`
    query {
      users {
        id
      }
    }
  `)
  
  return (
    <>
      {data.users.map(({ id }) => (
        <Profile userId={id} />
      ))}
    </>
  )
}
```

`users`가 사용자 3명의 데이터를 반환한 경우, 네트워크 요청의 흐름은 다음과 같습니다.

```plaintext
UserList.useQuery
----------------->
                 |   Profile.useQuery(＄userId: 1)
                 |-------------------------------->
                 |   Profile.useQuery(＄userId: 2)
                 |-------------------------------->
                 |   Profile.useQuery(＄userId: 3)
                 |-------------------------------->
```

보시다사피 쿼리가 단순히 여러 개 나갈뿐만 아니라, 첫 쿼리가 완료된 시점에서야 가져올 유저 ID를 알 수 있게 되기 때문에 `Profile.useQuery`의 호출 시점이 늦어지게 되고, 결과적으로 모든 데이터가 준비되는 시점이 매우 늦어지게 됩니다. 어떻게 하면 이 문제를 해결할 수 있을까요? 답은 `UserList.useQuery`에서 `<Profile />`에서 사용할 데이터까지 모두 한번에 가져오는 것입니다.

```javascript
const Profile = ({ user }) => {
  return (
    <section>
      <img src={user.profileImageURL} />
      <h2>{user.name}</h2>
    </section>
  )
}

const UserList = () => {
  const [{ data }] = useQuery(graphql`
    query {
      users {
        id
        name
        profileImageURL
      }
    }
  `)
  
  return (
    <>
      {data.users.map(user => (
        <Profile user={user} />
      ))}
    </>
  )
}
```

이렇게 코드를 작성하면 `UserList.useQuery`만으로 필요한 데이터를 모두 가져오고, 가져온 데이터를 Props로 전달해 주는 방식으로 네트워크 요청을 최적화할 수 있습나다. 하지만 이 방식에도 문제가 존재하는데요, 바로 GraphQL 쿼리를 통해 데이터를 가져오는 코드와 가져온 데이터를 사용하는 코드가 여러 위치로 분리되었다는 것입니다. 이는 코드의 유지보수성을 크게 떨어뜨리며, 더 이상 쓸모 없는 필드를 "혹시 어디서 쓰일지도 모른다"는 이유로 계속해서 가져오거나, 쓸모 없다고 생각한 필드를 섣불리 지웠다가 필요한 데이터가 사라져 런타임 에러가 발생하는 등, 다양한 문제의 원인이 될 수 있습니다.

이 두 문제를 한 번에 해결할 수 있도록 해 주는 라이브러리가 바로 Relay입니다. Relay는 각 컴포넌트가 필요로 하는 데이터를 GraphQL의 Fragment를 통해서 표현하고, 이 컴포넌트를 사용하는 상위 컴포넌트의 쿼리(혹은 또 다른 Fragment)에서 해당 컴포넌트를 위한 Fragment 데이터를 가져온 후 Props로 전달하는 방식으로 위 두 문제를 모두 해결합니다. 코드를 확인해볼까요?

```javascript
const Profile = ({ user }) => {
  const data = useFragment(graphql`
    fragment Profile_user on User {
      name
      profileImageURL
    }
  `, user)
  
  return (
    <section>
      <img src={data.user.profileImageURL} />
      <h2>{data.user.name}</h2>
    </section>
  )
}

const UserList = () => {
  const data = useLazyLoadQuery(graphql`
    query {
      users {
        ...Profile_user
      }
    }
  `)
  
  return (
    <>
      {data.users.map(user => (
        <Profile user={user} />
      ))}
    </>
  )
}
```

코드를 보면, 사용할 데이터를 정의하는 부분은 `Profile` 컴포넌트에, 이 데이터를 가져오는 부분은 UserList에 넣음으로써 위에서 언급한 두 문제를 모두 해결한 것을 확인할 수 있습니다! 이렇게 각 컴포넌트에서 필요로 하는 데이터를 Fragment로 표현하는 방식은 최초 쿼리 시의 성능 이득 외에도 효율적인 Refetch나 `@defer/stream`을 활용한 점진적 쿼리 & 렌더링 등 많은 상황에서의 이점을 가져다 주며, 개별 `useFragment`가 `Suspense`를 트리거할 수 있게 되기 때문에 원하는 UX에 따라 유동적으로 `Suspense`와 `startTransition`을 활용하여 다양한 동작 방식을 손쉽게 골라 구현할 수 있습니다.

### Node와 캐시 정규화를 통한 수많은 편의 기능

Relay는 올바른 작동을 위해 GraphQL API에 몇 가지 제약 사항을 요구하는데요, 이 중 대표적인 예시가 바로 Node 인터페이스입니다. Node 인터페이스는 단일 ID 필드로 구성된 인터페이스로, 캐시에 ID 단위로 정규화될 수 있는 타입들을 나타내기 위한 인터페이스입니다. 주로 사용자 정보와 같이 이미 ID를 가지고 구분 가능한 형태의 타입에 Node 인터페이스를 구현하게 됩니다. 이 ID는 스키마 전체에서 유일해야 하며, 쿼리 최상단의 `node(id: ID!): Node` 필드에 Node의 ID를 파라미터로 넣어 쿼리하면 해당 ID를 가진 Node를 반환할 수 있도록 설계되어야 합니다.

이렇게 얻어진 Node는 Relay의 내부 캐시에 맵/딕셔너리의 형태로 저장되게 되는데요, Node 타입의 데이터는 해당 Node의 ID를 Key로 해서 캐시에 저장되게 되고, 이외의 경우에는 Relay 내부의 알고리즘에 따라 쿼리 내에 위치한 경로에 따라 적절한 ID를 부여받아 캐시에 저장되게 됩니다.

Relay에서 Node 인터페이스를 정의하고, 이 데이터를 저장하는 캐시를 사용함으로써 얻는 이득은 무엇일까요? 대표적으로는 다음 두 개가 있습니다.

1. 같은 ID를 가진 항목은 같은 객체로 연결되게 된다.
    
2. 전체 쿼리뿐만 아니라 개별 Fragment 데이터의 갱신이 가능해진다.
    

첫 번째부터 설명해 보자면, Relay에서 GraphQL을 통해 쿼리한 데이터는 앱 전역에 걸쳐서 그 형태가 표현되게 됩니다. 예를 들어 블로그 글을 나타내는 `Article` 타입의 값은 글 목록 페이지에서는 목록의 항목 중 하나로 글 제목, 업로드 날짜, 내용 요약, 썸네일, 좋아요 수 등이 표현되게 될 것이고, 글 상세보기 페이지에서는 글의 전체 내용과 해당 글에 등록된 댓글 등의 다양한 추가적인 데이터를 표현하게 될 것입니다. 그런데 만약 유저가 블로그 글을 읽다가 좋아요를 누르면 어떻게 되어야 할까요? 먼저 현재 보여주고 있는 상세보기 페이지의 좋아요 버튼이 눌리며 좋아요 수가 1개 늘어나야 할 것이고, 또한 목록에서 보여지는 글의 좋아요 수 역시 1개 증가해야 할 것입니다. Relay에서는 이 두 데이터가 결과적으로 캐시 내의 같은 값을 가리키고 있기 때문에, 캐시 내의 값만 업데이트해주면 이 두 값이 모두 갱신되게 됩니다! 캐시 내의 값을 갱신하는 것은 GraphQL Mutation을 통해 값을 변경한 후, 해당 Node의 변경된 값을 Mutation의 반환값으로 가져오면 자연스럽게 이뤄지게 됩니다. 이러한 손쉬운 갱신 흐름은 모든 데이터가 중앙 캐시라는 같은 소스를 바라보고 있기 때문에 가능합니다.

두 번째는 조금 특이한데요, 전체 쿼리에서 깊은 곳에 위치한 Fragment의 데이터만 갱신하는 것이 가능해진다는 것입니다! 위에서 언급했다시피 Relay를 사용하면 개별 쿼리의 수가 줄어들고 각 쿼리의 크기가 커지게 되는데요, 이 경우 새 데이터를 로딩하기 위해서 전체 쿼리를 다시 가져왔다가는 불필요한 네트워크 트래픽이 발생할 것입니다. 하지만 Relay는 개별 Fragment 중 데이터를 갱신할 수 있도록 할 Fragment를 지정해줄 수 있고, 해당 Fragment는 전체 쿼리를 다시 가져오지 않더라도 데이터를 갱신할 수 있습니다.

이외에도 Optimistic Update라고 불리는, 네트워크 요청이 완료되기 전에 해당 요청이 성공할 것으로 가정하고 우선적으로 데이터가 갱신된 UI를 보여주는 기능이라던가, 커서 기반 페이지네이션을 전용 Hook을 활용해서 손쉽게 가져오는 등의 동작이 모두 Relay의 캐시 정규화 덕분에 가능합니다.

## 앞으로 배울 내용

이번 글에서는 Relay가 어떤 문제를 해결하기 위해 나온 라이브러리이고, 어떤 기능들과 강점들을 가지고 있는지 살펴 보았습니다. 다음 글에서는 Relay를 설치하고 사용 환경을 세팅하는 것을 알아볼 예정이고, 이후로는 간단한 쿼리와 프래그먼트 활용부터, 페이지네이션 쿼리, Preloaded 쿼리, Mutation과 Optimistic Update 등 다양한 Relay의 기능을 다뤄볼 것입니다. 이외에도 Relay를 SSR 환경에 적용하는 방법 등, 공식 문서에서도 접하기 힘든 내용까지 풀어볼 예정이니 많은 관심 부탁드립니다!