---
title: "[쉘 스크립트] 조건문"
date: '2021-09-20 21:21:14'
categories: Shell
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## 1. 함수 선언, 호출, 매개변수

함수 선언 형식은 다음과 같다.

```bash
# 함수 선언
함수이름 () {
	내용
}

# 함수 호출
함수이름
```



```bash
#! /bin/bash

print_hello() {
    echo "hello"
	return
}

print_hello
------------------------
hello
```



파라미터를 사용하려면 함수를 호출할 때 함수 뒤에 붙이면 되며, 함수 내에서는 $1, $2 ... 로 사용한다.

```shell
#! /bin/bash

add() {
    echo `expr $1 + $2`
}

echo "Enter two numers"
read a b
echo "$a + $b = $(add $a $b)"
------------------------------
Enter two numers
3 7
3 + 7 = 10
```

```read a b``` 하면 띄어쓰기를 기준으로 a와 b에 변수를 입력한다.



## 2. 불러오기

하나의 쉘 스크립트에 변수와 함수선언을 작성하고 다른 스크립트에서는 불러오기만 하고 싶을 때 ```source``` 를 사용한다.

```bash
#! /bin/bash
# func.sh

add() {
    echo `expr $1 + $2`
}
---------------------------------
#! /bin/bash
# main.sh

source func.sh

echo "Enter two numbers"
read a b
echo "$a + $b = $(add $a $b)"
---------------------------------
$ ./main.sh
Enter two numbers
12 3
12 + 3 = 15
```



## 3. 특정 값 리턴

함수를 통해 어떤 값을 리턴하는 방식이 일반적인 프로그래밍 언어와 다르다.

### 1) echo 를 통한 전달

함수 내에서 echo를 하고, 이를 호출할 때 결과를 변수에 담아서 활용한다.

```bash
#! /bin/bash

add() {
    echo `expr $1 + $2`
}

read a b
rst=$(add $a $b)
echo "result is $rst"
--------------------------
2 5
result is 7
```



### 2) 전역변수를 통한 전달

함수 내에서 전달할 변수를 전역변수에 담고 이를 밖에서 활용하는 방식이다.

```bash
#! /bin/bash

is_10 () {
    if [ $1 -eq 10 ]
    then
        rst="is 10"
    else
        rst="not 10"
    fi
}

read a
is_10 $a
echo $rst
------------------------
15
not 10
```

