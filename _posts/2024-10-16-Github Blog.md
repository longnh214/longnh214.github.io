---
title: Fork한 Github Blog에 커밋 시 잔디가 생기지 않는 현상 해결
date: 2024-10-16 11:01:55 +09:00
categories: [Problem Solve, Blog]
tags:
  [
    Problem Solve,
    Github Blog,
    jekyll,
    fork,
    fork repository,
    잔디
  ]
---
## 문제 및 원인 파악

![스크린샷 2024-09-25 오전 10 36 18](https://github.com/user-attachments/assets/20194724-8824-44ef-8e22-4dcd4977ec62)

이 블로그는 jekyll repository를 fork하여 생성한 github blog로 블로그 포스팅을 하고 있었다. 내 Github의 contributions(잔디)를 보니 잔디가 심어지지 않고 있어서 'fork 해왔더라도 내 계정 아이디 소유의 repository에 커밋을 했는데 왜 잔디가 심어지지 않는걸까?'라는 의문이 들었고, 안되는 이유를 찾아보았다.

대부분의 잔디가 안 심어지는 이유는 **git config에 등록된 이메일 주소가 github 계정의 이메일 주소와 일치하지 않아서**인데, 본인은 다른 repository의 경우에 잔디가 잘 쌓이므로 이 경우는 아니었다.

[Why are my contributions not showing up on my profile?](https://docs.github.com/ko/account-and-profile/setting-up-and-managing-your-github-profile/managing-contribution-settings-on-your-profile/why-are-my-contributions-not-showing-up-on-my-profile)

우선 Github 공식 문서에 나와있는 commit이 표시되는 경우를 번역해본다.

![스크린샷 2024-10-15 오후 12 46 40](https://github.com/user-attachments/assets/7d2a971a-50f1-436c-86b9-c1022fc707ab)

* 커밋의 이메일 주소가 github 계정과 맞아야하한다.
* fork repository가 아닌 개인 repository에서 커밋이 이루어져야한다.
* Repository의 기본 브랜치(예시로 `main`, `master`)에서 커밋이 되었다.
* (Github Blog의 경우) `gh-pages` 브랜치에서 커밋이 되었다.

> 위 경우 중에서 2번째의 이유때문에 commit이 표시되지 않았다.

## 문제 해결
Fork된 기존의 repository를 옮기고 새 repository를 생성해서 해결할 수 있지만, 기존의 commit 이력이 유지되지 않기 때문에 다른 방법을 찾았다. 검색해보니 Github에서 fork된 repository를 개인 repository로 분리할 수 있는 방법이 있다는 것을 알 수 있었다.

## Github의 Virtual Agent를 통한 Detach Fork

[https://support.github.com/contact?tags=rr-forks&subject=Detach%20Fork&flow=detach_fork](https://support.github.com/contact?tags=rr-forks&subject=Detach%20Fork&flow=detach_fork)

위 링크를 눌러서 들어가면 자동으로 챗봇 Virtual Agent의 진행에 따라 Detach Fork를 진행하면 된다.

![스크린샷 2024-09-25 오전 10 32 19](https://github.com/user-attachments/assets/6a096a24-a9ef-4374-b0b9-7d38184948ed)

스크린 샷의 하단 버튼 `Delete`와 `Detach/Extract` 중에서 `Detach/Extract`를 클릭한다.

![스크린샷 2024-09-25 오전 10 32 56](https://github.com/user-attachments/assets/6fd827c3-9c5d-4b3a-a3b0-85fa8ca687f3)

`owner/repository-name` 형태로 Detach Fork 대상 Repository 이름을 기입한다.

![스크린샷 2024-09-25 오전 10 33 06](https://github.com/user-attachments/assets/71bb5d4f-937f-48b0-8b0f-5e14a82df424)

Fork된 Repository이기 때문에 `Yes`를 선택한다.
그리고 Repository를 분리하고 싶은 이유를 물어보는데, Contribution(잔디) 탭에 보이지 않는 이유이므로 `Contributions to the repository aren't visible`을 선택했다.

![스크린샷 2024-09-25 오전 10 33 14](https://github.com/user-attachments/assets/fd20df37-8d25-44d8-89b6-f8f62cc996f0)

Note에 보면, Detach 완료까지 최대 1시간까지 걸릴 수 있다고 나와있다. 하지만 실제로는 다음날 오전에 완료된 것을 확인할 수 있었다.

![스크린샷 2024-10-15 오후 1 12 55](https://github.com/user-attachments/assets/5d34307e-2fed-480a-8e1d-0788fb2d71ce)

Fork Repository에서 개인 Repository로 Detach된 것을 확인했다.

> 답답한 부분을 해결할 수 있도록 포스팅해주신 dev-jonghoonpark님께 감사합니다.

## 출처

* [https://jonghoonpark.com/2023/10/02/fork%ED%95%9C-github-%EC%A0%80%EC%9E%A5%EC%86%8C-%EB%B6%84%EB%A6%AC%ED%95%98%EA%B8%B0](https://jonghoonpark.com/2023/10/02/fork%ED%95%9C-github-%EC%A0%80%EC%9E%A5%EC%86%8C-%EB%B6%84%EB%A6%AC%ED%95%98%EA%B8%B0)
* [https://stackoverflow.com/a/16052845](https://stackoverflow.com/a/16052845)