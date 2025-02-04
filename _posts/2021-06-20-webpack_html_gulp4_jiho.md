---
layout: post
title: Webpack보다 좋은 Html 번들러 Gulp4 시작하기
subtitle: Webpack보다 좋은 Html 번들러 Gulp4 시작하기
author: 전지호
categories: 머신러닝
tags: [docker,kubernetes,jupyter,aws]
---


# Gulp4으로 변경되면서, 많은 부분에서 실용적인 번들러가 되었습니다. 🐙

> ES6 문법과 비동기 문법, Image Sprite를 지원하기에, Html을 작성하기에 아주 괜찮은 도구가 되었습니다.



## 시작하기

----------

글로벌에 gulp-cli를 설치하면, gulp를 사용할 수 있습니다.

```csharp
npm install -global gulp-cli

```

작업 폴더에 생성하고 들어가 gulp를 설치합니다.

```lua
npm install gulp --save-dev

```



## 필수 라이브러리 설치하기

----------

### gulp-connect

-   gulp-connect는 작업 환경을 로컬 브라우저에서 띄어 확인하는 용도로 사용합니다.

### del

-   폴더안에 내용을 삭제하는 역할을 합니다. 보통 새 빌드전에 수행합니다.

다음 명령어로 gulp-connect(로컬서버)와 del을 설치합니다.

```css
npm install --save-dev gulp-connect del

```



## gulpfile.js 파일 만들기

----------

> gulpfile은 어떤 작업 task를 수행할 건지, 각 task를 수행할건 지 명시하는 파일입니다.
>
> > 모든 gulp 동작은 이 파일을 통해 수행됩니다.

gulpfile.js을 작업폴더에 생성합니다.

gulpfile.js안에 다음과 같이 작성합니다.

```javascript
//1. 필수 라이브러리
const { src, dest, watch, series, parallel } = require('gulp');
const connect = require('gulp-connect');
const del = require('del');

//2. 작업 전 배포폴더 지우기
async function clean(cb){
    await del('dist/**', { force:true });
    cb()
};

//3. 로컬서버
function connectServer() {
    connect.server({
        root: './dist',
        livereload: true,
        port: 8001
    });
};

//4. 파일변경확인
function watchFile() {

};

//5. 작업실행
exports.default = parallel(
    series(clean, connectServer),
    watchFile
);

```

기본 동작을 확인하기 위한 모든 작업이 완료되었습니다.

gulpfile.js 파일을 확인해보면서 어떻게 코드를 작성할 지 살펴봅시다.

모든 task는 함수형으로 작성되어 있는 것을 볼 수 있습니다.

일단 1번 부분에서 필요한 라이브러리(설치한 라이브러리)를 호출합니다.

그 다음, 함수형으로 작성된 2,3,4번 task를 맨 마지막 5번 task 에서 호출하는 것을 볼 수 있습니다.

이 처럼 코드를 작성한 사람이 원하는대로 작업 순서를 변경할 수 있어서 webpack보다 훨씬 직관적인 코드 작성이 가능합니다.



### parallel 함수

-   5번에 해당하는 코드를 살펴보면  `watchFile`과  `clean`,  `connectServer`  함수는 gulp 내장 함수인  `parallel`에 의해  병렬로 처리되는 것을 알 수 있습니다.

### series 함수

-   5번에 해당하는 코드를 살펴보면  `watchFile`과  `clean`,  `connectServer`  함수는 gulp 내장 함수인  `series`에 의해  차례대로 처리되는 것을 알 수 있습니다.



## task 수행

----------

> gulp명령어를 통해 gulpfile.js안에 명시한 작업들이 수행됩니다.

```undefined
gulp

```

위 예시에서는 gulp-connect를 통한 로컬 서버 동작을 확인할 수 있습니다.

![gulp-connect 서버 동작](https://solution-userstats.s3.ap-northeast-1.amazonaws.com/techblogs/batteryho/gulp/%EB%A1%9C%EC%BB%AC%EC%84%9C%EB%B2%84%EC%88%98%ED%96%89.JPG)

gulp-connect 서버 동작



## 기본적인 Html을 작성할 수 있는 프로젝트를 완성해봅시다.

----------

> Html을 작성하기 위해 제가 자주 사용하는 라이브러리인 ejs와 sass를 사용하기로 했습니다. 다른 환경을 원한다면 설치하여 사용할 수 도 있습니다.

다음과 같이 필요한 라이브러리를 추가로 설치했습니다.

```lua
npm install --save-dev gulp-sass node-sass gulp-ejs gulp-rename

```

### gulp-sass, node-sass

-   sass를 빌드하기 위한 라이브러리입니다. webpack에 loader에 해당합니다.

### gulp-ejs

-   ejs를 빌드하기 위한 라이브러리입니다. webpack에 loader에 해당합니다.

### gulp-rename

-   파일명을 변경하기 위한 라이브러리입니다. 위 예제에서는 ejs를 빌드한 뒤, html파일로 변환합니다.

gulpfile.js는 다음과 같이 수정했습니다.

```javascript
//1. 필수 라이브러리
const { src, dest, watch, series, parallel } = require('gulp');
const connect   = require('gulp-connect');
const del   = require('del');
//라이브러리 추가
const sass = require('gulp-sass');
sass.compiler = require('node-sass');
const ejs  = require('gulp-ejs');
const rename  = require('gulp-rename');

//2. 작업 전 배포폴더 지우기
async function clean(cb){
    await del('dist/**', { force:true });
    cb()
};

//3. 작업폴더
function connectServer() {
    connect.server({
        root: './dist',
        livereload: true,
        port: 8001
    });
};

//4. 파일변경확인 - 작업추가
function watchFile() {
    watch('./src/scss/**/*.scss', sassTask)
    watch('./src/ejs/*.ejs', ejsTask)
};

//sass파일의 작업
function sassTask() {
    return src('./src/scss/**/*.scss')
    .pipe(sass()) //sass로 파싱
    .pipe(dest('./dist/css')) //css폴더로 이동
    .pipe(connect.reload());
}

//ejs파일의 작업
function ejsTask() {
    return src('./src/ejs/*.ejs')
    .pipe(ejs()) //ejs로 파싱
    .pipe(rename({ extname: '.html' })) //확장자를 html로 수정
    .pipe(dest('./dist')) //dist폴더로 이동
    .pipe(connect.reload());
}

//5. 작업실행
exports.default = parallel(
    series(clean, sassTask, ejsTask, connectServer), // sassTask, ejsTask 함수 추가
    watchFile
);

```

`sassTask`와  `ejsTask`를 함수로 작성하고 해당 task들을 series와 watchFile 내부에 작성하여,

파일 변경 시, 로컬 서버에서 새로 고침하면 변경 내용을 확인할 수 있게 했습니다.

Webpack과 달리, 원하는 작업을 함수로 작성하고 호출하는 방식으로 사용할 수 있습니다.

### src 함수

-   Task를 수행할 폴더를 지칭하는 gulp 내장 함수입니다. pipe를 호출하기 전에 수행합니다.

### pipe 함수

-   Task를 연결하는 함수 체이닝입니다. 원하는 작업을 pipe 함수 체인으로 연결합니다.

### dest 함수

-   Task를 마친 ouput이 저장될 폴더입니다. 보통 가장 마지막에 사용됩니다.



## 원하는 Task를 추가하여, 빌드 자동화를 구축해봅시다.

----------

gulp는 이전 버전부터 Webpack에서 구현이 잘 되지않는 image sprite 라이브러리가 잘 구현되어 있습니다.

image sprite, svg sprite등 원하는 작업을 추가하여, Html 번들러는 나의 업무에 맞게 활용해 봅시다.

그럼 20,000



## Gulp 홈페이지

----------

👉  [Gulp.js](https://gulpjs.com/)
