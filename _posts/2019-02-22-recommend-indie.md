---
layout: post
title: Recommend indie
cover: cover.jpg
date:  2019-02-28
last_modified_at: 2019-02-28
summary: 음악 추천 앱 개발
categories: posts
---
## 개요

- 프로젝트 기반의 수업에서 했던 프로젝트
- 음악 추천 API를 이용한 음악 플레이어 앱 개발
- 최종 목적은 인디 음악의 공유

## 기술 스택

- Node.js / Express
- Android
- MongoDB
- AWS EC2 / S3

## 개발 내용 (서버)

### 음악 추천 API 서버와 연동

- **목적**
  - 음악 추천 API를 이용해 원하는 음악과 비슷한 음악을 찾음.

- **문제점**
  - 무조건 가장 비슷한 음악을 가져오면 같은 음악이 추천되는 경우가 있음.

- **해결방안**
  - 비슷한 음악의 리스트 중 score가 90점을 넘는 곡 중 랜덤하게 뽑아옴. 없으면 그냥 가장 비슷한 노래를 추천.

- **결과**
  - /recommendation 
  - 추천 서버 API를 통해 비슷한 음악의 제목을 얻는다.
  - 해당 음악의 제목을 이용해 추천 서버에서 곡의 정보와 youtube video id을 받아오고 음악 정보를 응답해준다.

### 음악 Feature 추출

- **목적**
  - 음악의 유사도를 찾기 위해서는 음악 Feature 데이터의 추출이 필요함.
  - Feature 데이터를 추출하는 라이브러리는 제공해주셨음.

- **문제점**
  - API를 호출하기 전에 음악의 Feature를 추출해야하는데 추출하는 시간이 오래걸림.
  - process fork 를 통해 실행하면 더 오래 걸릴 것 같았고, node-gyp을 통해 c++로 빌드할 수 있다고 듣고 코드에 심어서 직접 실행시켜 보려고 했지만 빌드에 동적링크가 잘 안되서 실패함.

- **해결방안**
  - 이전에 추출해놨던 Feature를 DB에 저장해 둠. 다음에 요청시 같은 음악이 있으면 새로 추출하지 않고 미리 추출해놨던 데이터를 씀.
  - 추천 음악이 스트리밍 된 후에는 미리 추출을 해 놓음.

- **결과**
  - 처음 추출할 때는 약간 느린감이 있으나 다음부터는 확실히 빨랐음.
  - 특히 음악을 이어들을때는 거의 딜레이 없이 받을 수 있음.
  - 추출 테스트에서는 2~3초 정도 걸려 길다 싶었는데 막상 만들고 테스트해보니 생각보다 짧았음. 하지만 사용자가 많아지면 이 시간은 매우 부담되는 시간일 것이라고 생각함.

### 음악 스트리밍

- **목적**
  - 음악 추천 API 에서 음원을 주는 것이 아니라 youtube video id를 반환함. 
  - 따라서 youtube id를 받아 음원을 스트리밍해줘야 함.

- **문제점**
  - 기존에는 youtube id를 이용해 url로 음원을 받을 수 있었으나 막혔음.
  - youtube-dl 라이브러리를 통해 음원을 받아왔으나 이것도 시간이 좀 걸림.

- **해결방안**
  - 받아온 음원을 아마존 S3에 저장해 둠.
  - 음원이 S3에 있으면 S3에서 가져와서 바로 스트리밍함.

- **결과**
  - /streaming
  - S3에 음원이 있으면 S3에서 없으면 youtube-dl로 받아서 스트리밍.
  - youtube-dl로 받은 음원은 S3에 업로드.
  - 음원은 미리 Feature를 추출해놓음.

- **아쉬운점**
  - 비슷한 코드들은 모듈화 할 수 있었을 것 같음.


### 음원의 정보 제공

- **목적**
  - 안드로이드에서 뮤직플레이어에 띄울 정보들 제공.

- **결과**
  - /musicinfo
  - 넘겨준 track_id와 같은 음원 정보가 있으면 넘겨 줌.
 
- **아쉬운점**
  - 음악의 타이틀, 아티스트, 좋아요, 싫어요 개수 등을 반환
  - 썸네일도 함께 넣어두면 좋을 것 같은데 따로 함.

### 음악 추천 서버에 음악 등록

- **목적**
  - 음악을 등록해 다른 유저가 새로운 음악을 추천받을 수 있도록 함.

- **결과**
  - /register
  - 음악의 Feature를 추출해 음악 추천 서버에 play를 호출하면 추천 서버 DB에 등록이 됨.
  - 이후에는 비슷한 음원 추천시 추천 됨.
  - 이렇게 추천된 음원은 youtube id가 없고 음원들은 S3에 있음.

### 음악 추천 서버의 음악 검색 기능

- **목적**
  - 음악 추천 서버 내의 DB에 등록된 음악을 검색

- **결과**
  - 키워드를 주면 키워드와 비슷한 음원 10개를 리턴해줌.

### MongoDB

- **목적**
  - 다양한 데이터 저장을 위해 사용.

- **결과**
  - 음악의 Feature 정보, 타이틀, 아티스트, 평가 정보 등의 정보를 저장.
  - 어떤 유저가 어떤 요청을 했는지 등의 로그 정보 저장.

## 개발 내용 (안드로이드)

### 서버에 음악 등록

- **목적**
  - 음악을 공유하기 위해 서버에 등록
  - 등록함으로써 비슷한 음원 추천시 추천될 수 있음.

- **결과**
  - 폰의 음악 파일을 보여주고 음원을 선택하고 음악 이름을 적어서 등록 버튼을 누르면 등록처리가 됨.
  - 서버의 /register로 음악의 정보와 음원의 등록을 요청.
 
### 음악 추천 및 스트리밍

- **목적**
  - 내가 듣는 음악의 비슷한 음악을 추천 받아 재생

- **결과**
  - 음원 주소와 화면에 표시될 음악 정보 등을 얻기 위해 재생 전에 서버에 몇 가지 요청을 함.
  - MediaPlayer로 음원의 주소를 넘겨주어 음악이 재생되게 함.
  - 이전에 추천받아 재생되었던 음악이 있으면 그 음악부터 재생.

- **아쉬운점**
  - 나중에 시간이 촉박한 상태로 개발하다보니 필요한 부분은 없고 필요없는 부분은 있고 복잡해 짐.
  - 서버와 HTTP 통신하는 부분은 거의 비슷하므로 따로 클래스로 묶었으면 더 깔끔한 코드가 될 수 있었을 것 같음.

### 음악 평가 기능

- **목적**
  - 음악의 평가 시스템으로 듣는 이용자의 반응을 확인할 수 있음.

- **결과**
  - 음악의 재생 전에 서버에서 데이터를 받아와서 좋아요, 싫어요 수를 화면에 표시함.
  - 좋아요, 싫어요 아이콘 클릭으로 평가할 수 있음.

### 음악 검색 기능

- **목적**
  - 원하는 음악을 검색으로 그 음원을 기준으로 추천을 받거나 원하는 음악을 찾아 들을 수 있음.

- **결과**
  - 키워드를 입력하면 서버에서 10개의 리스트를 받음.
  - 한번 클릭으로 그 음악을 기준으로 비슷한 음악을 추천 받을 수 있음.
  - 길게 누르고 있으면 해당 음악을 들을 수 있음.
