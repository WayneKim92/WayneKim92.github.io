---
layout: post
title:  "Expo 훑어보기 - 2"
date:   2025-2-8 22:55:00 +0900
categories: expo
---

# Expo Tutorial 훑어보기 - 2

## alert은 또 왜 Expo에서 사용할 수 있는 건데?

global 객체를 건드리는 것은 일반적이지 않은데... global.alert에 알림을 위한 함수가 등록되어 있다.
Expo 레포를 살펴보아도 해당 코드를 못 찾았다 ㄷㄷ
결국은 React-Native에 있는 Alert을 이용했을 것으로 추측된다.

## expo-image-picker

5년 전에 Expo가 네이티브 모듈을 커스텀해서 못 쓴다는 욕을 먹고 있을 때에도 Expo SDK는 품질이 좋다는 이야기를 많이들었다.
Expo-image-picker를 쏘보면 꽤 쓸만하다! ( 요즘에는 GPT로 있어서, 쓰는 법 알려주는 건 무의미하니 패스)

## React-Native Modal

Modal 많이 쓸만해졌네! animationType 지정하여서 원하는 형태로 쓰면 좋을 듯 하다

## react-native-gesture-handler

필요에 따라서 제스처 핸들러를 쓰면 좋을 듯 하다. 사용법은 GPT에게~

## 스크린샷 찍기!

오... 처음 보는 패키지가 있어서, 이건 보고 넘어가야겠다.

`react-native-view-shot`로 View에 있는 요소를 찍고 `expo-media-library`로 저장할 수 있다니?!
자세한 사용법은 [문서](https://docs.expo.dev/tutorial/screenshot/)를 보자~

### 그런데 웹은 미지원

`dom-to-image`을 사용하여, 웹에서는 스크린샷을 만들어서 처리하구나~
특정 플랫폼 종속 코드에는 Platform 객체를 이용해보자~ 

## 상태바, 스플레시 스크린, 앱 아이콘

상태바 변경은 `expo-status-bar`

### expo-status-bar 는 왜 필요할까? RN 내장 컴포넌트와 API로 충분하지 않나?

RN의 StatusBar와 동일한 인터페이스를 가지지만 Expo 환경에 적합하게 동작할 수 있게 [default 값](https://github.com/expo/expo/tree/main/packages/expo-status-bar)이 다르다고함.

## 앱 아이콘

앱 아이콘 변경은 `assets/images/icon.png`를 변경

## 스플래스 스크린

스플래쉬 스크린 이미지 변경은 `expo-splash-screen` 이미 추가 되어 있으니, app.json의 플러그인 설정 변경
명시적으로 splash screen을 숨기라고 하기 전까지 계속 보여준다고함.
필수 API 호출, 글꼴 로드 등 작업을 수행하고 스플래쉬 스크린을 숨김 처리하면 유용하다고함

# Expo의 추천 리소스
https://docs.expo.dev/tutorial/follow-up/