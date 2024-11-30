# 2024년 11월 29일 (금)
> 화살표 함수 vs 그냥 함수

## 개요
JS에서 함수를 만들 수 있는 방법은 두가지이다.
초기 JS에서 사용되던 `function`과 2015년 ES6에서 발표된 `arrow function`이 있다.

프론트엔드 개발할 때, 어떨 땐 `function`을 쓰는 것 같기도 하고, 어떨 땐 `arrow function`을 쓰는 것 같아서 이번에 정리해봤다.

## 주요 차이점

### Syntax
`function`은 아래와 같이 선언할 수 있다.

```js
function add(a, b) {
  return a + b;
}
```

이를 `arrow function`으로 변환하면 아래와 같이 선언할 수 있다.

```js
const add = (a, b) => {
  return a + b;
}
```

이 때, 구문이 한 줄이고, 그걸 return하는 경우 중괄호를 생략하여 아래와 같이 줄일 수 있다.

```js
const add = (a, b) => a + b;
```

이렇게만 본다면 `arrow function`을 안 쓸 이유가 없어보이는데, 왜 아직 `functin`이 존재하는걸까?

### `this` 바인딩
이 문제로 인해 Nest의 클래스에서 inject받은 인스턴스를 this로 접근하였는데 잘 안되었던 경험이 있다.

일반적인 `function`은 `this`가 동적으로 바인딩된다.

즉, 함수가 진짜 호출되는 시점에 `this`가 어떤 값인지 결정되는 것이다.

아래 예제 코드를 보자.

```js
const obj = {
  name: "Function",
  greet: function () {
    console.log(this.name); // "Function" (호출 시점에 `this`가 obj로 바인딩)
  },
};
obj.greet();
```

이 코드의 출력값을 뭘까?

호출 당시 greet의 `function`의 `this`는 바로 자신의 상위은 `obj`가 될 것이다.

따라서, `obj.name`인 "Function"이 출력된다.

이 코드를 그대로 `arrow function`으로 바꿔보자.

```js
const obj = {
  name: "Arrow",
  greet: () => {
    console.log(this.name);
  },
};
obj.greet();
```

이 코드또한 당연히 `this`가 `obj`를 들고있을 것 같지만, undefined가 나오게 된다.

왜냐하면 `arrow function`은 정의된 위치의 상위 Context의 `this`와 같기 때문이다.

### 호이스팅

호이스팅이라 하면, 인터프리터가 함수 선언을 먼저 찾아 선언 후, 나머지 코드가 돌아가게 하는 방식이다.

즉, 함수 선언을 나중에 할 수 있다.

```js
hi()

function hi() {
  console.log('Hi');
}
```

아래 코드에서 일반적으로는 `hi`라는 함수가 선언 전 사용되었기 때문에 오류가 날 것 같지만, 정상적으로 작동한다.

하지만 `arrow function`은 주로 변수에 담아서 사용하기 때문에, 호이스팅을 당할 자격을 가지고 있지 않다.

```js
hi()

const hi = () => { console.log('Hi'); }
```

당연히 이 코드는 오류가 날 것이다.

이러한 문제를 특히 React 계열에서 종종 생각하게 되는데, 나는 코드를 짤 때 `hook`을 맨 위에, 기능을 아래에 선언하는 편이다.

만약 `hook`에서 특정 함수를 필요로 한다고 하자.

```jsx
export default function Page() {
  const form = useForm<ApplicationRequest>({
    resolver: zodResolver(ApplicationRequestSchema),
    onSubmit: onSubmit,
  }); 
  
  const onSubmit = useCallback(async () => {}, []);
}
```

이 코드는 `hook` 아래에 필요한 함수가 정의되었기 때문에 onSubmit을 찾을 수 없을 것이다.

만약 `onSubmit`을 `function`으로 바꾸게 된다면 정상적으로 작동할 것이다.