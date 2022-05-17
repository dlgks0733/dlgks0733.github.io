---
layout: post
title: Vue.js 와 TypeScript를 이용하여 Validation Decorator 만들기
author: han
excerpt_separator: <!--more-->
published: true
---

<br />
## 오늘의 기록 😄
<!--more-->
<br />
오늘은 Toy Project 에서 개발한 Validation Decorator 에 대해 기록하려고 합니다.

저는 대규모 프로젝트에 투입했을 때 공통파트에서 개발한 Validation Decorator 를 사용하여 쉽게 유효성 검증 기능을 만든 경험이 있습니다.

모든 곳에서 사용되는 유효성은 통일된 정규식과 개발자에게 어느 정도의 통제가 필요합니다.

> 자주 쓰이는 유효성 검증을 하나의 Decorator 로 만들어 개발자에게 제공함으로 유지보수성이 높아지고 편의성이 좋아지는 기대효과를 볼 수 있습니다.

<br />
<br />
### Decorator

Decorator 는 클래스 선언, 메서드, 접근자, 프로퍼티 또는 매개변수에 첨부할 수 있는 특수한 종류의 선언으로 Java Annotation 과 같은 `@express` 형식을 가집니다.

만약 Validation 기능을 Property Decorator 개발하여 사용될 경우 아래와 같이 쓰입니다.

```typescript
    @Size(6, 100)
    id = ''

    @Size(6, 100)
    pwd = ''
```

<br />
<br />
### 구현해보기

구현하기 앞서 TypeScript 에서 Decorator 를 사용하기 위해 아래와 같은 커맨드로 tsconfig.json 을 수정해줍니다.

<br />
**_Command Line:_**

```typescript
tsc --target ES5 --experimentalDecorators
```

<br />
**_tsconfig.json:_**
```typescript
{
    "compilerOptions": {
        "target": "ES5",
        "experimentalDecorators": true
    }
}
```
