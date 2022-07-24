---
title: "[AWS] 테라폼 이중 반복문 구현"
date: "2022-07-24 23:03:30"
categories: AWS
toc: true
toc_sticky: true
sidebar:
  nav: docs
---

## 1. 테라폼 이중 반복문 구현

테라폼에서는 이중 for문을 제공하지 않지만, 수학적으로 구현할 수 있다.

예를 들어, 다음과 같은 2개의 변수를 생성한다.

```yaml
variable "list1" {
default = ["a", "b", "c"]
}

variable "list2" {
default = ["1", "2", "3", "4"]
}
```

[tag1:"a", tag2:"1"], [tag1:"a", tag2:"2"] .... [tag1:"c", tag2:"4"]

이렇게 각각의 변수 조합된 태그가 들어간 총 12개의 vpc를 만들어본다.

```yaml
resource "aws_vpc" "nested_for_test" {
count      = length(var.list1) * length(var.list2)
cidr_block = "10.${count.index}.0.0/16"
tags = {
tag1 = var.list1[floor(count.index / length(var.list2))]
tag2 = var.list2[count.index % length(var.list2)]
}
}
```

단순하게 몫과 나머지를 사용하여 구현한다. floor()는 값을 내림해주는 함수이다.

`terraform plan` 을 해보면 결과를 볼 수 있다.

```yaml
# aws_vpc.nested_for_test[1]
"aws_vpc" "nested_for_test" {
  cidr_block        = "10.0.0.0/16"
  tags              = {
  "tag1" = "a"
  "tag2" = "1"
  }
}

# aws_vpc.nested_for_test[2]
"aws_vpc" "nested_for_test" {
  cidr_block        = "10.1.0.0/16"
  tags              = {
  "tag1" = "a"
  "tag2" = "2"
  }
}

# aws_vpc.nested_for_test[3]
"aws_vpc" "nested_for_test" {
  cidr_block        = "10.2.0.0/16"
  tags              = {
  "tag1" = "a"
  "tag2" = "3"
  }
}

.
.
.

# aws_vpc.nested_for_test[11]
"aws_vpc" "nested_for_test" {
  cidr_block        = "10.11.0.0/16"
  tags              = {
  "tag1" = "c"
  "tag2" = "4"
  }
}
```

원했던 대로 두 변수가 모두 서로 매핑된 결과를 확인할 수 있다.
