---
layout: post
title:  "Expo 훑어보기 - 1"
date:   2025-2-7 22:55:00 +0900
categories: expo
---

# Expo

5년 전에 RN 개발자에게 실망감을 주었던 Expo가 달라졌어요?

Expo는 프레임워크가 되었다. 파일 기반 라우팅, 네이티브 표준 라이브러리 등을 제공하는 정도로 발전했다. (5년 전에는 둘 다 불가능 했다.)
거기다 EAS(Epo Application Services)를 통해 Expo를 이용하는 개발자에게 빌드 부터 배포에 들였던 시간을 줄여준다.

## Expo Tutorial

프로젝트 생성 명령어

```shell
npx create-expo-app@latest
```

보일러 플레이트 코드 제거하는 명령어 ( DX 좋고요! )

```shell
npm run reset-project
```

### reset-project 까보자~ DX를 위하여~

[코드](https://github.com/expo/expo/blob/1de6bb5632864184d8ea89d4650f70bb42dee4ac/templates/expo-template-default/scripts/reset-project.js#L6)를 보면 생각보다 간단하다.

app-example 폴더를 만들고 기존 예제 코드를 옮기고 app 폴더를 만들고 꼭 필요한 파일만 생성하면 끝!

이런 스크립트가 "어떤 폴더 지우고 어떤 파일을 복사-붙여넣기" 하세요 같은 문서 보다 백만배 더 효과적이다.

### expo router

expo router는 파일 기반 라우팅을 제공하는 프레임워크이다. ( 요즘에는 ChatGPT로 질문하면, 프레임워크와 라이브러리의 차이를 너무나도 쉽게 할 수 있어서 세상 참 좋아졌다. ) 

프레임워크이므로 몇가지 사용 규칙을 살펴보자.
* app 폴더: 라우터와 레이아웃만 포함하는 폴더, 이 폴더에 포함된 파일은 앱에서 화면이고 웹에서는 페이지가가 된다.
* app/_layout.tsx 파일: 일관성 유지를 위한 해더, 탭 바 같은 공유 UI 요소를 정의하는 곳
* 파일 이름 컨벤션
  * index 파일은 부모 폴더의 이름과 일치하는 경로를 가지므로 경로 세그먼트 추가 필요 없음
  * app/index.tsx 파일은 `/` 라우트와 매칭 된다.
* 라우트 파일은 default 값으로써 React 컴포넌트로 export 된다.
  * default로 export 되지 않은 값 expo-router에 의해 렌더링 되지 않는다.
* ios, android, web을 네비게이션 탐색 구조를 공유한다.


react-navigation 으로 부터 탈출!

### "(tabs)" 폴더

`(tabs)` 이름을 가진 폴더는 탭 네비게이션을 정의하는 곳이다. ( 잘 쓰이지 않겠지만 이중삼중으로 tabs 내비게이션을 구성할 수 있다. )
`_layout.tsx` 파일에서 탭 네비게이션을 정의하면 된다.

## expo router ?!
expo router에 어떤 발전이 있었는 지 보자.

- 웹에서 가장 뛰어난 파일 시스템 라우팅 개념을 접목하였다고 한다. ( 아주 [자화자찬](https://docs.expo.dev/router/introduction/#what-are-the-benefits-of-file-based-routing)이 ...)
- 지연 평가를 통하여 최적화도 되어있다고 한다.
- 웹 SEO를 위하여 빌드 타임에 정적 렌더링을 지원한다.

그리고 몇가지 특이사항도 보고 넘어가자.
- React Navigation과 같이 사용할 수 있지만... 당연히 새앱에서는 Expo Router만 사용하기를 권장한다.
- Wix의 React Native Navigation을 Expo Go에서 호환이 안되어서 사용하지 못 한다.
- 서버 측 렌더링은 기본적으로 지원하지 않아서 추가 설정을 해야한다. ( 소스코드를 살펴보니, metro를 이용하여 SSR을 지원하려는 흔적이 보인다. )
- 웹 번들러로 Metro web을 사용한다.

### expo router의 파일 기반 네비게이션 시스템은 어떻게 만들어져있을까?

까보고 싶어서 양이 엄청나게 많다... 단순히 하나의 라이브러리로 만들기에는 규모가 크다.
그래서인지  여러 패키지를 의존하고 있다.

expo-router를 설치할 때 같이 설치하는 의존성 패키지들
```
react-native-safe-area-context react-native-screens expo-linking expo-constants expo-status-bar
```

[package](https://github.com/expo/expo/tree/main/packages/expo-router)를 까보자...
expo-router의 [package.json](https://github.com/expo/expo/blob/main/packages/expo-router/package.json)을 살펴보니, @react-navigation에 의존하고 있는 것으로 확인되었다.
그럼... expo-router의 핵심 부분은 파일 기반 라우터를 처리하는 코드로 구성되어 있을 것으로 추측된다.

코드 파일을 읽고 가공하는 일은 주로 babel로 처리하니 [expo-router-plugins.ts](https://github.com/expo/expo/blob/main/packages/babel-preset-expo/src/expo-router-plugin.ts)를 살펴보자.
코드만 보아서는 이해가 되지 않아서, 다른 파일을 찾아보자.

결국은 react-navigation으로 구성되어 있으니, react-navigation을 사용할 때 처럼 App을 등록하는 부분이 있을 거 같아 찾아보니 이 [파일](https://github.com/expo/expo/blob/80b3d8e6f6a659806b6bc06b555e5397090a36db/packages/expo-router/src/qualified-entry.tsx#L20
)을 찾았다.

Head.Provider는 무시해도 될 거 같아보이고, ExpoRoot를 살펴보니 ctx가 어떻게 만들어지는 지만 알면, 파일 기반 네비게이션을 위한 라우터 정보를 어떻게 만드는 지 알 수 있을 것으로 추측된다.
ctx에 담겨오는 children가 어떻게 만들어지는만 보면 될것으로 추측됨!

### [_ctx.js](https://github.com/expo/expo/blob/80b3d8e6f6a659806b6bc06b555e5397090a36db/packages/expo-router/_ctx.js)

[require.context](https://webpack.js.org/guides/dependency-management/#requirecontext)는 webpack 컴파일러가 지원하는 특수 기능으로, 특정 기본 디렉토리에서 일치하는 모든 모듈을 가져올 수 있습니다.
그런데... webpack은 웹에서만 사용될 터인데... expo의 모바일 번들링 당시에는 메트로를 사용하는 것일텐데, 무언가 잘못 추측하고 있는 거 같다.
음... 중간 과정을 놓친 거 같지만 ctx를 통해서 app 폴더의 트리 구조 정보를 받아올 것으로 추측된다.

[expo의 라우터](packages/expo-router/src/rsc/router/expo-definedRouter.ts)를 정의하는 파일을 찾았으니, 이 파일을 기반으로 역으로 추적해보자.
문맥을 잃어버림 ... ( Expo Router 개발자들은 DX를 위아혀 얼마나 딥다이브 하고 있는 것인지... 이걸 개발하였던 배경지식이 있는 사람만 제대로 이해 할 수 있는 수준처럼 느껴진다. )

도움을 받을 수 있을 지 모르지만 [Help!](https://github.com/expo/expo/discussions/34744)을 요청해보자.

아쉽지만 튜토리얼의 다음 단계로 넘어가자.

