---
layout: post
title:  "React Native Web 파헤치기 2"
date:   2024-12-21 01:30:00 +0900
categories: react-native react-native-web
---

# 알고리즘 책의 1장에 나오는 Big O
알고리즘 책의 첫장에는 시간복잡도 라는 게 나오는데요. 여기서 `Big O` 라는 표기법이 나옵니다. 알고리즘의 최악의 성능을 나타낼 때 쓰이는 표기법이지요.
여기서 재미 있는 부분은 연산 수치에 크게 영향을 주지 않는 것에 대해서는 버림 처리를 한다는 것 입니다.

`3n + 10000` 연산이 필요한 알고리즘의 big O는 `O(n)`이 됩니다. 왜냐하면 `n`이 무한이 커지면 상수들은 무시해도 될 만큼 작아지기 때문입니다.

# 게시물 주제가 "React Native Web 파해치기 2" 인데? 뜬끔 알고리즘???
react를 사용할 때 useCallback, useMemo, React.memo 등을 사용하면 성능 개선을 할 수 있다는 이야기를 많이 하더라고요.
그런데 이게 얼마나 성능에 긍정적인 영향을 미치는지는 잘 모르겠습니다... 
알고리즘에 나오는 Big O 시간 복잡도 처럼 n에 대한 연산을 줄이는 걸 목표로하는 게 중요하지 않을까는 생각을 자주하게 됩니다.

**_IMHO_**: 성능저하가 발견되지 않았는데, useCallback, useMemo, React.memo를 남발하는 것은 그렇게 좋지 않은 거 같습니다. 가성비가 떨어지는 거 같아서요.

# 게임에서 흔히 보이는 렌더링 최적화
아래 영상은 게임에서 카메라 외부 요소들이 어떻게 사라지는 지를 보여줍니다.

![game-rendering-optimazation-example.gif](/assets/img/game-rendering-optimazation-example.gif)

아주 그래픽 자원을 아끼기 위한 노력에 저는 눈물을 흘립니다... 웹, 모바일 개발자 여러분들도 지금 같이 눈물을 흘리고 계시나요?

<img src="/assets/img/crying-nb.png" width="300px" alt="crying-nb.png"/>

그럼 어떻게 눈물나는 렌더링 최적화를 react-native-web에서는 어떻게 하고 있는 지 궁금하지 않습니까?

# React Native Web의 VirtualizedList

VirtualizedList 컴포넌트는 react-native에서 FlatList의 기반이 되는 컴포넌트 입니다.
최적화 렌더링을 위해 Viewport 밖에서 눈물의 쇼를 하는 컴포넌트이지요.

이제 [react-native-web의 VirtualizedList](https://github.com/necolas/react-native-web/blob/master/packages/react-native-web/src/vendor/react-native/VirtualizedList/index.js)를 살펴보겠습니다.

![react-native-web-virtualized-list-1.png](/assets/img/react-native-web-virtualized-list-1.png)

우선 상태값 부터 보겠습니다.

```
type State = {
  renderMask: CellRenderMask,
  cellsAroundViewport: {first: number, last: number},
};
```

cellsAroundViewport 라는 게 있군요. 네이밍에서 알 수 있듯이 Viewport 주변에 있는 cell의 번호를 기억하는 용도로 보입니다. 
왜 이런 걸들일 기억해야할지 상상을 해보고 다음 코드를 이어가보겠습니다.

> 상상
> 
> 실질적으로 렌더링 되고 있는 cell의 첫번째 번호와 마지막 cell의 번호를 알고 있다면, 
> 위의 게임 렌더링 영상 처럼 viewport(In Game, 카메라 시아) 주변에 있는 것들 첫번째 항목 부터 마지막 항목까지만 렌더링하게 처리하면 최적화 렌더링할 때 아주 유용할거 같군요?!
> 사용자들이 볼 수 있는 Viewport와 그 주변에 있는 것들만 렌더링한다면 사용자들은 자신들이 보고 싶은 요소들을 볼 수 있고 개발자 입장에서는 컴퓨터의 연산을 줄이고, 
> 메모리를 적게 사용할 수 있을 터이니 아주 서로 윈윈 이겠군요. 개꿀?!

그 다음 부터는 특정 상황에서 쓰기 위한 메소드들이 다수 보입니다. 
scrollToEnd, scrollToIndex, scrollToItem 등등 이런 것들은 최적화 렌더링 보다는 스크롤과 관련 있는 것이기에 가볍게 넘어가줍니다.

constructor는 그래도 유심히 봐주어야 할 거 같군요. FillRateHelper와 Batchinator 그리고 initialRenderRegion가 중요해보입니다.

![react-native-web-virtualized-list-2.png](/assets/img/react-native-web-virtualized-list-2.png)

이것들에 대해서 알아봅시다.

## FillRateHelper ??
`VirtualizedList`의 최대 충족률을 감지하기 위한 헬퍼 클래스라는 군요?!

특정 인덱스의 프레임 지표를 구하기 위한 함수를 받는 생성자를 가졌군요. 
흠?? 여기서 index는 특정 프레임이 존재 유무와  이 프레임의 길이, 오프셋, 레이아웃 내에 있는 지를 지표로 얻을 수 있는 거 같군요?
얻을 수 있는 지표 정보를 보니 성능 측정을 위한 FPS는 아닌 거 같고 어떤 렌더링 요소의 길이, 오프셋, 인덱스, viewport 내에 있는 지를 나타내는 것이 아닐지 추측해보고 넘어가겠습니다.

감지를 활성화하는 activate 메소드와 감지를 비활성화하고 감지를 비활성화하고 데이터를 날리는 deactivateAndFlush 메소드가 있군요.
그리고 그 다음 메소드에는 공백을 계산한느 computeBlankness 메소드가 있네요.

오?!? computeBlankness에 아주 흥미로운 것들이 많습니다.
param에 초기 렌더링에 표시할 항목수를 결정하는 initialNumToRender가 있고 VirtualizedList의 상태값 cellsAroundViewport 그리고 스크롤 지표(측정항목)들이 있군요.

흠... Metrics는 이제 "지표"가 아닌 측정하고 있는 항목들의 집합... 더 간단하게는 상태값으로 해석하는 맞는 거 같네요.

cellsAroundViewport은 어떻게 초기화되는 지 봐야겠습니다.

![react-native-web-virtualized-list-3](/assets/img/react-native-web-virtualized-list-3.png)

firstCellIndex는 일반적으로 0이 되지만, 리스트의 렌더링 초기에 처음으로 보여줄 항목을 결정하는 initialScrollIndex prop에 의해서 0이 아닐 수도 있군요.
lastCellIndex은 일반적으로 `(firstCellIndex + 처음에 렌더링하려는 항목의 수)-1`이 되는군요.


![react-native-web-virtualized-list-4.png](/assets/img/react-native-web-virtualized-list-4.png)

`cellsAroundViewport.last < cellsAroundViewport.first` 가 왜 필요한지 어느 정도 추측이 될 거 같습니다.
쳣번째로 렌더링 될 항목이 마지막으로 렌더링 될 항목보다 크다면, cellsAroundViewport.first 이후에 렌더링 될 요소가 없을 거 같군요?!
그래서 공백이 필요 없으니 0?!?!

오... 하나 얻어갑니다. velocity에 1000을 꼽하면 초당 스크롤 px을 알 수 있군요!

```
const scrollSpeed = Math.round(Math.abs(velocity) * 1000); // px / sec
```

그 다음 부터는 blankTop을 구하는 군요. 상단 공백 크기를 구하는 부분으로 같습니다.
Viewport 외각에서 렌더린 된 첫 번째 항목의 번호를 기반으로 이 항목의 렌더링 정보를 _getFrameMetrics를 이용하여 가져오는군요.
getItemLayout으로 아래 정보들을 가져옵니다.

```
(
  data: any,
  index: number,
) => {length: number, offset: number, index: number}
```

첫번째 렌더링 요소를 찾거나 마지막 셀의 번호 보다 작을 때 까지 순회하는군요.
그러다가 처음 렌더링하는 항목이 0번째 항목이 아닌 경우, 그러니까 initialScrollIndex prop으로 지정한 경우에만 blankTop을 계산하는군요.
기본적으로 blankTop은 0 이 되는군요. 지금 생각해보면 당연한 결과군요...

음... 그렇군요. 너무 힘을 많이 빼버렸군요. 여기 연산은 리스트의 해더의 높이와 푸터의 높이에 대한 공백 크기를 구하기 위한 연산이었군요?
음... 한번 더 상상을 해봐야겠군요.

```
if (firstFrame && first > 0) {
  blankTop = Math.min(
    visibleLength,
    Math.max(0, firstFrame.offset - offset),
  );
}
```

> 상상 - 처음 렌더링 되는 요소가 만약 10번째 요소이고 0 ~ 9 번째 요소는 렌더링 되어 있지 않다면?
> 
> ... 아 상상이 어렵다... visibleLength 값은 무엇을 의미하는 것일까? 
> visibleLength은 리스트의 높이를 의미하는 거 같고 offset은 스크롤이 이동한 범위 같다.
> blinkTop = Math.min(2000, Math.max(0, 1000 - 1150))
> 에헤이... 0으로 나오네... 정말 맞는 것일까???!?!?

![react-native-web-virtualized-list-5.png](/assets/img/react-native-web-virtualized-list-5.png)

찜찜하여서 넘어가지 못 하겠다. Expo로 간단한 샘플 앱을 만들어서 [확인](/assets/video/VirtualizedList-FillRateHelper-blankTop.mp4)해보자.

[Expo에서 확인해보기](/assets/video/VirtualizedList-FillRateHelper-blankTop.mp4)

FillRateHelper의 computeBlankness 메소드는 렌더링 초기 및 스크롤 이벤트 발생 시에도 실행되지 않는다.
그렇다는 거는 이 부분은 잠시 미뤄두고 다음 코드를 살펴보면 되겠다.

## "react-native-web 파헤치기 - 2"을 마치며

상상과 예측과 분석을 통해서 코드를 파헤치는 것은 정말 재미있는 일이다.
이번에는 VirtualizedList의 FillRateHelper를 살펴보았다.
다음에는 Batchinator를 살펴보면서 VirtualizedList의 최적화 렌더링에 대해서 더 자세히 알아보겠다.

[docs react-native-web]: https://necolas.github.io/react-native-web/
[github react-native-web]: https://github.com/necolas/react-native-web
