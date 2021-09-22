# Blog 사용설명서

Create: 2021년 9월 12일 오후 10:00

블로그에 무슨 사용설명서가 필요할까 싶지만  
개발 블로그라는 목적이 있어 명확한 표현방식에 규칙을 이용해서 만들려고 한다.  
이에 Blog 사용 설명서를 만들어 보는 이들에 도움을 주고자 한다.  
개발서적을 보면 책에 사용되는 표현 방법들을 이야기 한것과 비슷하다고 생각하면 될것 같다.

## 목차

모든 목차는 `제목1` 이라는 표현형식으로 표현됩니다.  
블로그에서는 제목 다음으로 가장 큰글씨 입니다.  
목차는 다음과 같은 구성을 같습니다.

1. 개요 및 이론적은 내용 이것을 고민하게된 이유
2. Step By Step : 따라하기
4. Troubleshooting : 하면서 삽질 했던것 에러나는것들을 정리
3. TL;DR : 실질적인 코드부분, 빠른 사용을 위한 핵심내용

## 주의사항

!!! note
    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla et euismod
    nulla. Curabitur feugiat, tortor non consequat finibus, justo purus auctor
    massa, nec semper lorem quam in massa.

## 따라하기이미지
??? note "여기를 눌러서 확인"
    Lorem ipsum dolor sit amet, consectetur adipiscing elit. Nulla et euismod
    nulla. Curabitur feugiat, tortor non consequat finibus, justo purus auctor
    massa, nec semper lorem quam in massa.



## 코드

```bash hl_lines="1"
//직접적으로 입력 해야 하는 명령들을 여기에 표기 합니다.
#!/bin/bash
clear
StartTime=$(date +%s)
this_folder=$(pwd)

username="your's name"
useremail="you@are-mail.com"
commit_msg="$(date '+%Y%m%d%H%M%S')"

work_dir=/Users/${USER}/work
source_name=android_org
source_dir=${work_dir}/${source_name}

function sync_bit() {
  if [ ! -d "$target_dir/" ]; then
    cd ${work_dir} || exit
    $target_clone
  fi
  cd "${target_dir}" || exit
  git pull
  git config user.name "$username"
  git config user.email "$useremail"
  rsync -vah --delete --exclude=.git --exclude-from=${source_dir}/.gitignore  ${source_dir}/ ${target_dir}
  cd "${target_dir}" || exit
  git add .
  git commit -m "$commit_msg"
  git push
}
```


=== "C"

    ``` c
    #include <stdio.h>

    int main(void) {
      printf("Hello world!\n");
      return 0;
    }
    ```

=== "C++"

    ``` c++
    #include <iostream>

    int main(void) {
      std::cout << "Hello world!" << std::endl;
      return 0;
    }
    ```
<!-- - [x] Lorem ipsum dolor sit amet, consectetur adipiscing elit
- [ ] Vestibulum convallis sit amet nisi a tincidunt
    * [x] In hac habitasse platea dictumst
    * [x] In scelerisque nibh non dolor mollis congue sed et metus
    * [ ] Praesent sed risus massa
- [ ] Aenean pretium efficitur erat, donec pharetra, ligula non scelerisque -->


## 인용

> 실행의 결과물 output 등을 인용을 통해 표시합니다.

코드에 윗부분은 내용이 들어가야 하는 파일, 콘솔, 등이 표현되고 코드 내부는 직접적으로 붙여 넣기에 적합한 내용만 들어갑니다.

예시

D:\blog>에서 입력

```shell
dir
```

> D 드라이브의 볼륨에는 이름이 없습니다.  
볼륨 일련 번호: A00E-09BF  
D:\blog 디렉터리  
2021-09-12  오전 04:10    <DIR\>          .  
2021-09-12  오전 04:10    <DIR\>          ..  
...  
2021-09-12  오전 03:51             2,653 _config.yml  
6개 파일             195,658 바이트  
9개 디렉터리  81,542,119,424 바이트 남음  


<!-- ## 참고링크

목차 다음에는 참고 링크를 첨부해서 직접적인 정보를 얻은 부분을 표시해서 원문이나 더 많은 정보를 확인 할수 있도록 합니다.

## TL;DR

빠른 사용을 위한 코드 -->

<!-- ## 이미지 삽입

내부이미지 삽입

![Untitled](Untitled.png)


외부이미지 삽입

![https://media.giphy.com/media/f9eSYJ9UbDd7xTZKKL/giphy.gif](https://media.giphy.com/media/f9eSYJ9UbDd7xTZKKL/giphy.gif) -->

