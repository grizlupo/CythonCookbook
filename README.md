# CythonCookbook

### \#define으로 정의된 상수 사용하기

일단 상수가 정의된 헤드 파일에 대해 `extern` 선언으로 선언하고, 그 아래에 enum으로 필요한 상수들을 나열하면 된다.

```cython
cdef extern from 'header.h':
    enum:
        MAX_LENGTH
```