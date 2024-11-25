# 2024년 11월 25일 (월)
> Typescript 추론 속도 올리기

## 개요
최근 Typescript 기반 프로젝트를 진행하다가, 타입 추론 속도가 매우 느린 현상을 겪었다.

그냥 내 노트북이 낡아 모든 IDE가 느리기에 이것도 그런 문제인 줄 알았으나, 이건 너무 심한 것 같아 이에 대해 찾아보았다.

일단, Typescript CLI에서 `--extendedDiagnostics` 옵션을 통해 컴파일러가 소요한 시간을 아래처럼 확인할 수 있다.

```
Files:                         6
Lines:                     24906
Nodes:                    112200
Identifiers:               41097
...
commentTime time:          0.00s
I/O Write time:            0.00s
printTime time:            0.01s
Emit time:                 0.01s
Total time:                1.75s
```

이렇게 trace를 쫓아가다가 특정 컴포넌트에서 오래 걸리고 있는 것을 확인했다.

## 원인
```tsx
<Gap component={"div"} gap={2} direction={"row"} >
</Gap>
```
이처럼 특정 컴포넌트(HTML Component)에 flex를 적용시켜주는 것이다.

이 때, 그 특정 컴포넌트를 입력받기 위해 `JSX.IntrinsicElements`의 키를 사용한다.

```ts
type HTMLElements = keyof JSX.IntrinsicElements;

type Direction = 'row' | 'col';

type GapProps<E extends HTMLElements> = JSX.IntrinsicElements[E] & { direction: Direction, gap: number };

export const Gap = <E extends HTMLElements>({ direction, gap, ...props }: GapProps<E>) {
    // 어쩌고
}
```

이 때, `JSX.IntrinsicElements`는 총 175개의 유니온 타입이기에, Gap 컴포넌트를 정의하는 순간, 175개의 하위 타입이 모두 정의되고 있다.

## 개선
- Type Narrowing
- Dynamic Type Inference

### Type Narrowing
위 코드를 아래처럼 자주 사용되는 Element로 좁혀보자.
```diff
- type HTMLElements = keyof JSX.IntrinsicElements;
+ type HTMLElements = 'div' | 'section' | etc... ;
```

이러면 typescript이 타입 추론할 때, 몇 개 안되는 타입만 추론하면 되므로 속도가 매우 빨라진다.

하지만, 만약 새롭게 HTMLElement를 추가하고 싶다면, HTMLElements의 타입을 수정해야하는 번거로움이 있다.

### Dynamic Type Inference
Typescript가 런타임 입력값을 바탕으로 타입을 추론하도록 하는 것이다.
> 주로 Generic으로 구현된다.

```ts
type HTMLElements = keyof JSX.IntrinsicElements;

type HTMLElementType<P = any> = {
    [K in HTMLElements]: P extends JSX.IntrinsicElements[K] ? K : never
}[HTMLElements] | ((props: P) => JSX.Element)
```

이렇게 수정한다면, P에 들어오는 값을 기반으로 런타임으로 계산하여 만약 필요하지 않는 값이라면 never을 사용한다.
