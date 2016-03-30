Title: Cookbook
Date: 2016-03-29 23:36
Modified: 2016-03-30 11:22
Category: Cython

# Cookbook

### #define으로 정의된 상수 사용하기

일단 상수가 정의된 헤드 파일에 대해 `extern` 선언을 하고, 그 아래에 `enum`으로 필요한 상수들을 나열하면 된다.

```cython
cdef extern from 'header.h':
    enum:
        MAX_LENGTH
```