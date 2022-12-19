# 리액트 렌더링 및 렌더링 최적화

# 렌더링의 발생

렌더링(rendering)이란 쉽게는 **코드로 정의된 내용이 실제 브라우저 화면에 그려지는 과정**을 의미한다고 할 수 있다. 정확히는 브라우저 화면에 그릴 내용을 만드는 과정이라고도 볼 수 있는데, 리액트는 이 때 컴포넌트의 props와 state를 기반으로 UI를 만든다.
리액트가 컴포넌트를 렌더링하는 조건들은 다음과 같다.

1. **state의 교체(`setState`, `dispatch`)**
2. **부모 컴포넌트의 재렌더링**
3. `Root.render(...)` 함수 호출(react 18)
    - react 18 이전은 `ReactDOM.render(...)`
4. **props의 교체?**
5. **중앙 상태값(Redux store, Mobx store, Recoil atom, Context Api 등) 변화**

## state, props

state가 교체되었을 때(`setState`)에 리액트는 리렌더링을 queueing한다. 따라서 state가 교체되는 경우에 그 컴포넌트는 **무조건** 렌더링이 이루어진다. 또한 그 하위 컴포넌트들도 rendering queue에 들어가게 된다. 

props의 경우, 통상 props가 교체될 때 해당 props를 받은 컴포넌트가 렌더링되기는 한다. 그런데 이건 **결과**에 불과하고, 사실 리액트가 **props의 교체를 주시(또는 subscribe)하고 있는 것은 아니다.** 다음을 보면 알 수 있다.

```tsx
let testText = '1';
let testNum = 1;
const obj = { testText, testNum }

const PropsTestParentComponent = () => {

  const [renderer, setRenderer] = useState(false);

  const onClickText = () => {
    testText += '1';
    obj.testText += '1';
  };

  const onClickNum = () => {
    testNum += 1;
    obj.testNum += 1;
  }

  const onClick = () => console.log(testText, testNum);

  return (
    <div>
      <PropsTestChildComponent testNum={testNum} testText={testText} obj={obj}/>
      <button onClick={onClickText}>text</button>
      <button onClick={onClickNum}>num</button>
      <button onClick={onClick}>log parent</button>
      <button onClick={() => setRenderer(current => !current)}>render parent</button>
    </div>
  )
}

export default PropsTestParentComponent;
```

```tsx
type Props = {
  testNum: number;
  testText: string;
  obj: any;
}

const PropsTestChildComponent = ({testNum, testText, obj}: Props) => {
  const [renderer, setRenderer] = useState(false);
  return (
    <div id={renderer.toString()}>
      <h3>str: {testText}</h3>
      <h3>num: {testNum}</h3>
      <h3>obj: {obj.testNum} / {obj.testText}</h3>
      <button onClick={() => setRenderer((current) => !current)}>render child</button>
      <button onClick={() => console.log(testText, testNum, obj)}>log child</button>
    </div>
  );
}

export default PropsTestChildComponent;
```

위 화면은 다음과 같다.

![CPT2212091439-450x289.gif](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/CPT2212091439-450x289.gif)

위처럼 버튼을 눌러 보면, child 컴포넌트는 렌더링되지 않는다. 만약 child 컴포넌트가 props의 교체를 관찰하고 있었다면, text와 num 버튼을 눌러 값이 교체됐을 때 렌더링이 이루어져야 할 것이다. 

![Untitled](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/Untitled.png)

콘솔창에 나타나는 값은 다음과 같다.

![Untitled](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/Untitled%201.png)

**child가 props로 가지고 있던 값은 바뀌지 않는다.** 이는 child를 재렌더링해도 마찬가지이다.

![Untitled](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/Untitled.gif)

![Untitled](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/Untitled%202.png)

그러나 parent를 렌더링해 주면 child가 props가 교체된 대로 렌더링이 이루어진다.

![CPT2212121410-450x289.gif](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/CPT2212121410-450x289.gif)

즉, 일반적으로 리액트는 **props의 교체를 신경 쓰고 있지 않다. 정확히는 props가 immutable하기 때문에 이를 변경하는 것은 불가능하고, parent가 렌더링될 때 child에게 넘겨주는 props가 비로소 교체된다.** props변화를 따라 렌더링이 안되는 것은 이러한 이유이다.  다만 `obj`의 값이 바뀐 것은 child가 `obj`를 참조하고만 있기 때문이다.

흔히들 props가 변화하면 컴포넌트가 재렌더링된다고 말하지만, 개인적인 생각으로는 props**변화가 렌더링을 일으키는 것이 아니라,** **부모의 렌더링이 이루어질 때 (부모 함수 컴포넌트를 호출하면서) 비로소 자식 컴포넌트의** props**가 교체되고 이에 맞게 자식의 렌더링이 이루어지는 것**이다. 물론 이 때 props의 변화가 없더라도 렌더링은 이루어진다.

# 렌더링 과정

## 최초 렌더링

1. **함수 컴포넌트 호출**

이 단계에서는 정의된 함수 컴포넌트(JSX를 리턴하는 함수)를 실행한다. 컴포넌트 내의 props, state, **hook** 등은 리액트의 `Fiber` 객체([타입](https://github.com/facebook/react/blob/f101c2d0d3a6cb5a788a3d91faef48462e45f515/packages/react-reconciler/src/ReactInternalTypes.js), [참고](https://velog.io/@jangws/React-Fiber)*TODO)에 의해 관리되고, 이에 따라 함수 내 구현부를 준비한다. 그리고 이를 토대로 JSX를 리턴함으로써 본격적인 렌더링이 시작된다.

1. **렌더 및 커밋 단계(Render Phase and Commit Phase)**

렌더 단계에서는 새로운 가상 DOM을 생성한다. 그리고 커밋 단계에서 이를 실제 DOM에 반영한다.

1. **passive effects**

커밋 이후에는 [useLayoutEffect](https://reactjs.org/docs/hooks-reference.html#uselayouteffect)가 실행된다. 이는 useEffect와 달리 브라우저 화면에 가시적인 변화가 생기기 전에 동기적으로 실행된다. 그리고 짧은 `timeout` 세팅 이후 화면이 그려지고 [useEffect](https://reactjs.org/docs/hooks-reference.html#useeffect)가 실행된다. 이 단계에서도 `setState()`등의 원인으로 또다시 렌더링이 시작될 수 있다.

1. **결과**

렌더링 결과물이 DOM에 반영되고 화면이 그려진다. 물론 렌더링이 끝난 이후에도 가시적인 변화가 없을 수도 있다. 컴포넌트가 이전과 같은 렌더링 결과물을 리턴한다면 화면상에서는 아무런 변화가 없을 것이다. 또한 리액트18의 `[ConcurrentMode](https://reactjs.org/blog/2022/03/29/react-v18.html#gradually-adopting-concurrent-features)`에서는 브라우저 이벤트 처리를 위해 렌더링을 일시중지하는 것이 가능하다. 이 때 해당 작업은 다시 시작되거나 버리게 된다. 따라서 이 때 해당 컴포넌트는 여러 번 렌더링될 수 있다.

## 리렌더링

리렌더링은 [위](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a.md)에 나온 원인 중 하나에 의해 발생하고, 그 과정은 최초 렌더링과 다를 바 없다. 그러나 각 단계별로 이루어지는 일은 살짝 다를 수 있다.

1. **함수 컴포넌트 호출**

이 때 함수 내부의 변수 및 함수는 재생성된다. 다만 **hook**은 그 특성에 따라 재사용되기도 한다. 이후 이를 토대로 JSX를 리턴한다.

1. **렌더 및 커밋 단계**

렌더 단계에서 리액트는 새로운 가상 DOM을 생성한다. 그리고 리액트의 [Reconcilation](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a.md) 알고리즘?을 통해 기존 DOM과 새로운 DOM을 비교하여 변경된 컴포넌트에만 따로 렌더링을 위한 플래그를 지정해 둔다. 그리고 커밋 단계에서 플래그가 있는 컴포넌트들을 렌더링한다.

1. **그 이후**

나머지는 최초 렌더링과 같다. 다만 useEffect에 return함수가 있다면 useEffect의 구현부보다 먼저 (렌더링 이전의 상태값을 가지고) 실행된다.

# 렌더링 최적화

 렌더링 작업은 때때로 리소스를 낭비하게 될 수 있다. 컴포넌트의 렌더링 결과가 이전 DOM과 다르지 않다면 해당 컴포넌트를 렌더링하는 것은 시간, 자원의 낭비에 불과하다. 이를 해소하려면 1)렌더링을 더 빠르게 하거나 2)렌더링을 더 적게 해야 한다. 이 글에서는 앞서 살펴본 렌더링이 일어나는 조걱과 그 과정을 토대로 렌더링을 최소화하는 방법을 알아볼 것이다.

## Reconcilation

[React문서](https://reactjs.org/docs/reconciliation.html)

위 글을 요약하면, react는 두 렌더링 트리를 root부터 쭉 재귀적으로 비교하게 된다. 이 때 알고리즘은 엘리먼트의 타입이 바뀌었는가(1)와 key의 변경이 있는가(2)를 확인한다. 

### type diff

엘리먼트 또는 컴포넌트의 타입이 다른 경우, 예를 들어 <PrevComponent/>가 <NextComponent/>로 바뀌는 경우에는 해당 트리 자체와 그 아랫부분은 완전히 버려지고 새로 만들어진다. 이 때 이전 트리와 연결된 모든 state또한 사라진다. 따라서 불필요한 렌더링을 막으려면 같은 타입의 컴포넌트 인스턴스가 새로 생기는 것을 방지해야 한다. 그 예시는 다음과 같다.

```tsx
const Parent = () => {
	const Child = () => <div>My type changes every render</duv>  // Parent가 렌더링 될 때마다 instance가 새로 생성된다.
	return <Child/>
}
```

위 <Parent/>는 렌더링될 때마다 새로운 <Child/>컴포넌트의 인스턴스를 생성하기 때문에, <Child/>와 그 자식 컴포넌트들은 파괴되었다가 재생성된다. 이를 방지하려면 다음과 같이 <Child/>를 <Parent/>외부에 정의해야 한다. (또는 useMemo를 사용해야 한다)

```tsx
const Child = () => <div>My type is now immutable</div>
const Parent = () => {
return <Child/>
}
```

DOM 노드들의 자식들을 재귀적으로 비교할 때, react는 두 리스트를 돌면서 서로 차이점이 있을 때 변경을 결정한다. 아래 코드들은 react 문서에서 가져온 것들이다.

```tsx
// prev DOM
<ul>
  <li>first</li>
  <li>second</li>
</ul>
// next DOM
<ul>
  <li>first</li>
  <li>second</li>
  <li>third</li>
</ul>
```

위와 같은 경우 ul태그의 자식들 리스트 내에서 첫 번째와 두 번째는 바뀌지 않았으니 세 번째(<li>third</li>)만 렌더링 큐에 들어간다. 그런데 이러한 방식은 다음과 같은 문제가 발생할 수 있다. 

```tsx
// prev DOM
<ul>
  <li>Duke</li>
  <li>Villanova</li>
</ul>
// next DOM
<ul>
  <li>Connecticut</li>
  <li>Duke</li>
  <li>Villanova</li>
</ul>
```

이 경우, 두 리스트 안에서 비교하기 때문에 리액트는 자식들 리스트 내에서 첫 번째와 두 번째가 바뀌고 세 번째가 추가된 것으로 보아 모든 <li/>들이 다시 렌더링된다.

### key diff

방금과 같은 문제를 해결하기 위한 것이 `key`이다. `key`는 실제 컴포넌트로 전달되는 prop은 아니지만 리액트는 이를 토대로 컴포넌트의 인스턴스를 구분한다. 따라서 위의 문제를 다음과 같이 해결할 수 있다.

```tsx
// prev DOM
<ul>
  <li key="2015">Duke</li>
  <li key="2016">Villanova</li>
</ul>
// next DOM
<ul>
  <li key="2014">Connecticut</li>
  <li key="2015">Duke</li>
  <li key="2016">Villanova</li>
</ul>
```

이러면 이제 react는 `key=’2014’`인 엘리먼트가 추가되었고 `key=’2015’` 와 `key=’2016’` 인 엘리먼트는 이제 이동하기만 하면 된다는 것을 알게 된다. 다만 이 key는 (같은 부모 내에서) **고유하면서 불변하는 값으로 설정해야 한다.** 가끔 아래와 같이 key를 설정하는 경우가 있을 수 있는데, 이는 나중에 문제를 야기할 수 있다. 

```tsx
// todoList.length: 10
const TodoList = (todoItems: TodoItem[]) => {
	return (
		<Fragment>
			{todoItems.map((todoItem, index) => <Todo todo={todoItem} id={index}/>}
		</Fragment>
	)
}
```

위 컴포넌트에서 중간 3, 4, 5번째 컴포넌트를 지우고 새롭게 4개를 추가하면 `key`는 `0..10`이 된다. 이 때 `key`의 변화는 `key=10`인 컴포넌트 뿐이기에 리액트는 단순히 하나의 컴포넌트가 추가된 것으로만 인식하게 된다. 따라서 리액트는 `key`가 `0..9`인 컴포넌트는 재사용하게 된다. 기존 목록의 `todoItem`이 이전과 다른 데이터를 표시해야 하고 이를 위해 리액트는 목록의 아이템 중 3, 4, 5번째를 업데이트해야 하는데, 목록의 `todoItem`이 사실상 변한 것이 아니므로 리렌더링 및 업데이트가 필요하지 않은 것으로 간주된다([예제](https://ko.reactjs.org/redirect-to-codepen/reconciliation/index-used-as-key)). 따라서 불필요한 렌더링을 방지하고 컴포넌트의 정합성을 유지하기 위해서  `key` 에 `todo.id` 처럼 고유하고 불변하는 값을 넣어서 리액트가 똑바로 인식할 수 있도록 해야 한다. 

## React.memo✨

### 개요

react에는 렌더링을 줄여주는 마법 같은 함수가 존재하는데, 이게 바로 `memo(FunctionComponent, fn?)` 이다. `memo`는 react가 제공해 주는 함수로, 이 함수는 이 함수로 감싼 컴포넌트가 렌더링되려고 할 때 이전 props와 현재 props를 비교하여 바뀐 props가 없으면 렌더링을 하지 않도록 막아준다. 이 함수의 타입?은 다음과 같다.

```tsx
function memo<P extends object>(
        Component: FunctionComponent<P>,
        propsAreEqual?: (prevProps: Readonly<P>, nextProps: Readonly<P>) => boolean
    ): NamedExoticComponent<P>;
```

`Component`에는 `memo`를 감쌀 컴포넌트를 넣으면 된다. 그리고 `propsAreEqual`부분이 재밌는데, 여기 넣은 함수는 props를 비교하는 부분을 대체하게 된다. 이 함수의 output이 `true`이면 렌더링이 이루어지지 않고, `false`일 경우에만 렌더링이 이루어진다. 함수를 넣지 않을 경우 react는 props 각 항목들을 **일치 연산자**(strict equality, ===)를 사용해 비교한다. 예시를 들어 보자.

```tsx
type Props = { fixed: number, childCounter: number };

const MemoizedComponent = memo(({ fixed, childCounter }: Props) => {
  return (
    <div style={{ marginLeft: 10, marginRight: 10 }}>
      <hr/>
      <h3>fixed: {fixed}</h3>
      <h3>child counter: {childCounter}</h3>
      <hr/>
    </div>
  );
});

const MemoizationTestParentComponent = () => {
  const [childCounter, setChildCounter] = useState(1);
  const [parentCounter, setParentCounter] = useState(1);

  const onClickAddChildCounter = () => setChildCounter(current => ++current);
  const onClickAddParentCounter = () => setParentCounter(current => ++current);

  return (
    <div style={{ width: 500 }}>
      <h3>parent counter: {parentCounter}</h3>
      <h3>child counter in parent component: {childCounter}</h3>
      <button onClick={onClickAddParentCounter}>parent+</button>
      <button onClick={onClickAddChildCounter}>child+</button>
      <MemoizedComponent fixed={12345} childCounter={childCounter}/>
    </div>
  );
};
```

해당 컴포넌트는 다음과 같다.

![CPT2212140925-564x332.gif](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/CPT2212140925-564x332.gif)

위 화면에서 윤곽선이 깜빡이는 것은 렌더링이 이루어졌음을 의미한다. 원래대로라면 parent+버튼을 눌러서 `parentCounter` state가 교체되면 부모 컴포넌트인 `<MemoizationTestParentComponent/>`가 렌더링되면서 그 자식인 `<MemoizedComponent/>` 가 렌더링되어야 했지만, props인 `fixed`와 `childCounter`가 변하지 않았기에 자식 컴포넌트는 렌더링되지 않았고, child+버튼을 눌러 `childCounter` prop이 바뀌었을 때만 렌더링되었다.

`memo()`의 두 번째 파라미터를 넣어서 해 볼 수도 있다.

```tsx
type Props = { fixed: number, childCounter: number };
const arePropsEqual = (prev: Props, next: Props) => {
	return prev.fixed === next.fixed;
}

const MemoizedComponentWithDiffFn = memo(({ fixed, childCounter }: Props) => {
  return (
    <div style={{ marginLeft: 10, marginRight: 10 }}>
      <hr/>
      <h3>fixed: {fixed}</h3>
      <h3>child counter: {childCounter}</h3>
      <hr/>
    </div>
  );
}, arePropsEqual);
```

이러면 props중 `fixed`항목이 변하지 않는 이상 렌더링되지 않는다.

![CPT2212140924-611x389.gif](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/CPT2212140924-611x389.gif)

하지만 위 함수를 남용하는 것은 버그를 야기할 수 있다. 리액트에서도 `arePropsEqual`함수는 **최적화만을 위한 것**이므로, **렌더링을 방지하고자 사용하지 말라**고 [경고](https://reactjs.org/docs/react-api.html#reactmemo)한 바 있다. 게다가 컴포넌트마다 비교 함수를 넣는 것은 버그를 유발함은 물론이고 DX를 저해함과 동시에 유지보수를 힘들게 만든다.

### useCallback, useMemo

[앞서 언급했듯이](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a.md) 리렌더링이 이루어질 때 함수형 컴포넌트는 다시 호출되면서 그 구현부의 함수와 변수들은 재생성된다(새로운 instance로 교체된다). 그렇다면 부모 컴포넌트 내에서 정의된 함수와 변수는 그 인스턴스가 바뀌기 때문에 이를 props로 자식 컴포넌트에게 내리면 `memo`로 감싸더라도 리렌더링이 이루어질 수밖에 없다. 이를 방지할 수 있는 게 **useCallback**, **useMemo**이다. 이 두 **hook**은 컴포넌트 함수 내에 정의된 변수나 함수의 인스턴스를 고정해 준다(Fiber객체에서 관리되는 것 같다). 

또한 자식 컴포넌트 자체를 **useCallback**이나 **useMemo**로 감싸면 그 컴포넌트의 인스턴스가 마찬가지로 유지되기 때문에 자식 컴포넌트를 `memo` 로 감싼 것과 동일한 효과를 낼 수 있다.  사용법은 다음과 같다.
**useCallback**

```tsx
const onChange = useCallback((event: ChangeEvent<HTMLInputElement>) => {
	someFnDefinedInComponent(event.target.value);
	setValue(event.target.value + someValueDefinedInComponent);
// 2번째 파라미터에는 이 함수가 재정의되어야 하는 조건을 넣는다.
// 이 dependencies배열 안의 요소가 교체되지 않는 한 이 함수의 인스턴스는 유지된다.
// 컴포넌트 내에서 정의된 값들은 모두 넣는다고 보면 된다.
}, [someFnDefinedInComponent, someValueDefinedInComponent]);
```

의도치 않은 버그를 줄이려면, 컴포넌트 안에서 정의되었으면서 이 함수 내에서 사용되는 모든 변수나 함수는 deps에 넣어 주어야 한다.

**useMemo**

```tsx
const complicatedValue = useMemo(() => {
	return someComplicatedFunction(state1 + state2) + '입니다.'
// 2번째 파라미터에는 이 함수가 재정의되어야 하는 조건을 넣는다.
// 이 dependencies배열 안의 요소가 교체되지 않는 한 이 함수의 인스턴스는 유지된다.
// 컴포넌트 내에서 정의된 값들은 모두 넣는다고 보면 된다.
// someComplicatedFunction이 컴포넌트 내에서 정의되지 않았거나 단일 인스턴스임이 보장되면 넣지 않아도 된다.
}, [state1, state2, someComplicatedFunction]);
```

## Suspense, lazy

이 글과는 결이 살짝 다르지만, 코드 분할(code splitting)을 통해 앱을 최적화할 수도 있다. 코드 분할이란 쉽게 말해 번들을 쪼개는 것이다. SPA의 경우 기본적으로 minifyng, mangling 등을 거쳐 bundling되어 하나의 js파일을 가지고 페이지를 구성한다. 그런데 앱이 커질수록 번들링된 파일이 커져 사용자가 js파일을 다운받으면서 흰 화면을 보는 시간이 길어진다. 이를 보완하는 것이 code splitting이다. 코드 분할을 사용하여 컴포넌트를 import하면 그 컴포넌트가 렌더링될 때 js파일을 가져오게 된다. 다음은 react문서에 나온 예시이다.

```tsx
const OtherComponent = React.lazy(() => import('./OtherComponent'));

function MyComponent() {
  return (
    <div>
      <Suspense fallback={<div>Loading...</div>}>
        <OtherComponent />
      </Suspense>
    </div>
  );
}
```

`<Suspense/>` 컴포넌트는 `<OtherComponent/>` 가 로딩 중일 때 fallback의 내용을 보여주기 위해 사용된다. 이를 `[ErrorBoundary](https://reactjs.org/docs/error-boundaries.html#gatsby-focus-wrapper)`로 묶어서 사용할 수도 있다.

## batch, asynchronous

리액트에서 batch는 여러 `setState` 를 통해 하나의 렌더 패스가 대기열에 저장된 후 실행되는 것을 의미한다. `setState`를 호출하여 state를 교체하면 리액트는 새로운 렌더링 패스를 시작한다. 이는 동기적으로 실행되는 것이 기본이지만, 리액트는 자동으로 렌더링 배치를 최적화해 준다. 리액트 이벤트 핸들러는 내부적으로 `instability_batchedUpdates`라는 함수로 래핑되어 있는데, 이는 렌더링 큐에 대기 중인 모든 상태 업데이트를 추적하여 단일 렌더링으로 만들어 준다.

### react 18 이전 - unstable_batchUpdates

그런데 비동기 함수를 만나면 batch는 생각하는 대로 이루어지지 않을 수 있다. 다음 코드를 보자.

```tsx
const [counter, setCounter] = useState(0);
const [alertText, setAlertText] = useState('');

const onClickAdd = async () => {
	setCounter(1); // render 1
	fetchCounter()
		.then(response => {
			setCounter(response); // render 2
			setAlertText('completed'); // render 3
		}); 
	setCounter(2); // render 4
}
```

위 코드를 실행하면 렌더링이 한 번만 이루어질 것 같지만, 아쉽게도 네 번 이루어진다. 리액트 batch는 콜 스택 외부에 대기 중인 상태 업데이트와는 함께 이루어지지 않는다. 그래서 `setCounter(1)` → `setCounter(response)` → `setAlertText` → `setCounter(2)` 가 모두 다른 batch에서 동기적으로 실행된다.

이를 최적화하고 싶다면 batch를 묶어 주면 된다. ReactDOM은 `unstable_batchedUpdates`라는 함수를 가지고 있는데, 이는 batch를 묶어 주는 기능을 한다.

```tsx
const [counter, setCounter] = useState(0);
const [alertText, setAlertText] = useState('');

const onClickAdd = async () => {
	setCounter(1); 
	setCounter(2); // render 1
	fetchCounter()
		.then(response => {
			unstable_batchedUpdates(() => {
				setCounter(response); 
				setAlertText('completed'); // render 2
			}
		}); 
}
```

번외로 커밋단계의 라이프사이클 메소드에는 `useLayoutEffect`와 같은 엣지 케이스가 존재하는데, 이는 주로 브라우저가 페인팅을 하기전에 렌더링 후 추가 로직을 수행할 수 있도록 하기 위해 존재한다. 일반적인 사용사례는 다음과 같다.

- 불완전한 일부 데이터로 컴포넌트를 최초 렌더링
- 커밋 단계 라이프 사이클에서 DOM 노드의 실제 크기를 `ref`를 통해 측정하고자 할 때
- 위 측정을 기준으로 일부 컴포넌트의 상태 설정
- 업데이트된 데이터를 기준으로 즉시 리렌더링

이러한 사용사례에서, 초기의 부분 렌더링된 UI가 사용자에게 절대로 표시되지 않도록 하고, 최종 UI 만 나타날 수 있게 한다. 브라우저는 수정중인 DOM 구조를 다시 계산하지 자바스크립트는 여전히 실행중이고,이벤트 루프를 차단하는 동안에는 실제로 화면에 아무것도 페인팅하지 않는다. 그러므로, `div.innerHTML = 'a'`, `div.innerHTML='b'` 같은 작업을 수행하면 `a`는 나타나지 않고 `b`만 나타날 것이다.

이 때문에 리액트는 항상 커밋 단계 라이프사이클에서 렌더링을 동기로 실행한다. 이렇게 하면 부분적인 렌더링을 무시하고 최동 단계의 렌더링 내용만 화면에 표시할 수 있다.

### react 18 - automatic batching

그런데 react 18부터는 자동 batch가 더욱 강화되어, `unstable_batchedUpdates` 를 사용하지 않아도 자동으로 batch가 묶인다. 

## 기타 최적화 도구

### useTransition, useDeferredValue

이 **hook**은 react 18버전에서 새로 나왔는데, 이는 디바운싱(debounce)를 통해 렌더링 batch를 뒤로 미루어 주는 역할을 수행한다. 다음과 같은 컴포넌트를 가정해 보자.

```tsx
const NoAnswerRenderingComponent = () => {
	const [input, setInput] = useState<string>('');
	const onChange = () => setIn
	return (
		<div>
			<input onChange={onChange}/>
			{new Array(10000).fill(0).map(() => <div>{input}</div>)}
		</div>
	)
}
```

이 컴포넌트의 input에는 입력 지연이 생긴다. (좋은 컴퓨터라면 안 생길 수도 있다...)

![CPT2212190950-487x742.gif](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/CPT2212190950-487x742.gif)

여기에 useTransition을 사용하면 다음과 같이 입력 지연이 사라진다.

```tsx
export const NoAnswerRenderingComponent = () => {
  const [input, setInput] = useState<string>('');
  const [isPending, startTransition] = useTransition();
  const onChange = (event: ChangeEvent<HTMLInputElement>) => startTransition(() => setInput(event.target.value));

  return (
    <Fragment>
      <input onChange={onChange}/>
      {
        isPending ? <div>loading...</div> : new Array(10000).fill(0).map(() => <div>{input}</div>)
      }
    </Fragment>
  );
};
```

isPending은 transition이 이루어지고 있음을 알려주는 `boolean`값이다.

![CPT2212191002-483x757.gif](%E1%84%85%E1%85%B5%E1%84%8B%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%85%E1%85%A6%E1%86%AB%E1%84%83%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20%E1%84%8E%E1%85%AC%E1%84%8C%E1%85%A5%E1%86%A8%E1%84%92%E1%85%AA%203ed689d31d9b437589f8c0fd9c3cca4a/CPT2212191002-483x757.gif)

[useDeferredValue](https://reactjs.org/docs/hooks-reference.html#usedeferredvalue)를 사용할 수도 있다.

그런데 사실 이러한 방식은 렌더링이 빨라지게 만드는 것이 아니라, 그저 렌더링 타이밍을 지연시킬 뿐이므로 근본적인 해결책이라고 보기기는 어렵다. 페이지를 나누거나 pagination(infinite scroll 등)을 적용하는 것이 더 나은 해결책이라고 본다.