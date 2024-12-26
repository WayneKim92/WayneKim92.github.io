---
layout: post
title:  "React Native Web 파헤치기 1"
date:   2024-12-16 23:20:00 +0900
categories: react-native react-native-web
---

# 리액트 네이티브 웹 ?

React Native 개발자라면 React Native Web에 대해 들어본 적이 있을 것입니다.
React Native Web은 [React Native Core 컴포넌트](https://reactnative.dev/docs/0.72/components-and-apis)와 [Core API](https://reactnative.dev/docs/0.72/usewindowdimensions)를 웹에서도 사용할 수 있게 해주는 라이브러리입니다.

# 어떻게 ??

사실상 React Native 코드는 인터페이스에 불과합니다. RN의 인터페이스는 최종적으로 네이티브 코드로 입장하는 관문이지요.

![react-native-last-end-point.png](/assets/img/react-native-last-end-point.png)

React Native Web도 마찬가지로 React Native 코드를 React DOM 코드로 입장시키는 관문 입니다.
우리가 만든 React Native 코드는 `import { ... } from 'react-native'`로 시작합니다. 이 코드의 끝에 있는 목적지 모듈의 이름(`react-native`)를 `react-native-web`으로 수작업으로 바꾸면 대부분의 react-native 코드를 재활용할 수 있습니다.

다음 플로우를 따르면, 어떻게 수작업으로 react-native 코드를 react-native-web 코드로 바꿀 수 있는지 알 수 있습니다.
1. react-native 프로젝트에서 react-native 컴포넌트를 이용하여 간단하게 HelloWorld를 출력하는 컴포넌트를 만들어보세요.
2. react 프로젝트에 `react-native-web` 모듈을 설치 합니다. ( `yarn add react-native-web` 또는 `npm install react-native-web` )
3. 아까 전에 만들었던 HelloWorld 컴포넌트를 react 프로젝트에 복사 붙여넣기 합니다.
4. HelloWorld 컴포넌트에서 `import { ... } from 'react-native'`를 `import { ... } from 'react-native-web'`로 바꿉니다.
5. react 프로젝트를 실행합니다. Tada~! HelloWorld 컴포넌트가 잘 작동하는 것을 볼 수 있습니다.

하지만 이런 단순하고 반복적인 일은 사람이 할 일이 아니지요. 그래서 react-native-web 모듈에는 이 작업을 수행하는 [babel-plugin](https://github.com/necolas/react-native-web/tree/master/packages/babel-plugin-react-native-web)이 포함되어 있습니다. 
[바벨 플러그인의 코드](https://github.com/necolas/react-native-web/blob/master/packages/babel-plugin-react-native-web/src/index.js)를 한번 살펴보지요.
이름 부터 역할이 아주 확실하지요. `'Rewrite react-native to react-native-web'` 그렇습니다. 이 플러그인은 위에서 했던 수작업 `from 'react-native'` 의 `from 'react-native-web'` 코드로 바꿔주는 처리를 합니다.

FYI. Babel은 자바스크립트/타입스크립트 코드를 읽고 변환하여 AST(Abstract Syntax Tree)로 만듭니다. 개발자는 바벨이 만들어준 AST를 기반으로 변환 규칙을 정의하여 새로운 코드를 만들 수 있지요.

[AST Explorer](https://astexplorer.net/) 사이트에서 아래 코드의 추상 문법 트리를 자세히 살펴보겠습니다. ( 사이트에 접속하여 아래 코드를 복사-붙여넣이 하세요~)

```tsx
import { Text as RNText } from 'react-native'

function HelloWorld(){
  return (
    <Text>Hello World</Text>
  )
}
```

Program.body > ImportDeclaration.sorce.value 까지 이동해보면 값이 'react-native'로 나오는 것을 볼 수 있습니다.
react-native-web babel-plugin 코드로 돌아가서 코드를 보면 import하는 모듈이 ReactNative 모듈이면 작성한 바벨 플러그인 코드에 의해서 react-native-web/dist/exports/Text/index.js로 변환되는 것을 볼 수 있습니다.
react-native-web github 레포에는 dist 폴더가 없으므로 npm의 Code 탭에서 위 경로로 이동하여 해당 컴포넌트가 있는 것을 확인 수 있지요.

이로써 react-native-web은 react-native 코드를 react-native-web 코드로 입장시키는 관문이라는 것을 알 수 있습니다.

이 바벨 플러그인을 조금 더 살펴보지요. getDistLocation 함수에서 `react-native-web/dist/${...}` 형식으로 최종 목적지를 문자열을 반환해주는데요. 
이 목적지에 있는 컴포넌트를 1~2개 정도 보겠습니다.

## 먼저 View 컴포넌트 부터

모듈 설치도 귀찮으니, [npm의 Code 탭](https://www.npmjs.com/package/react-native-web?activeTab=code)에서 살펴보겠습니다.
/react-native-web/dist/exports/View/index.js 파일을 먼저 살펴보겠습니다. react 기반 웹 프로젝트에서 코드를 build 하였을 때 코드가 보이는 군요.
어셈블러 장인만 볼 수 있는 코드 수준으로 변환되어 있지는 않지만 그래도 읽기가 불편하니 /react-native-web/src/exports/View/index.js 로 이동하여 코드를 살펴보겠습니다.

![react-native-web-view-1.png](/assets/img/react-native-web-view-1.png)
![react-native-web-view-2.png](/assets/img/react-native-web-view-2.png)
![react-native-web-view-3.png](/assets/img/react-native-web-view-3.png)

그렇습니다. 그저 React 코드이지요. JSX로 작성되어 있지 않을 뿐, 그저 React 코드이지요.

갑분사담. Expo 프로젝트의 package.json > scripts에는 기본적으로 웹 플랫폼에서 RN 코드를 실행시킬 수 있는 스크립트를 제공해주시요. Expo를 이용하여 아주 꽤큰 애플리케이션을 만든 후에 웹도 서비스하게 되는 경우가 종종 있는 거 같습니다.
이때 웹 프로젝트를 실행시켜보면 초기 렌더링이 많이 느린 것을 볼 수 있는데요. 번들 사이즈가 크다보니 성능이 좋지않지요. 이런 케이스를 본 개발자들이 꽤 있는 거 같습니다. 제 주변에만 3~4명이 react-native-web 성능이 안 좋다고 이야기하더군요.
이 이야기는 제 머릿속에 물음표를 떠오르게 하는데요. 음... 어떤 기준을 가지고 있는 것이냐에 따라서 달라지기 때문에 이야기가 끝나지 않는 것 같습니다. 그래서 이 이야기는 여기서 끝내겠습니다.

벤치마킹 결과가 궁금하시다면, [다음 사이트](https://necolas.github.io/react-native-web/benchmarks/)에서 보실 수 있습니다.

## 다음은 TextInput 컴포넌트

npm Code 탭에서 /react-native-web/src/exports/TextInput/index.js 파일로 이동해봅시다!
React Native의 TextInput과 동일하게 웹에서 동작하기 위한 많은 코드들이 보이는군요.

![react-native-web-textinput-1.png](/assets/img/react-native-web-textinput-1.png)

여기서 한번 생각해보면 좋을 부분은 React Native 공식 문서에 있는 "Learn once, write anywhere" 슬로건 입니다.
"한번 배워서 어디서든 코딩 할 수 있다!" 이 슬로건을 위한 노력이 바로 React Native Web 입니다.

[macOs](https://github.com/microsoft/react-native-macos), [windows](react-native-windows) 등등 다양한 노력들이 있지요.

## "react-native-web 파헤치기 - 1"을 마치며

React Native 개발자라면 슬로건을 한번 더 생각해보면 좋을 거 같습니다. "한번 배워서 어디서든 코딩 할 수 있다!" 이 슬로건이 얼마나 중요한지요.
우리가 만든 코드가 어디서든 잘 동작 할 수 있다는 것이 얼마나 큰 장점인지요. RN 개발자라면 클린코드를 작성하면서 위 장점을 최대한 누릴 수 있는 스킬이 필요한 거 같습니다.

[docs react-native-web]: https://necolas.github.io/react-native-web/
[github react-native-web]: https://github.com/necolas/react-native-web
