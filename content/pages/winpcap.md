Title: winpcap wrapping
Date: 2016-03-30 12:38
Modified: 2016-03-30 12:39
Category: Cython

winpcap은 libpcap의 윈도우즈 판이고, 이걸 파이썬에서 사용할 수 있도록 해주는 모듈도 이미 있다. 그런데도 같은 기능의 모듈을 만들려는 것은 Cython을 연습해보려는 것이 가장 크고, 그 다음이 libpcap에 익숙해지려는 것이다. 이 글은 64비트 윈도우즈 10에서 파이썬은 3.5, Cython은 0.23.4을 사용하는 환경에서 쓰고 있다.

우선은 winpcap이 필요하다. [http://www.winpcap.org/install/default.htm](http://www.winpcap.org/install/default.htm/)에서 받을 수 있다. 이 글을 쓰고 있는 현재 최신 버전은 4.1.3이다. 라이브러리도 챙겨야 한다. [http://www.winpcap.org/devel.htm](http://www.winpcap.org/devel.htm/)에서 받을 수 있다. 버전은 4.1.2다. 4.1.3에 사용해도 된다고 되어 있다. winpcap은 설치하고, 라이브러리는 적당한 곳에 압축을 푼다.

## 1 단계

먼저 libpcap 자체가 아직은 낯서니까 익숙해지는 과정으로 winpcap 문서에 있는 학습서tutorial을 Cython으로 진행해 볼 것이다.

```c
#include "pcap.h"

main()
{
    pcap_if_t *alldevs;
    pcap_if_t *d;
    int i=0;
    char errbuf[PCAP_ERRBUF_SIZE];
    
    /* Retrieve the device list from the local machine */
    if (pcap_findalldevs_ex(PCAP_SRC_IF_STRING, NULL /* auth is not needed */, &alldevs, errbuf) == -1)
    {
        fprintf(stderr,"Error in pcap_findalldevs_ex: %s\n", errbuf);
        exit(1);
    }
    
    /* Print the list */
    for(d= alldevs; d != NULL; d= d->next)
    {
        printf("%d. %s", ++i, d->name);
        if (d->description)
            printf(" (%s)\n", d->description);
        else
            printf(" (No description available)\n");
    }
    
    if (i == 0)
    {
        printf("\nNo interfaces found! Make sure WinPcap is installed.\n");
        return;
    }

    /* We don't need any more the device list. Free it */
    pcap_freealldevs(alldevs);
}
```

### main 함수 껍데기 만들기

일단은 구체적인 처리는 잠시 접어두고, 파이썬에서 껍데기 뿐이더라도 main을 호출하도록 해 보자! 이건 libpcap보다는 Cython에 대한 것이다.

_pcap.pyx라는 파일을 만든다. 내용은

```cython
def main():
    print('pcap')
```

그리고 이걸 빌드할 파일과, 테스트할 파일을 만든다.

빌드용으로는 아래 내용으로 setup.py 파일을 만든다.

```python
from distutils.core import setup
from distutils.extension import Extension
from Cython.Build import cythonize

ext_modules = cythonize(
    Extension(name = '_pcap',
        sources = ['_pcap.pyx'])
    )

setup(name ='pcap',
    ext_modules = ext_modules,
    )
```

테스트는 아래 내용으로 test.py를 만든다.

```python
import _pcap

_pcap.main()
```

그리고 실제 빌드는 아래 명령을 계속 해야 하는데 귀찮으니까 배치파일을 하나 만든다. build.bat다.
 
```sh
py -3 setup.py build_ext --inplace
```

제대로 되었다면 _pcap.cp35-win_amd64.pyd가 만들어진다. 그리고 test.py를 했을 때 화면에 'pcap'이 찍히면 일단 첫 고비는 넘은 것이다.

### pcap_findalldevs_ex 함수 호출하기 

이제 _pcap.pyx에 아래의 코드를 추가한다.

```python
cdef extern from 'pcap.h':
    cdef struct pcap_rmtauth:
        pass
    cdef struct pcap_if:
        pass
    int pcap_findalldevs_ex(char* source, pcap_rmtauth* auth, pcap_if** alldevs, char* errbuf)
```

pcap_findalldevs_ex 함수를 사용하기 위한 준비 단계라고 보면 된다. 이 코드를 제대로 컴파일하려면 pcap.h가 어디에 있는지 그리고 최종적으로 어떤 라이브러리를 링크해야 하는지 등을 알려 줘야 한다. 이를 위해서 setup.py도 아래처럼 고친다.

```python
ext_modules = cythonize(
    Extension(name = '_pcap',
        sources = ['_pcap.pyx'],
        libraries = ['wpcap', 'Packet'],
        define_macros = [('WIN32', None)],
        include_dirs = [os.path.join(winpcap_dir, 'Include')],
        library_dirs = [os.path.join(winpcap_dir, 'Lib', 'x64')]),
    )
```

winpcap_dir는 앞서 받은 라이브러리를 푼 디렉토리다. 그리고 'WIN32'라는 매크로를 추가로 정의해 줘야 한다. 기본적으로 정의되는 매크로는 '_WIN32'로 앞에 밑줄이 있는데, pcap.h에서는 밑줄이 없는 WIN32를 사용하고 있으므로 추가해 준다. 여기까지 하고 에러없이 빌드가 되는지를 확인한다.

이제 함수를 호출해 보자!

```c
    pcap_if_t *alldevs;
    char errbuf[PCAP_ERRBUF_SIZE];
    /* Retrieve the device list from the local machine */
    if (pcap_findalldevs_ex(PCAP_SRC_IF_STRING, NULL /* auth is not needed */, &alldevs, errbuf) == -1)
```

C에서는 이렇게 되어 있다. 이를 Cython으로 처리하려면 먼저 `#define`으로 정의된 2개의 상수에 대해 알려주어야 한다. 숫자인 PCAP_ERRBUF_SIZE는 `enum`을 사용하면 되고, 문자열인 PCAP_SRC_IF_STRING는 변수를 선언하듯이 하면 된다.

```cython
cdef extern from 'pcap.h':
    enum:
        PCAP_ERRBUF_SIZE  # 256
    char* PCAP_SRC_IF_STRING  # "rpcap://"
```

그리고 실제 호출은 main 함수 안에서 다음과 같이 한다. 

```cython
def main():
    cdef pcap_if_t* alldevs
    cdef char errbuf[PCAP_ERRBUF_SIZE]
    cdef int result = pcap_findalldevs_ex(PCAP_SRC_IF_STRING, NULL, &alldevs, errbuf)
    if result == -1:
        fprintf(stderr, 'Error in pcap_findalldevs_ex: %s\n', errbuf)
        exit(1)
    printf('result: %d\n', result)
```

주석을 지우고, `cdef`가 추가된 것만 다르지 거의 같다. `NULL`은 Cython이 알아서 한다. printf나 exit의 C 표준 라이브러리를 사용하려면 _pcap.pyx에 다음 코드도 함께 넣어야 한다.

```cython
from libc.stdio cimport printf, fprintf, stderr
from libc.stdlib cimport exit
```

이대로 빌드 하면 PCAP_SRC_IF_STRING가 없다는 에러가 발생한다. PCAP_SRC_IF_STRING는 remote-ext.h 헤드 파일에 정의되어 있는데, HAVE_REMOTE가 정의되어 있는 경우에만 포함된다. 그러므로 에러 없이 컴파일을 하려면 setup.py의 define_macros에 ('HAVE_REMOTE', None)도 추가해야 한다.

여기까지 하고 빌드하고 test.py했을 때 'result 0'이 찍히면 된다.

