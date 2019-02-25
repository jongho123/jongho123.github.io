---
layout: post
title: Nurumi Refactoring
date:  2019-02-22
last_modified_at: 2019-02-25
summary: 키보드 앱의 한글 오토마타 리팩토링
categories: posts
---

## 개요

- 2학기 졸업프로젝트로 다른 팀에 합류
- 최종 목적은 사용자의 키보드 커스터마이징
- 오토마타 리팩토링 부분을 맡아서 함.
- [소스코드(github)](https://github.com/2015nlpcapstone/Nurumi)

## 기술 스택

- Android / JAVA

## 개발 내용

### 오토마타 리팩토링

- **목적**
  - 최종적으로 키보드를 커스터마이징을 가능하도록 하고 싶지만 현재 상태로는 확장의 어려움이 있음.
  - 조건문으로 케이스마다 개발로 이해하기 어렵고 유지보수하기 힘듦.
  - 자바 패턴을 적용해 보고 싶었음.
  - 따라서 오토마타 부분을 리팩토링 해보기로 함.
    
- **문제점**
    - IF 문의 중첩으로 깊이가 깊고, 케이스 마다 IF문의 사용으로 길이가 길어 가독성이 떨어짐.     
    적게는 2단계 깊게는 4단계까지 다양했고 버퍼의 케이스마다 IF문을 구성.
    ~~~java
      // Before
      // 종성의 상태일 때 입력된 값이 모음인지 확인하고 중성의 값이 'ㅗ' 일때 입력된 값이 'ㅏ' 이면 중성에 있던 값을 'ㅘ'로 변경해주는 코드
      private void LEVEL_JONG_SEONG() {
          KoreanCharacter temp = kMap.get(motion);

          if(temp.getType() == MOEUM) {
              if(buffer[JUNG_SEONG].getCharNum() == OH.getCharNum()) {
                  if(temp.getCharNum() == AH.getCharNum()){
                      ic.deleteSurroundingText(1, 0);
                      buffer[JUNG_SEONG] = kMap.get(18L); // 을 으로
                      setText(String.format("%c", generate_korean_char_code(buffer[CHO_SEONG].getCharNum(),
                            buffer[JUNG_SEONG].getCharNum(), 0)));
                      automata_level = LEVEL_JONG_SEONG;
                  }
              ....
    ~~~
    - 키보드 추가를 하려면 버퍼의 상태에 따른 케이스마다 위의 조건문을 수정해야 하므로 확장의 어려움.  
    ![before-refactoring-simple-class](/images/before-simple-class.png)

- **해결방안**
    - 자바의 패턴을 찾아보던 중 상태 패턴을 찾음
    - 오토마타의 상태에 따라 입력에 대한 처리가 다르므로 상태 패턴과 비슷하다고 생각해서 사용해 봄.
    - 오토마타 상태 다이어그램
    ![automata](/images/after-automata.png){:width="100%" data-action="zoom"}
    - 오토마타 클래스 다이어그램
    ![automata-class](/images/automata-class.png)

- **결과**
    - 기존 한글 키보드 클래스 평균적으로 1000줄에 달하던 코드를 공통된 코드를 뽑아 100줄 내외로 줄임. (공통 부분은 6~700줄 정도)
    - 키보드의 키맵과 복자음[^1], 복모음[^2] 등의 구현만 해주면 되므로 확장이 용이해졌음
    ~~~java
      public class KoreanNaratgul extends Korean {
        public KoreanNaratgul() {
          asc.setKorean(this);

          // one dot
          kMap.put(32L, YIEUNG);    // ㅇ
        ...
        }

        protected KoreanCharacter buildBokJaEum(KoreanCharacter first, KoreanCharacter second) {
          ...
          }
        }
      }
    ~~~
    - 공통된 코드의 분리로 짧아져서 이해하기 쉬워 짐.
    ~~~java
        // Before
        // 종성의 상태일 때 입력된 값이 모음인지 확인하고 
        // 중성의 값이 'ㅗ' 일때 입력된 값이 'ㅏ' 이면 중성에 있던 값을 'ㅘ'로 변경해주는 코드
        KoreanCharacter temp = kMap.get(motion);

        if(temp.getType() == MOEUM) {
            if(buffer[JUNG_SEONG].getCharNum() == OH.getCharNum()) {
                if(temp.getCharNum() == AH.getCharNum()){
                    ic.deleteSurroundingText(1, 0);
                    buffer[JUNG_SEONG] = kMap.get(18L); // 을 으로
                    setText(String.format("%c", generate_korean_char_code(buffer[CHO_SEONG].getCharNum(),
                          buffer[JUNG_SEONG].getCharNum(), 0)));
                    automata_level = LEVEL_JONG_SEONG;
                  ...

        // After
        // 키보드 클래스에서는 단순히 합쳐질 수 있는 값인지 아닌지만 판단하고 오토마타에 넘김
        ...
        switch (first.getName()) {
          case 'ㅗ':
              if (second == AH) bokMoEum = WA;  // ㅘ
              break;
          ...
        }
        return bolMoEum;
   ~~~
    
    - 오토마타 코드의 일반화로 코드의 양이 줄고 이해하기 좋아짐.
    ~~~java
        // 오토마타 코드
        // 종성에 모음이 입력 되었을 때 복모음으로 만들 수 있으면 변경해주는 코드
        else if (inputChar instanceof Vowel) {
            if ((tempBuildBokMoEum = korean.buildBokMoEum(buffer[1], buffer[2])) != null) {
                buffer[1] = tempBuildBokMoEum;
                result = String.format("%c", korean.generate_korean_char_code(((Consonant) buffer[0]).getCharNumCho(), 
                                       buffer[1].getCharNum(), 0));
    ~~~

    - 팀원들도 이전보다 확실히 코드가 보기 편해졌다고 평가

- **아쉬운 점**
    - 단순히 비슷한 기능이 있다는 이유로 코드 중복을 줄이기 위해 중성 클래스를 상속받아 3자음 중성, 종성 클래스를 만듦. 
    하지만 단순히 기능이 비슷하다고 상속을 하는 것은 잘못이라는 것을 알게 됨.  
    ![automata-problem-class](/images/problem-class.png)  
    - 키보드의 복모음과 복자음 처리하는 메소드를 오토마타에서 사용하게 하려다보니 참조 구조가 복잡하게 됨. 자료구조를 이용해서 오토마타에 전달하고 처리하는게 더 나은 방법일 것 같음.
    ~~~java
      // 키보드 클래스에서 오토마타에 키보드 클래스 전달
      public KoreanAdvancedNaratgul() {
        asc.setKorean(this); 
        ...
      }

      // 오토마타 부분에서 키보드 클래스의 복모음, 복자음 메소드 사용
      public String buildCharacter(AutomataStateContext context, KoreanCharacter inputChar, KoreanCharacter buffer[]) {
        ...
        Korean korean = context.getKorean();

        ...
        if ((tempBuildBokMoEum = korean.buildBokMoEum(buffer[0], inputChar)) != null) {
        ...
    ~~~


## Footnotes

[^1]: 복자음: 겹쳐지는 자음. 일반적인 'ㄳ','ㄺ' 이외에도 키보드 특성상 'ㅇ'+'ㅇ'->'ㅎ' 같은 경우도 포함.
[^2]: 복모음: 겹쳐지는 모음. 'ㅗ' + 'ㅏ' -> 'ㅘ'
