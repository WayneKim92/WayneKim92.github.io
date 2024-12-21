---
layout: post
title:  "React Native Web 파헤치기 2 - 2"
date:   2024-12-21 19:26:00 +0900
categories: react-native
---

# Batchinator ?

[Batchinator](https://github.com/necolas/react-native-web/blob/master/packages/react-native-web/src/vendor/react-native/Batchinator/index.js)의 코드는 짧으니, 처음 부터 하나씩 씹어먹어보겠습니다.
InteractionManager를 import 하는군요. InteractionManager는 RN에도 있는 API 입니다.
[공식 문서](https://reactnative.dev/docs/interactionmanager)에는 상호작용/애니메이션이 완료 된 후에 오랫동안 실행되는 작업에 대해 예약할 수 있는 API라고 하는 군요.

이 API를 사용해본 적이 없기 때문에... 왜 필요할지 한번 추측/생각 해보고 넘어가봐야할 거 같습니다.
일반적으로 React Native에서 JS Thread는 하나만 사용할 수 있지요?! 우리가 실행하는 작업이 완료되기 까지 오랜 시간이 걸리는 작업이라면,
이 작업이 완료되기 까지 다른 작업을 실행할 수 없게 됩니다. 이런 경우에 InteractionManager를 사용하면, 이점이 있는 것으로 추측해볼 수 있을 거 같습니다.

특히 자바스크립트 애니메이션 작업을 부드럽게 처리할 수 있다는군요?!

```javascript
InteractionManager.runAfterInteractions(() => {
  // ...long-running synchronous task...
});
```

터치 핸들링 시스템은 터치를 상호작용으로 간주하고 모든 터치가 종료되거나 취소될 때까지 runAfterInteractions의 콜백을 지연한다고 하는군요.
그럼 사용자의 반응이 발생 중 일 때에는 콜백을 실행하기 않기 때문에 애니메이션 작업에 영향을 주지 않는 콜백이 등록 되는 것으로 보면 되겠군요.

InteractionManager에 대한 이해도가 올라갔으므로 Batchinator 코드를 다시 보겠습니다.

Batchinator의 주석을 보니, 더 명확해지는 군요. 낮은 우선순위의 콜백 호출을 일괄 처리하기 위한 클래스라는군요.
예약 된 횟수와 상관 없이 기다렸다가 오직 한번 실행한다는군요.

그럼 Batchinator에 대해서 파악되었군요. 우선 순위 높은 애니메이션 작업 부터 실행 될 수 있게 하여 UX를 높이기 위해 필요한 것이군요.
이제 JS Thread에 의해서 실행되는 작업이 많을 때, InteractionManager를 사용하여 UX를 고려하여 특정 연산을 처리할 수 있을 거 같습니다.

# VirtualizedList 코드 이어서 보자

VirtualizedList의 생성자 코드를 보다가 여기까지 왔는데요. 다시 생성자 코드로 돌아가서 순차적으로 살펴보겠습니다.

FillRateHelper로 List의 상단과 하단 높이를 측정하는 데, 일반적으로 0이므로 크게 신경쓰지 않아도 되겠군요.
Batchinator는 우선순위가 낮은 콜백을 일괄 처리하기 위한 클래스로 이해했으니, 이것도 크게 신경쓰지 않아도 되겠군요.

다시 최적화 렌더링을 이해하기 위한 코드를 찾아나서겠습니다.
viewabilityConfigCallbackPairs prop은 필수 prop이 아니므로 패스해도 될 거 같습니다.

initialRenderRegion은 보아야 될 거 같군요. 우리가 처음 렌더링 될 요소에 대해서 결정하는 부분이니까요.

```typescript
constructor(props)
{
  //...
  const initialRenderRegion = VirtualizedList._initialRenderRegion(props);

  this.state = {
    cellsAroundViewport: initialRenderRegion,
    renderMask: VirtualizedList._createRenderMask(props, initialRenderRegion),
  };
  //...
}

static _initialRenderRegion(props: Props): {first: number, last: number} {
  const itemCount = props.getItemCount(props.data);

  const firstCellIndex = Math.max(
    0,
    Math.min(itemCount - 1, Math.floor(props.initialScrollIndex ?? 0)),
  );

  const lastCellIndex =
    Math.min(
      itemCount,
      firstCellIndex + initialNumToRenderOrDefault(props.initialNumToRender),
    ) - 1;

  return {
    first: firstCellIndex,
    last: lastCellIndex,
  };
}
```

_initialRenderRegion으로 렌더링 될 cell의 범위를 구하는군요. 그 후 VirtualizedList의 state이 초기화 됩니다.
renderMask는 VirtualizedList._createRenderMask로 만들어지는데요. 렌더링 시에 중요한 역할을 할 것으로 추측됩니다. 한번 살펴보아요!

```typescript
static _createRenderMask(
  props: Props,
  cellsAroundViewport: {first: number, last: number},
  additionalRegions?: ?$ReadOnlyArray<{first: number, last: number}>,
): CellRenderMask {
  const itemCount = props.getItemCount(props.data);

  invariant(
    cellsAroundViewport.first >= 0 &&
    cellsAroundViewport.last >= cellsAroundViewport.first - 1 &&
    cellsAroundViewport.last < itemCount,
    `Invalid cells around viewport "[${cellsAroundViewport.first}, ${cellsAroundViewport.last}]" was passed to VirtualizedList._createRenderMask`,
  );

  const renderMask = new CellRenderMask(itemCount);

  if (itemCount > 0) {
    const allRegions = [cellsAroundViewport, ...(additionalRegions ?? [])];
    for (const region of allRegions) {
      renderMask.addCells(region);
    }

    // The initially rendered cells are retained as part of the
    // "scroll-to-top" optimization
    if (props.initialScrollIndex == null || props.initialScrollIndex <= 0) {
      const initialRegion = VirtualizedList._initialRenderRegion(props);
      renderMask.addCells(initialRegion);
    }

    // The layout coordinates of sticker headers may be off-screen while the
    // actual header is on-screen. Keep the most recent before the viewport
    // rendered, even if its layout coordinates are not in viewport.
    const stickyIndicesSet = new Set(props.stickyHeaderIndices);
    VirtualizedList._ensureClosestStickyHeader(
      props,
      stickyIndicesSet,
      renderMask,
      cellsAroundViewport.first,
    );
  }

  return renderMask;
}
```

invariant를 통해 cellsAroundViewport가 유효한 값인지 확인하는군요. 이 함수는 scrollToIndex 할 때 에러로 자주 보셨을 거 같네요.
이런 형태의 코드로 에러를 쉽게 파악 할 수 있는 거 같군요. 나중에 한번 써봐야겠다는 생각을 하고 넘어갑니다.

CellRenderMask를 통해서 renderMask라는 걸 인스턴스를 만드는군요. CellRenderMask는 아래에서 자세히 살펴볼 예정인데요.
그 전에 Mask는 무엇을 의미하는 것인지 생각해보고 넘어가야 좋을 거 같습니다.
마스크를 쓰게 되면 보이는 부분과 안 보이는 부분이 아주 명확하게 나뉘어지지요. Mask를 통해서 우리가 보여줄 부분과 안 보여줄 부분을 결정 할 수 있을 거 같네요. 
이러한 마스크의 특징을 이용한다면 마스크에 의해 안 가려진 부분만 렌더링하게 된다면 성능이 향상될 거 같습니다.

![robbery-nb.png](/assets/img/robbery-nb.png)

강도 엔비 사진을 예시로 들면, 가면에 의해 가려져서 눈 부분만 렌더링하면 되기 때문에 다른 부분에 대해서는 렌더링에 대한 부담을 덜 수 있겠습니다.

## CellRenderMask

[CellRenderMask 코드](https://github.com/necolas/react-native-web/blob/master/packages/react-native-web/src/vendor/react-native/VirtualizedList/CellRenderMask.js)를 살펴보겠습니다.

```typescript
export type CellRegion = {
  first: number,
  last: number,
  isSpacer: boolean,
};

export class CellRenderMask {
  _numCells: number;
  _regions: Array<CellRegion>;

  constructor(numCells: number) {
    invariant(
      numCells >= 0,
      'CellRenderMask must contain a non-negative number os cells',
    );

    this._numCells = numCells;

    if (numCells === 0) {
      this._regions = [];
    } else {
      this._regions = [
        {
          first: 0,
          last: numCells - 1,
          isSpacer: true,
        },
      ];
    }
  }

  enumerateRegions(): $ReadOnlyArray<CellRegion> {
    return this._regions;
  }

  addCells(cells: {first: number, last: number}): void {
    ...
  }

  numCells(): number {
    return this._numCells;
  }

  equals(other: CellRenderMask): boolean {
    return (
      this._numCells === other._numCells &&
      this._regions.length === other._regions.length &&
      this._regions.every(
        (region, i) =>
          region.first === other._regions[i].first &&
          region.last === other._regions[i].last &&
          region.isSpacer === other._regions[i].isSpacer,
      )
    );
  }

  _findRegion(cellIdx: number): [CellRegion, number] {
    let firstIdx = 0;
    let lastIdx = this._regions.length - 1;

    while (firstIdx <= lastIdx) {
      const middleIdx = Math.floor((firstIdx + lastIdx) / 2);
      const middleRegion = this._regions[middleIdx];

      if (cellIdx >= middleRegion.first && cellIdx <= middleRegion.last) {
        return [middleRegion, middleIdx];
      } else if (cellIdx < middleRegion.first) {
        lastIdx = middleIdx - 1;
      } else if (cellIdx > middleRegion.last) {
        firstIdx = middleIdx + 1;
      }
    }

    invariant(false, `A region was not found containing cellIdx ${cellIdx}`);
  }
}
```

생성자 내부 코드를 살펴보면, 마스크 역할을 할 Cell들을 만들 거 같습니다. 우선 data prop 만큼의 범위를 가진 마스크를 만드는군요.
isSpacer 필드도 보이는군요. 빈공간을 위한 것이 맞다는 게 점점 확실해지는군요. 
최적화 렌더링을 위하여 초기에 Cell을 취급하는 방식이 재미 있습니다. 렌더링할 Cell을 모두 빈공간으로 취급한 상태로 인스턴스를 생성하는군요. 

이어서 addCells 메소드를 보겠습니다. 여기서 부터는 실질적으로 렌더링 할 부분에 대한 처리가 이루어지는 것으로 보입니다.

```typescript
addCells(cells: {first: number, last: number}): void {
    invariant(
      cells.first >= 0 &&
        cells.first < this._numCells &&
        cells.last >= -1 &&
        cells.last < this._numCells &&
        cells.last >= cells.first - 1,
      'CellRenderMask.addCells called with invalid cell range',
    );

    // VirtualizedList uses inclusive ranges, where zero-count states are
    // possible. E.g. [0, -1] for no cells, starting at 0.
    if (cells.last < cells.first) {
      return;
    }

    const [firstIntersect, firstIntersectIdx] = this._findRegion(cells.first);
    const [lastIntersect, lastIntersectIdx] = this._findRegion(cells.last);

    // Fast-path if the cells to add are already all present in the mask. We
    // will otherwise need to do some mutation.
    if (firstIntersectIdx === lastIntersectIdx && !firstIntersect.isSpacer) {
      return;
    }

    // We need to replace the existing covered regions with 1-3 new regions
    // depending whether we need to split spacers out of overlapping regions.
    const newLeadRegion: Array<CellRegion> = [];
    const newTailRegion: Array<CellRegion> = [];
    const newMainRegion: CellRegion = {
      ...cells,
      isSpacer: false,
    };

    if (firstIntersect.first < newMainRegion.first) {
      if (firstIntersect.isSpacer) {
        newLeadRegion.push({
          first: firstIntersect.first,
          last: newMainRegion.first - 1,
          isSpacer: true,
        });
      } else {
        newMainRegion.first = firstIntersect.first;
      }
    }

    if (lastIntersect.last > newMainRegion.last) {
      if (lastIntersect.isSpacer) {
        newTailRegion.push({
          first: newMainRegion.last + 1,
          last: lastIntersect.last,
          isSpacer: true,
        });
      } else {
        newMainRegion.last = lastIntersect.last;
      }
    }

    const replacementRegions: Array<CellRegion> = [
      ...newLeadRegion,
      newMainRegion,
      ...newTailRegion,
    ];
    const numRegionsToDelete = lastIntersectIdx - firstIntersectIdx + 1;
    this._regions.splice(
      firstIntersectIdx,
      numRegionsToDelete,
      ...replacementRegions,
    );
  }
```

추가 된 Cell들 범위에서 첫번째와 마지막 cell의 번호를 구하네요.  그 후에는 Head, Tail, Main으로 Region을 나누기 위한 코드가 보입니다.
MainRegion이 실질적으로 렌더링 할 Sell들이라서 isSpacer를 false로 설정하는군요.

```typescript
const newLeadRegion: Array<CellRegion> = [];
const newTailRegion: Array<CellRegion> = [];
const newMainRegion: CellRegion = {
  ...cells,
  isSpacer: false,
};
```

이후 부터는 허허~ Mask 의해 안 가려진 렌더링 할 부분에 대한 처리가 쭈욱 나옵니다.
메소드에 입력 된 cells에 의해서 마스크가 의해 안 가려진 부분이 결정되고 초기에 빈영역으로 지정했던 부분을 splice 함수로 덮어버리는군요.

```typescript
this._regions.splice(
  firstIntersectIdx,
  numRegionsToDelete,
  ...replacementRegions,
);
```

다시 VirtualizedList로 돌아가서 renderMask를 만드는 부분을 살펴보겠습니다.

```typescript
    //...

    const renderMask = new CellRenderMask(itemCount);

    if (itemCount > 0) {
      const allRegions = [cellsAroundViewport, ...(additionalRegions ?? [])];
      for (const region of allRegions) {
        renderMask.addCells(region);
      }

      // The initially rendered cells are retained as part of the
      // "scroll-to-top" optimization
      if (props.initialScrollIndex == null || props.initialScrollIndex <= 0) {
        const initialRegion = VirtualizedList._initialRenderRegion(props);
        renderMask.addCells(initialRegion);
      }

      // The layout coordinates of sticker headers may be off-screen while the
      // actual header is on-screen. Keep the most recent before the viewport
      // rendered, even if its layout coordinates are not in viewport.
      const stickyIndicesSet = new Set(props.stickyHeaderIndices);
      VirtualizedList._ensureClosestStickyHeader(
        props,
        stickyIndicesSet,
        renderMask,
        cellsAroundViewport.first,
      );
    }

    return renderMask;
}
```

cellsAroundViewport는 렌더링 초기에는 렌더링하기로 결정된 Cell의 번호를 가지고 있다는 것 기억나십니까? ( 저는 까먹어서 다시 보았습니다. )
initialScrollIndex, initialNumToRenderOrDefault prop과 initialRenderRegion 내부 메소드에 의해서 렌더링하기로 결정한 Cell들의 범위이지요.

```typescript
renderMask.addCells(region);
```

renderMask.addCells가 실행되면 이제 초기에 렌더할 cell들의 범위가 정해집니다.
아무리 data prop이 크더라도 렌더링 할 적당한 범위가 정해졌으므로 무시성 렌더링과 비교하면 Big O 차이가 명확해지겠습니다.

> 최적화 렌더링이 있는 컴포넌트와 없는 컴포넌트의 초기 렌더링 비용 Big O로 만들어보자.
> 
> 최적화 렌더링이 있는 컴포넌트는 렌더깅하기로 약속했던 상수 크기이므로 만약 10개만 렌더링하리고 약속했다면 O(1)이 되겠지요.
> 
> 최적화 렌더링이 없는 컴포넌트의 비용은 데이터 크기에 비례하므로 O(n)이 되겠지요.
> O(10) vs O(n)

## “react-native-web 파헤치기 2-2”을 마치며

대략적으로 VirtualizedList에서 최적화 렌더링 할 수 있게 준비하는 과정을 알게되었습니다. 흥미로군요!
아마 네번째 게시물에서 감동의 눈물을 흘리는 시간이 될 거 같군요.

react-native-web의 VirtualizedList를 살펴보면서 특정 역할을 잘 수행하는 클래스의 정의 구분이 아주 인상적입니다.
FillRateHelper, Batchinator, CellRenderMask 등이 정의 된 것을 보니, 
이러한 클래스를 정의하여 역할을 분리시키는 것이 객체지향에서 이야기하는 어떠한 객체를 만드는 과정의 결과물이라는 것을 느낄 수 있었습니다.
프론트 개발자가 React로  단순히 UI 컴포넌트만 만들다보면 이러한 클래스들을 만들 기회가 많이 없는 거 같습니다.
객체의 역할을 나누고 정의하기 위한 설계를 해볼 기회가 많이 없는 신입 개발자라면 더더욱 오픈소스를 많이 보고 이해하고 따라하는 것이 중요하다는 것을 느꼈습니다.

최근에 "그림 멘토 버트 도드슨의 드로잉 수업" 책을 읽었습니다. 책에는 모사에 대한 이야기가 나왔습니다.
그림에서 "모사"는 그림을 그리는 사람이 다른 사람의 그림을 최대한 똑같이 그려보는 것을 말합니다.
모사 통해 다른 사람의 그림을 그리면서 그림을 그리는 방법을 배울 수 있습니다. 저자는 모사를 많이 하면서 처음에는 뭔가 부끄러움을 느꼈다고 합니다.
자신이 그림을 잘 그린 것이 아니라 다른 사람의 그림을 그린 것이라는 생각이 들었기 때문이라고 합니다.
하지만 모사를 지속하다보니, 그림을 그리는 방법을 많이 알게 되었고 자신만의 그림 스타일이 생겼다고 합니다.
그 후 부터는 모사를 통해서 다른 사람의 그림을 그리는 것이 "자신의 그림을 그리는 방법"을 배우는 것이라고 생각하게 되었다고 합니다.

저 또한 다른 사람의 코드를 보면서 코드를 모사하고 어떠한 것을 만들었을 때, 부끄러운 감정을 느꼈던 것이 기억납니다.
그러다가 몇 년이 지난 후에야 모사를 통해서 다른 사람의 코드를 보고 이해하고 따라하면서 어떠한 것을 만들었을 때, 좋은 코드를 배울 수 있다는 것을 느끼게 되었습니다.

모사는 아주 의미있는 시간이라는 것을 한번 더 강조하여 마무리하겠습니다.

[docs react-native-web]: https://necolas.github.io/react-native-web/
[github react-native-web]: https://github.com/necolas/react-native-web
