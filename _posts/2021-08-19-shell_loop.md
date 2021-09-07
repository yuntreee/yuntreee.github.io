---
title: "[쉘 스크립트] 반복문"
date: '2021-08-19 23:55:14'
categories: Shell
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## for 문

C/C++의 형식과 비슷하게 사용할 수 있다. 대신 소괄호"()"를 두번 써야 하며 중괄호"{}"로 묶이는 대신 do 와 done으로 묶인다.

```shell
#!/bin/sh
#for.sh

for((i=0;i<5;i++))
do
        echo $i
done
exit 0
```



혹은 파이썬의 for 문처럼 사용할 수 있다.

```shell
#!/bin/sh
#for.sh

for i in 0 1 2 3 4
do
        echo $i
done
exit 0
```

![image](https://user-images.githubusercontent.com/60495897/130087171-09c6a9a6-7f26-4f6b-bfc2-2180038b6ff3.png)



## while 문

C/C++과 비슷한 형식이지만, 조건식이 소괄호"()" 대신 대괄호"[]" 안에 들어가며 중괄호"{}" 대신 do와 done으로 묶인다. 

```shell
#!/bin/sh
#while.sh

echo "Enter Password"
read passwd
while [ $passwd != "1234" ] 
do
        echo "Wrong!!!"
        read passwd
done
echo "Correct"
exit 0
```

![image](https://user-images.githubusercontent.com/60495897/130089333-74efbd61-7998-4771-a44a-2dd4451f1dc0.png)



## break, continue

break와 continue 또한 사용 가능하다. 대신 ;; 를 붙여 사용한다.

```shell
#!/bin/sh
#for2.sh

echo "Enter yuntreee to break the loop"

for ((;;))
do
        read str
        case $str in
                yuntreee)
                        break;;
                *)
                        echo "Enter Again"
                        continue;;
        esac
done
echo "Good!!!"
exit 0
```

![image](https://user-images.githubusercontent.com/60495897/130090813-0ee8a77e-b7ed-417e-aca1-91163ef5b9b4.png)