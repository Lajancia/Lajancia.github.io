---
title: "Next14 Prerendering 성능 개선 딜레마"
date: 2025-01-01 08:00:00 +0900
categories: [개발]
tags: [Next14, Prerendering, PPR, SSR, CSR, 성능최적화, React]
author: L.J
---

## Next14 Prerendering

### **성능 개선의 딜레마**

종종 여러 인프라 개선을 위해 무언가를 적용해 보면, 꼭 기존에 사용하던 장점을 버려야 하는 경우가 발생한다. 프레임워크나 라이브러리의 한계라고도 볼 수 있을 것 같다. 블로그에도 추후 다루게 될 이슈이기도 하지만, 최근 이러한 개선과 관련된 업무를 진행하는 과정에서 Next14 프레임워크의 여러 세밀한 기능적 부분에 대해 충분한 이해가 없으면 해결하기 쉽지 않은 이슈에 봉착했다. 그 중 하나는 Next14의 Prerender였다. ~~물론 이외에도 생각보다 내가 제대로 이해하지 못한 부분이 여전히 많다. 업데이트 할 떄 마다 확확 바뀌니.... 살려줘~~

이슈 내용은 다음과 같다. 현재 특정 태그를 API로 불러와 처리해야 하는 방식으로 변경할 경우, Prerendering을 사용하지 못한다는 것이다.

### **Prerendering VS Partial Prerendering**

Prerendering에는 위와같이 기존의 Prerendering과 Partial Prerendering이 있다. 사실상 기존에 Prerendering이라고 부르는 것은 CSR과 SSR 랜더링을 가리키는 것으로 보면 된다. 각각의 랜더링 방식이 기본적으로 Prerendering을 기반으로 하고 있기 때문이다. 다만 CSR은 클라이언트 측에서 해당 static 요소들을 미리 랜더링하는 것이고, SSR은 서버 사이드에서 미리 랜더링을 하는 차이일 뿐이다.

문제는 이러한 Prerendering이 수행되기 위해서는 해당 요소들이 기본적으로 Static 상태여야 한다는 것이다. page router에서는 getServerSideProps를 통해 동적 데이터들도 prerendering 되지만, app router에서는 RSC 방식으로 해야 하며, 그마저도 CSR에서는 Prerendering이 불가하다는 문제도 있다.

결국 간단히 정리하면, API로 dynamic 하게 데이터를 호출해오는 경우, 현재 CSR이 적용되어 있는 페이지에서는 제대로 Prerendering이 되지 않는다. 때문에 특정 태그를 API 방식으로 불러와 처리하는 방법은 Prerendering 성능을 뺄 만큼 중요하다고 생각되지는 않아 변경하지 않기로 결정되었다.

### **Next14 PPR (Partial PreRendering)**

<https://nextjs.org/learn/dashboard-app/partial-prerendering>

Prerendering에 대해 알아보다, Partial Prerendering이라는 것이 있다고 하여 한번 정리해보았다. 공식 홈페이지에서도 해당 Partial Prerender에 대해 자세히 설명하고 있다.

Next.js에서는 static routes 혹은 Dynamic routes를 선택하여 사용할 수 있지만, 만약 Dynamic function을 라우터에서 호출할 경우, route는 통째로 dynamic해지는 문제가 있다.

하지만 알다시피, 대부분의 라우터는 완전히 static하거나 완전히 dynamic 할 수는 없다. 예를 들어 아래와 같이 일부는 static 하고 일부는 dynamic 하기를 원한다.

![](https://blog.kakaocdn.net/dna/VLFS8/btsLBdNQP5U/AAAAAAAAAAAAAAAAAAAAABCkOtd_R7EasgQuGhtm_W1c3iI4gwL6bUmHPpQTqvLY/img.png)

Prerendering을 하게 되면 위와 같이 static 부분에 대해서만 미리 서버에서 랜더링을 하고, dynamic은 호출 시에 랜더링 된다. 기준이 되는 것은 Suspense 태그이다. 이 Suspense 태그 내부의 컴포넌트는 Dynamic으로 가져오고, 바깥 태그는 static으로 PPR 된다. 이때 Dynamic으로 가져오는 동안 Suspense를 통해 가져와지는 부분들에 대하여 Skeleton을 적용하여 CLS를 방지할 수 있다.

### **SSR, CSR, PPR**

부분 사전 랜더링에 대해 보다보니, SSR과 CSR 방식의 랜더링과 조금 헷갈리기 시작한다. 언뜻 보기에 PPR은 SSR의 특징과 CSR의 특징을 둘 다 가졌기 때문이다.

간단하게 이야기하면 PPR = SSR + CSR 과 같이 각각의 랜더링 방식을 합친 방식이지만, SSR과 CSR이 유저가 요청할 때 랜더링 된다면, PPR은 빌드 시점에 해당 정적 쉘들이 생성된다. 때문에 SSR+PPR 조합이나 CSR+PPR 조합도 가능하다. 이러한 방식을 통해 초기 로딩 시간을 개선할 수 있다.

코드 예시는 다음과 같다.

```javascript
const nextConfig = {
  experimental: {
    ppr: 'incremental',
  },
};

export default function Page() {
  return (
    <main>
      <header>
        <h1>My Store</h1>
        <Suspense fallback={<CartSkeleton />}>
          <ShoppingCart />
        </Suspense>
      </header>
      <Banner />
      <Suspense fallback={<ProductListSkeleton />}>
        <Recommendations />
      </Suspense>
      <NewProducts />
    </main>
  );
}
```

위와 같이 Suspense를 경계로 static, dynamic 동작이 갈리게 된다. 아무래도 호출 시점에 prerendering을 하는 부분을 포함하여 미리 static한 부분을 빌드 시점에 쉘 스크립트로 만들어두어 사용할 수 있다는 점에서 추후 정식 기능으로 포함되면 도입을 고려해보아도 좋을 듯 하다.