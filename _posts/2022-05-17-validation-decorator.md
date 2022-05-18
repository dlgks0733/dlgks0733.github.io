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

저는 대규모 프로젝트에 투입했을 때 공통파트에서 개발한 Validation Decorator 를 사용하여 쉽게 유효성 검증 기능을 만든 좋은 경험이 있습니다.

이 경험을 되살려 값 입력 시 실시간으로 유효성 검증되는 Validation Property Decorator 를 Toy Project 에 적용했습니다.

<br />
<br />
<br />
### Decorator

Decorator 는 클래스 선언, 메서드, 접근자, 프로퍼티 또는 매개변수에 첨부할 수 있는 특수한 종류의 선언으로 Java Annotation 과 같은 `@express` 형식을 가집니다.

만약 Validation 기능을 Property Decorator 개발하여 사용할 경우 아래와 같이 쓰입니다.

```typescript
    @Size(6, 100)
    adId = ''

    @Size(6, 100)
    adPwd = ''
```

<br />
### reflect-metadata
JavaScript 에 내장되어 있는 Reflect 에 metadata 기능을 확장한 패키지 입니다.
이 패키지를 사용하면 사용자 정의 metadata 를 추가할 수 있습니다.

<br />
<br />
<br />
### 구현

구현하기 전에 reflect-metadata, Decorator 를 사용하기 위해 패키지 설치와 tsconfig.json 을 수정해줍니다.

**_Command Line:_**

```typescript
npm install reflect-metadata
```

**_tsconfig.json:_**

```typescript
{
    "compilerOptions": {
        "experimentalDecorators": true, // Decorator 에 대한 실험적 지원 활성화
        "emitDecoratorMetadata": true // Decorator 가 있는 선언에 대해 특정 타입의 metadata 를 내보내는 실험적 지원 활성화
    }
}
```

<br />
<br />
먼저 자주 쓰이는 유효성 검증을 Decorator 로 지정하여 method 로 만듭니다. 저는 Size 라는 method 를 만들어 문자열의 길이를 제한하는 Decorator 를 만들었습니다.
<br />
[vue-class-component 의 createDecorator](https://class-component.vuejs.org/guide/custom-decorators.html) 는 사용자 정의 Decorator 를 생성하기 위한 도움 역할을 해줍니다.

```typescript
// validation.ts
import { createDecorator, VueDecorator } from 'vue-class-component'

/**
 * @param min 최소 길이
 * @param max 최대 길이
 * @param errMsg 에러 메세지 (default: 최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.)
 */
export function Size(min: number, max: number, errMsg = `최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.`): VueDecorator {
  return createDecorator((options, key) => {
        console.log('size decorator!')
    })
}
```

<br />
<br />
위와 같은 코드를 작성하면 아래와 같이 Vue Component 에서 Size 를 import 하여 사용할 수 있습니다.
```typescript
// Login.vue
<script lang="ts">
import { Component, Vue } from 'vue-property-decorator'
import { Size } from '@/common/validation'

@Component
export default class Login extends Vue {
    @Size(6, 100)
    adId = ''

    @Size(6, 100)
    adPwd = ''
}
</script>
```

<br />
<br />
이제 Decorator 함수 안을 구현해줍니다. 정규식과 에러 메세지가 담겨있는 metadata 를 정의합니다.

```typescript
// validation.ts
import { createDecorator, VueDecorator } from 'vue-class-component'

const validationsKey = Symbol('validationsKey')

/**
 * @param min 최소 길이
 * @param max 최대 길이
 * @param errMsg 에러 메세지 (default: 최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.)
 */
export function Size(min: number, max: number, errMsg = `최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.`): VueDecorator {
  return createDecorator((options, key) => {
        let validtions = Reflect.getMetadata(validationsKey, options) // 객체 또는 속성의 프로토타입 체인에 있는 메타데이터 키의 메타데이터 값을 가져옵니다.
        if (!validtions) {
            validtions = {}
        }
        validtions[key] = {
            regExp: new RegExp(`^\\s*(?:\\S\\s*){${min},${max}}$`),
            errMsg: errMsg
        }
        Reflect.defineMetadata(validationsKey, validtions, options) // options 에 validations 를 validationsKey 로 메타데이터 정의
        console.log(Reflect.getMetadata(validationsKey, options))
    })
}
```
metadata 를 console 로 확인하면 정규식과 에러메세지가 담겨져있는 것을 확인할 수 있습니다.
{% include aligner.html images="pexels/metadata1.PNG" column=1 %}

<br />
<br />
metadata 에 담겨져 있는 정규식과 에러메세지를 이용하여 Validator Method 를 options 의 methods 에 추가해줍니다.
<br />
Validator Method 명칭은 `${propertyKey}Validator` 로 만들었습니다.
유효성 검증이 실패하면 `${propertyKey}Rules` 배열에 접근하여 값을 push 합니다.

```typescript
// validation.ts
import { createDecorator, VueDecorator } from 'vue-class-component'

const validationsKey = Symbol('validationsKey')

/**
 * @param min 최소 길이
 * @param max 최대 길이
 * @param errMsg 에러 메세지 (default: 최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.)
 */
export function Size(min: number, max: number, errMsg = `최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.`): VueDecorator {
  return createDecorator((options, key) => {
        let validtions = Reflect.getMetadata(validationsKey, options) // 객체 또는 속성의 프로토타입 체인에 있는 메타데이터 키의 메타데이터 값을 가져옵니다.
        if (!validtions) {
            validtions = {}
        }
        validtions[key] = {
            regExp: new RegExp(`^\\s*(?:\\S\\s*){${min},${max}}$`),
            errMsg: errMsg
        }
        Reflect.defineMetadata(validationsKey, validtions, options) // options 에 validations 를 validationsKey 로 메타데이터 정의
        createValidator(options)
    })
}

// methods 객체에 validator 등록
function createValidator(options: any) {
    const metadata = Reflect.getMetadata(validationsKey, options)
    const keys = Object.keys(metadata)
    const currentKey = keys[keys.length - 1]
    options.methods[`${currentKey}Validator`] = function validator(value: any) {
        const validtions = metadata[currentKey]
        if (!validtions.regExp.test(value) && value) {
            this.$data[`${currentKey}Rules`].push(false || validtions.errMsg)
        } else {
            this.$data[`${currentKey}Rules`] = []
        }
    }
    console.log(options)
}
```
options 를 확인한 결과 methods 에 등록된 걸 확인할 수 있습니다.
{% include aligner.html images="pexels/metadata2.PNG" column=1 %}

<br />
<br />
이제 마지막으로 실시간 값 감지를 위한 watch 를 options 에 추가합니다.

```typescript
// validation.ts
import { createDecorator, VueDecorator } from 'vue-class-component'

const validationsKey = Symbol('validationsKey')

/**
 * @param min 최소 길이
 * @param max 최대 길이
 * @param errMsg 에러 메세지 (default: 최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.)
 */
export function Size(min: number, max: number, errMsg = `최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.`): VueDecorator {
  return createDecorator((options, key) => {
        let validtions = Reflect.getMetadata(validationsKey, options) // 객체 또는 속성의 프로토타입 체인에 있는 메타데이터 키의 메타데이터 값을 가져옵니다.
        if (!validtions) {
            validtions = {}
        }
        validtions[key] = {
            regExp: new RegExp(`^\\s*(?:\\S\\s*){${min},${max}}$`),
            errMsg: errMsg
        }
        Reflect.defineMetadata(validationsKey, validtions, options) // options 에 validations 를 validationsKey 로 메타데이터 정의
        createValidator(options)
    })
}

// methods 객체에 validator 등록
function createValidator(options: any) {
    const metadata = Reflect.getMetadata(validationsKey, options)
    const keys = Object.keys(metadata)
    const currentKey = keys[keys.length - 1]
    options.methods[`${currentKey}Validator`] = function validator(value: any) {
        const validtions = metadata[currentKey]
        if (!validtions.regExp.test(value) && value) {
            this.$data[`${currentKey}Rules`].push(false || validtions.errMsg)
        } else {
            this.$data[`${currentKey}Rules`] = []
        }
    }
    createWatch(options)
}

// watch 등록
function createWatch(options: any) {
    const metadata = Reflect.getMetadata(validationsKey, options)
    const keys = Object.keys(metadata)
    const currentKey = keys[keys.length - 1]
    if (!options.watch) {
        options.watch = {}
    }
    options.watch[currentKey] = [
        {
            handler: `${currentKey}Validator`,
            deep: false,
            immediate: false,
            user: true
        }
    ]
    console.log(options)
}
```
options 에 watch 가 추가된 것을 확인할 수 있습니다.
그리고 실시간으로 값 유효성을 체크하는 것을 확인할 수 있습니다.
{% include aligner.html images="pexels/metadata3.PNG,pexels/metadata4.gif" column=2 %}

<br />
<br />
여러 가지 Validation Decorator 를 위해 createValidationDecorator 함수를 만들었습니다.

```typescript
// validation.ts
import { createDecorator, VueDecorator } from 'vue-class-component'
import 'reflect-metadata'

const validationsKey = Symbol('validationsKey')

// decorator 생성
function createValidationDecorator(regExp: RegExp, errMsg: string): VueDecorator {
    return createDecorator((options, key) => {
        let validtions = Reflect.getMetadata(validationsKey, options)
        if (!validtions) {
            validtions = {}
        }
        validtions[key] = {
            regExp: regExp,
            errMsg: errMsg
        }
        Reflect.defineMetadata(validationsKey, validtions, options)
        createValidator(options)
    })
}

// methods 객체에 validator 등록
function createValidator(options: any) {
    const metadata = Reflect.getMetadata(validationsKey, options)
    const keys = Object.keys(metadata)
    const currentKey = keys[keys.length - 1]
    options.methods[`${currentKey}Validator`] = function validator(value: any) {
        const validtions = metadata[currentKey]
        if (!validtions.regExp.test(value) && value) {
            this.$data[`${currentKey}Rules`].push(false || validtions.errMsg)
        } else {
            this.$data[`${currentKey}Rules`] = []
        }
    }
    createWatch(options)
}

// watch 등록
function createWatch(options: any) {
    const metadata = Reflect.getMetadata(validationsKey, options)
    const keys = Object.keys(metadata)
    const currentKey = keys[keys.length - 1]
    if (!options.watch) {
        options.watch = {}
    }
    options.watch[currentKey] = [
        {
            handler: `${currentKey}Validator`,
            deep: false,
            immediate: false,
            user: true
        }
    ]
}

/**
 * @param min 최소 길이
 * @param max 최대 길이
 * @param errMsg 에러 메세지 (default: 최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.)
 */
export function Size(min: number, max: number, errMsg = `최소 ${min}자 이상 최대 ${max}자 이하 문자를 입력해주세요.`): VueDecorator {
    return createValidationDecorator(new RegExp(`^\\s*(?:\\S\\s*){${min},${max}}$`), errMsg)
}

/**
 * @param errMsg 에러 메세지 (default: '이메일 형식에 맞지 않습니다.')
 */
export function Email(errMsg = '이메일 형식에 맞지 않습니다.'): VueDecorator {
    return createValidationDecorator(new RegExp('^[0-9a-zA-Z]([-_\\.]?[0-9a-zA-Z])*@[0-9a-zA-Z]([-_\\.]?[0-9a-zA-Z])*\\.[a-zA-Z]{2,3}$'), errMsg)
}
```
<br />
<br />
<br />
### 향후 계획
Property Decorator 를 통해 유효성 검증을 만들었지만 유효성 검증할 Property 가 많아질수록 그에 따른 method, watch 가 많아져 그 부분을 보완할 점을 생각해야 할 것 같습니다.
<br />
그리고 이러한 `@CusmtomValidate(regExp: RegExp, errMsg: string)` 직접 정의한 정규식으로 검증할 수 있는 Decorator 를 제공하여 자유도를 높여야 할 것 같습니다.

<br />
<br />
<br />
**참고**
<br />
[https://typescript-kr.github.io/pages/decorators.html](https://typescript-kr.github.io/pages/decorators.html)
<br />
[https://www.npmjs.com/package/reflect-metadata](https://www.npmjs.com/package/reflect-metadata)
<br />
[https://medium.com/jspoint/introduction-to-reflect-metadata-package-and-its-ecmascript-proposal-8798405d7d88](https://medium.com/jspoint/introduction-to-reflect-metadata-package-and-its-ecmascript-proposal-8798405d7d88)
