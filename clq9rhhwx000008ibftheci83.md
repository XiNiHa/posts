---
title: "Partial Hydration 둘러보기"
datePublished: Wed Jan 04 2023 19:17:33 GMT+0000 (Coordinated Universal Time)
cuid: clq9rhhwx000008ibftheci83
slug: partial-hydration-introduction
canonical: https://velog.io/@xiniha/Partial-Hydration-%EB%91%98%EB%9F%AC%EB%B3%B4%EA%B8%B0
tags: frontend-development, server-side-rendering

---

안녕하세요! 이번 글에서는 최근 1~2년간 프론트엔드 영역에서 상당히 핫했던 주제 중 하나인 Partial Hydration에 대해 소개하고, 이를 구현하는 다양한 기법들을 살펴보는 시간을 가져보려고 합니다.

SSR의 문제
-------

SSR의 도입은 웹 애플리케이션의 최초 로딩 퍼포먼스에 있어서 많은 개선을 가져왔습니다. (SSR이 단순히 SEO를 위한 것이라고만 알고 계셨다면, [제 과거 발표를 시청하신 후에](https://youtu.be/9xl9X2pfHeI) 글을 마저 읽으시는 것을 추천드립니다) 하지만, SSR은 일반적인 웹 페이지를 만드는 상황에서는 기존 전통적인 MPA 대비 여전히 비효율적인 모습을 보여 주었는데요, 결국 JS로 UI를 그리는 SPA 프레임워크를 활용하는 이상, 필요한 만큼의 동적 기능을 위해 최소한의 JS만 다운로드하면 충분했던 MPA 대비 큰 용량의 JS를 다운로드하게 되고, 또한 이를 실행하여 HTML로 그려졌던 UI를 프레임워크에서 읽어들이는 하이드레이션 과정이 필요하게 되어, 결과적으로 최초 로딩 퍼포먼스에서 MPA 대비 손해를 볼 수밖에 없다는 사실이 드러난 것입니다.

이 문제는 React (Next, Remix), Vue (Nuxt), Svelte (SvelteKit), Solid (SolidStart) 등, 모던 SPA 프레임워크들을 활용하여 SSR을 구현한다면 프레임워크에 상관 없이 동일하게 겪게 되는 문제였습니다. 따라서 SSR 성능을 최적화하려면 반드시 해당 문제를 해결해야 했고, 이에 대한 해답으로 Partial Hydration이 제시되었습니다.

Partial Hydration이란?
--------------------

Partial Hydration은 Partial(부분적)이라는 말뜻 그대로, 페이지 전체가 아닌 페이지의 일부분에 대해서만 하이드레이션 작업을 진행하는 것을 이야기합니다. Partial Hydration을 적용함으로써 얻을 수 있는 이점은 다음과 같습니다.

*   하이드레이션을 진행해야 할 요소의 양이 적어지기 때문에 상대적으로 빠른 시간 안에 하이드레이션을 끝마칠 수 있음
*   하이드레이션을 진행하지 않아도 되는 요소들에 대한 코드를 번들에 포함시키지 않을 수 있기 때문에, 앱 번들의 크기를 크게 감소시킬 수 있음

이 두 이점은 위에서 언급된 SSR 성능 이슈를 최소화시켜주며, 따라서 처음 제시되자마자 많은 주목을 받으며 실제 구현을 위한 다양한 기법들이 연구되기 시작했습니다.

다양한 Partial Hydration 기법들
-------------------------

### 아일랜드 아키텍처 (Islands Architecture)

아일랜드 아키텍처는 현재 Partial Hydration을 위해 가장 범용적으로 사용되고 있는 기법으로, 하이드레이션이 이루어지지 않은 채로 존재하는 페이지 위에 하이드레이션된 구성요소들이 배치되는 방식이 마치 바다 위에 섬들이 있는 모습 같다고 하여 이름 붙여진 아키텍처입니다. **대체로 정적인 페이지 위에 일부분만 동적으로 작동하는 페이지를 개발할 때**에 가장 사용하기 적합한 기법입니다. 해당 기법은 [Astro](https://astro.build)에 의해 최초로 소개되었으며, 이후 [Fresh](https://fresh.deno.dev)나 [iles](https://iles.pages.dev), [SolidStart](https://start.solidjs.com) 등에 도입되었습니다.

### 서버 컴포넌트 (Server Components)

서버 컴포넌트는 리액트 측에서 개발한 기법으로, 말 그대로 **서버에서만 실행되는 컴포넌트**를 만들 수 있게 함으로써 해당 컴포넌트가 하이드레이션될 필요성 자체를 없애는 기법입니다. 위에서 소개한 아일랜드 아키텍처가 **기본적으로 정적으로 구성된 페이지에 일부분만 동적인 요소를 추가하는** 방식으로 구성되어 있었다면, 서버 컴포넌트의 경우 **기본적으로 앱 전체가 동적으로 하이드레이션되나, 서버 컴포넌트들만이 하이드레이션 대상에서 제외되는** 방식으로 동작합니다. 현재는 리액트와 그 파생 프레임워크들 일부([NextJS App 디렉토리](https://beta.nextjs.org/docs/getting-started), [Hydrogen](https://hydrogen.shopify.dev))에서 지원하며, Vue 진영의 NuxtJS에서도 지원을 위한 여러 PR ([#5689](https://github.com/nuxt/framework/pull/5689), [#5688](https://github.com/nuxt/framework/pull/5688))들을 작업 중입니다.

### `<NoHydration />`

`<NoHydration />`은 해당 컴포넌트의 자식으로 전달된 컴포넌트들에 대해 하이드레이션을 수행하지 않는 방식으로 동작하는 컴포넌트로, 위의 서버 컴포넌트와 같이 기본적으로 앱 전체가 동적인 상황에서 **컴포넌트 트리의 일부만 정적으로 렌더링하고 싶을 때** 사용합니다. 일반적인 상황에서는 그다지 큰 효과를 발휘하지 못하나, Lazy하게 로딩되는 컴포넌트([React](https://beta.reactjs.org/reference/react/lazy), [Vue](https://vuejs.org/guide/components/async.html), [Solid](https://www.solidjs.com/docs/latest/api#lazy))와 함께 사용하면 해당 컴포넌트에 대한 번들의 다운로드 자체를 막아버리는 방식의 최적화를 적용할 수 있습니다. 해당 컴포넌트는 Solid에 자체 구현되어 있으며, React에서는 [다소 꼼수에 가까운 방법으로](https://github.com/facebook/react/issues/12617#issuecomment-590096314) 구현이 가능합니다.

#### `<Hydration />`

Solid에서는 이에 더해 **해당 컴포넌트의 자식으로 전달된 컴포넌트에 대해서만 하이드레이션을 수행하도록 하는** `<Hydration />` 컴포넌트를 제공하며, 따라서 이 둘을 적절히 조합하는 방식으로 다양한 형태의 Partial Hydration 기법들을 구현할 수 있습니다. (실제로 [SolidStart의 아일랜드 구현](https://github.com/solidjs/solid-start/blob/1d572a091b496ab85ecab9265f818fa9f494bc70/packages/start/islands/index.tsx#L24-L78)은 이들을 활용합니다\])

### 아일랜드 라우팅 (Islands Routing)

아일랜드 라우팅은 위에서 소개한 아일랜드 아키텍처에 서버 컴포넌트를 약간 섞어 놓은 기법이라고 볼 수 있습니다. 기본적으로는 아일랜드 아키텍처로 구성된 앱처럼 정적인 페이지에 아일랜드가 심어진 형태로 페이지를 렌더링하지만, 이후 페이지 내에서 경로를 이동할 때 전체 페이지를 이동하는 대신 **페이지에서 변경될 부분에 대해서만 HTML을 렌더링하도록 서버에 요청하여 해당 요청의 결과물을 변경될 부분에 삽입하는 방식**으로 페이지 이동 없이 한 페이지 내부에서 부드럽게 페이지 전환 효과를 구현합니다. 이는 아일랜드 아키텍처를 적용한 앱들이 전통적인 MPA에 가깝기 때문에 라우팅 시의 사용자 경험이 떨어진다는 비판을 듣던 것을 서버 컴포넌트의 구현 방식에서 아이디어를 채용하여 해결해낸 케이스라고 볼 수 있습니다. 현재 SolidStart에 실험적 기능으로 포함되어 있으며, 여전히 매우 불안정한 상태로 많은 리팩토링이 이뤄지고 있는 기능입니다.

정리
--

Partial Hydration은 프론트엔드에서 최근 몇 년 사이에 가장 큰 발전을 이뤄낸 분야 중 하나로써, SSR 앱들의 성능 최적화에 큰 도움을 준 기법입니다. 아마 NextJS App 디렉토리의 정식 출시와 Astro의 보급, SolidStart의 안정화 및 보급 등의 절차를 거쳐 차츰 대중화될 것으로 보이는데, 개인적으로 현재 해당 분야에서 가장 많은 실험을 해 나가고 있는 것으로 보이는 SolidStart에 많은 관심이 필요해 보인다는 말을 덧붙이며 글을 마무리해보도록 하겠습니다. 만약 이해가 잘 되지 않는 내용이 있으시거나 잘못된 서술을 발견하신 분께서는 댓글로 남겨주시기 바라며, 저는 다음에도 좀 더 유용한 내용의 글들로 다시 찾아오도록 하겠습니다 😆