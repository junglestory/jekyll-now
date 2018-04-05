---
layout: post
title: 로컬 프로젝트를 Github에 등록 방법
---

Github을 사용해서 프로젝트를 관리하려면, Github 저장소를 로컬에 clone 해서 사용해야 하는데,
기존 프로젝트나 CLI(Command Line Interface) 등을 이용해서 프로젝트를 생성한 경우 Github에 등록하려면 오류가 발생한다.

구글에 찾아보면 비슷한 방법들이 있는데, 최근에 Github 저장소를 생성했던 방법을 설명하겠다.


1. **[Github](https://github.com/)**에 저장소를 생성한다.

2. 로컬에 프로젝트를 생성한다.

3. 로컬 프로젝트로 이동해서 로컬 저장소를 생성한다.
	```
    git init
    ```

4. 모든 파일을 저장소에 추가한다.
	```
    git add .
    ```

5. 로컬 저장소에 커밋한다.
	```
    git commit -m ‘Initial project'
    ```

6. Github에 생성한 원격 저장소를 추가한다.
	```
    git remote add origin https://github.com/[myaccount]/[myproject].git
    ```

7. 워킹 디렉토리의 파일을 저장한다.
	```
    git stash
    ```

8. 원격 저장소 파일을 로컬 저장소에 갱신한다.
	```
    git pull
    ```
    
9. 변경사항을 로컬 저장소에 커밋한다.
	```
    git commit -m ‘Merge project’
    ```
    
10. 로컬 저장소에 변경된 파일을 원격 저장소에 전송한다.
	```
    git push --set-upstream origin master
    ```
