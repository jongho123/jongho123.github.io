---
layout: post
title: Recommend indie
cover: cover.jpg
date:  2019-02-28
last_modified_at: 2019-04-01
project_date: 201504
summary: 음악 추천 앱 개발
categories: posts
---

# Recommend Indie

<div class='ui divider'></div>

## 개요

프로젝트 중심의 수업에서 했던 프로젝트입니다. 음악 추천 API를 이용한 음악플레이어 앱 개발을 했습니다.  
안드로이드와 서버를 개발했고 비슷한 음악을 추천해주는 API 서버와 연동했습니다.

[소스코드(github)](https://github.com/jongho123/Recommandation-Indie)

<div class='ui divider'></div>

## 기술 스택

- Node.js / Express 
- Android
- MongoDB
- AWS EC2 (서버) / S3 (음원)

<div class='ui divider'></div>

## 개발 내용 (서버)

<div class='ui divider'></div>

### 음악 추천 API 서버와 연동

**소개**  

URL : /recommendation  
음악 추천 API를 이용해 원하는 음악과 비슷한 음악을 찾아주는 API.

추천 서버 API를 통해 비슷한 음악의 제목을 얻어 오고 제목을 이용해 곡의 정보와 youtube video id를 받아오고 음악 정보를 응답해준다.

**문제점**

가장 비슷한 음원은 똑같은 음원이기 때문에 단순히 가장 비슷한 음악을 가져오면 같은 음악이 추천되는 경우가 있었음.

따라서 비슷한 음악의 리스트 중 score가 90점을 넘는 곡 중 랜덤하게 뽑아옴. 없으면 가장 점수가 높은 노래를 추천.

**소스코드**

90점 이상의 score를 가지는 음원 중에서 랜덤하게 하나를 뽑는 소스코드
```javascript
  // tracks: 추천 API 서버에서 받은 비슷한 음원들의 정보. score 내림차순
  var i = 0;
  while(i < tracks.length && Number(tracks[i].score) > 90) { ++i }
  i = Math.floor(Math.random() * i);

  // 추천 API 서버에 곡 정보와 youtube id 를 요청하기 위한 세팅 
  var searchObj = new Object();
  searchObj.title = tracks[i].title;
  ...
```

<div class='ui divider'></div>

### 음악 Feature 추출

**소개**

URL: /analysis

업로드된 음원의 Feature 데이터를 추출한다. 추출한 Feature를 이용해 음악 추천 API 서버에 비슷한 음원을 요청함.

**문제점**

추천 API를 호출하기 전에 음악의 Feature를 추출해야하는데 추출하는 시간이 오래걸림.
process fork 를 통해 실행하면 더 오래 걸릴 것 같았고, node-gyp을 통해 c++로 빌드할 수 있다고 듣고 코드에 심어서 직접 실행시켜 보려고 했지만 빌드에 동적링크가 잘 안되서 해결하지 못함.

차선책으로 추출을 할 때 시간을 걸려 하더라도 다음엔 빠르게 받을 수 있길 바랬음. 따라서 이전에 추출해놨던 Feature를 DB에 저장해 두고 다음에 요청시 같은 음악이 있으면 새로 추출하지 않고 미리 추출해놨던 데이터를 쓰도록 함.  

또한 추천된 음악을 youtube에서 찾아 스트리밍 해준 뒤 추천된 음원의 Feature를 미리 추출해 놓음.

**소스코드**

DB에 Track의 데이터가 있는지 확인하는 코드. 있으면 Feature의 데이터도 있다는 것.
```javascript
  // playinfo: 분석될 음원의 제목과 가사 정보
  Track.find(playinfo, function(err, track) {
    if (err) return callback(err);
    else if (track.length > 0) {
      res.sendStatus(200);
    }
  }
```

fork 를 통해 추출프로그램으로 Feature를 추출하는 코드
```javascript
  // cp: child process module
  var extract = cp.fork('./extract.js');

  extract.on('exit', function(code, signal) {
    callback(null);
  });

  extract.send({ input: tempMusicFile, output: featureFile });
```

추출한 Feature와 음원의 정보를 DB에 저장
```javascript
  Track.create({ title: playinfo.title, artist: playinfo.artist, feature: featureData }, function (err) {
    if (err) return callback(err);
    fs.unlink(featureFile);
    fs.unlink(tempMusicFile);
    res.sendStatus(200);
    callback(null, 'analysis complete');
  });
```

처음 추출할 때는 약간 느린감이 있으나 다음부터는 확실히 빨랐음. 특히 스트리밍 후 미리 추출을 해놓기 때문에 음악을 이어들을때는 거의 딜레이 없이 받을 수 있었음.

<div class='ui divider'></div>

### 음악 스트리밍

**소개**

URL: /streaming

음악 추천 API 에서 음원을 주는 것이 아니라 유튜브의 video id를 반환하기 때문에 id를 받아 음원을 스트리밍함.

**문제점**

기존에는 youtube id를 이용해 youtube에서 url로 음원을 받을 수 있었음. 음원 얻는 방법을 검색하면 url로 음원을 받는 방법의 소개가 많았으나 개발하기 얼마전에 막혔음.

youtube-dl이라는 라이브러리를 통해 음원을 받아왔으나 Feature 데이터 추출과 마찬가지로 받아오는데 시간이 좀 걸림.

따라서 음원 추출할 때와 마찬가지로 받아온 음원을 아마존 S3에 저장해 둠. 음원이 S3에 있으면 S3에서 가져와서 바로 스트리밍함.

**소스코드**

trackId가 있을때 S3에서 음원을 찾아보고 있으면 바로 스트리밍하는 코드
```javascript
  var params = { params: { Bucket: 'jhmusic', Key: trackId + '.mp3'} };
  s3 = new AWS.S3(params);

  s3.headObject({}, function(notExist, data) {
    if(notExist) callback(null); 
    else {
      var musicstream = s3.getObject().createReadStream();
      musicstream.pipe(res);
      musicstream.on('end', function(){
        ...
      });
    }
  });
```

youtube id를 이용해 음원을 다운받는 코드. 저장은 mp3 파일로 저장됨.
```javascript
  var youtubedl = cp.fork('./youtube-dl.js');
  youtubedl.on('exit', function(code, signal) {
    callback(null);
  });
  youtubedl.send({ url: "https://www.youtube.com/watch?v=" + req.params.videoId, filename: filename + ".m4a" });
```

다운받은 음원 파일을 S3에 저장하는 코드. 이후에 스트리밍 함.
```javascript
  var putStorage = fs.createReadStream(filename + '.mp3');
  s3.upload({Body: putStorage}).on('httpUploadProgress', function(evt) {
  }).
  send(function(err, data) {
    if(err) return callback(err);
    callback(null);
  });
```

스트리밍 이후에는 Feature를 추출해서 DB에 저장

<div class='ui divider'></div>

### 음원의 정보 제공

**소개**

URL: /musicinfo

안드로이드에서 뮤직플레이어에 띄울 정보들 제공.

**소스코드**

음악의 정보를 찾아오는 코드
```javascript
  MusicInfo.find({ track_id: trackId }, function(err, track) {
    if(err) return callback(err);
    else if(track.length == 0) return callback(new Error('No track with track_id ' + trackId + ' found.'));
    callback(null, track[0]);
  });
```

트랙의 정보를 담아 응답해주는 코드
```javascript
  var resMusicInfo = JSON.stringify({ track_id: trackId, url: videoId, title: track.title, artist: track.artist, like: track.like, unlike: track.unlike });  
  res.end(resMusicInfo);
  callback(null);
```

썸네일같은 부분도 함께 넣어두면 좋을 것 같은데 따로 함. 그래서 안드로이드에서 따로 썸네일을 찾아오는 코드가 필요하게됨.

<div class='ui divider'></div>

### 음악 추천 서버에 음악 등록

**소개**

URL: /register

음악을 등록해 다른 유저가 새로운 음악을 추천받을 수 있도록 함.

**소스코드**

등록한 음원이 추천될 수 있도록 음악 추천 서버에 등록하는 코드
```javascript
  var inputObj = new Object();
  
  // inputObj set data..
  ...

  var input = querystring.stringify({ 'data' : JSON.stringify(inputObj) }); 

  // 추천 서버의 play를 호출하여 음원을 등록하면 음악 추천을 받을 수 있게 됨.
  options.path = '/soundnerd/user/play',
  options.headers = { ... }

  var playReq = http.request(options, function(playRes) {
    var body = "";
    playRes.on('data', function(chunk){
      body += chunk;
    })
    .on('end', function(){
      console.log(body);
      callback(null);
    });
  });

  playReq.end(input);
```

기존 서버에 있던 음원들과 분류를 위해 앨범은 created로 생성함.
이렇게 추천된 음원은 youtube id가 없고 음원들은 S3에 있음.

위의 /recommendation 에서 음원 추천시 먼저 등록한 음원 중에 추천이 되고, 등록된 음원이 없을 때는 전체 추천 리스트 중 알고리즘에 따라 추천되게 됨.

<div class='ui divider'></div>

### 음악 추천 서버의 음악 검색 기능

**소개**

URL: /find

음악 추천 서버 내의 DB에 등록된 음악을 검색

**소스코드**

음악 추천 서버의 search API를 통해 음악의 키워드로 검색하는 코드
```javascript
    var inputObj = new Object();

    // inputObj 키워드 값 세팅
    ...

    var input = querystring.stringify({ 'data' : JSON.stringify(inputObj) });

    options.path = '/soundnerd/music/search',
    options.headers = { ... }

    var searchReq = http.request(options, function(searchRes){
      var body = '';
      searchRes.on('data', function (data) {
        body += data;
      })
      .on('end', function () {
        var tracks = JSON.parse(body).tracks;
        callback(null, tracks);
      });
    });

    searchReq.end(input);
```

키워드를 주면 키워드와 비슷한 음원 10개를 리턴해줌.

<div class='ui divider'></div>

### MongoDB

**소개**

데이터 저장을 위해 사용.

**소스코드**

MusicInfo collection
```javascript
var MusicInfoSchema = new Schema({
  track_id:String,
  title:String,
  artist:String,
  like:Number,
  unlike:Number,
  count:Number
});
```
Track collection
``` javascript
var TrackSchema = new Schema({
  title:String,
  artist:String,
  feature:String
});
```

Log collection
```javascript
var LogSchema = new Schema({
  user_id:String,
  request:String,
  log:String,
  createdAt:{ type:Date, default:Date.now() }
});
```

<div class='ui divider'></div>

## 개발 내용 (안드로이드)

<div class='ui divider'></div>

### 서버에 음악 등록

**소개**

음악을 공유하기 위해 서버에 등록하는 기능. 이를 통해 음악 추천 서버에서 비슷한 음원 추천시 등록한 음악이 추천될 수 있도록 할 수 있음.

폰에 있는 음악 파일 리스트를 보여줌. 음원을 선택하고 음악 타이틀을 입력해서 등록 버튼을 누르면 등록처리가 됨.

서버의 /register로 음악의 정보와 음원의 등록을 요청.
 
<div class='ui divider'></div>

### 음악 추천 및 스트리밍

**소개**

내가 듣는 음악의 비슷한 음악을 추천 받아 재생함.

재생 전에 음원 주소와 화면에 표시될 음악 정보 등을 얻기 위해 서버에 몇 가지 요청을 함.
- /analysis 음원의 Feature 추출
- /musicinfo 를 통해 음원 데이터 요청.
- 유튜브에서 썸네일을 다운로드 받음.

이전에 추천받아 재생되었던 음악이 있으면 그 음악을 Seed로 비슷한 음악부터 재생.

**아쉬운점**

서버와 HTTP 통신하는 부분은 거의 비슷하므로 따로 클래스로 묶었으면 더 깔끔한 코드가 될 수 있었을 것 같음.

<div class='ui divider'></div>

### 음악 평가 기능

**소개**

음악의 평가 시스템으로 듣는 이용자의 반응을 확인할 수 있음.

좋아요, 싫어요 아이콘 클릭으로 평가할 수 있음.

<div class='ui divider'></div>

### 음악 검색 기능

**소개**

원하는 음악을 검색으로 그 음원을 기준으로 추천을 받거나 원하는 음악을 찾아 들을 수 있음.

키워드를 입력하면 서버에서 최대 10개의 리스트를 받음.
- 한번 클릭으로 그 음악을 기준으로 비슷한 음악을 추천 받을 수 있음.
- 길게 누르고 있으면 해당 음악을 들을 수 있음.

<div class='ui divider'></div>

## 후기

개발 당시 목표를 크게 잡았고 할수 있는데 까지 해보자 생각하며 개발을 시작함. 뒤로 갈수록 하고 싶은건 많은데 시간은 없으니 머리는 복잡하고 일단 만들고 보니 코드도 지저분해졌음.  
결국 원하는데까지 만들지는 못했고 코드가 지져분해져서 이해하기만 더 힘들어졌음. 내가 해낼 수 있는 양을 파악하고 조금 더 계획적으로 접근했으면 더 나은 결과가 나오지 않았을까 생각됨.
