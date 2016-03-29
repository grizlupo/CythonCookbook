# Cython�� ����ؼ� winpcap ����ϱ�

winpcap�� libpcap�� �������� ���̰�, �̰� ���̽㿡�� ����� �� �ֵ��� ���ִ� ��⵵ �̹� �ִ�. �׷����� ���� ����� ����� ������� ���� Cython�� �����غ����� ���� ���� ũ��, �� ������ libpcap�� �ͼ��������� ���̴�. �� ���� 64��Ʈ �������� 10���� ���̽��� 3.5, Cython�� 0.23.4�� ����ϴ� ȯ�濡�� ���� �ִ�.

�켱�� winpcap�� �ʿ��ϴ�. [http://www.winpcap.org/install/default.htm](http://www.winpcap.org/install/default.htm/)���� ���� �� �ִ�. �� ���� ���� �ִ� ���� �ֽ� ������ 4.1.3�̴�. ���̺귯���� ì�ܾ� �Ѵ�. [http://www.winpcap.org/devel.htm](http://www.winpcap.org/devel.htm/)���� ���� �� �ִ�. ������ 4.1.2��. 4.1.3�� ����ص� �ȴٰ� �Ǿ� �ִ�. winpcap�� ��ġ�ϰ�, ���̺귯���� ������ ���� ������ Ǭ��.

## 1 �ܰ�

���� libpcap ��ü�� ������ �����ϱ� �ͼ������� �������� winpcap ������ �ִ� �н���tutorial�� Cython���� ������ �� ���̴�.

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

### main �Լ� ������ �����

�ϴ��� ��ü���� ó���� ��� ����ΰ�, ���̽㿡�� ������ ���̴��� main�� ȣ���ϵ��� �� ����! �̰� libpcap���ٴ� Cython�� ���� ���̴�.

_pcap.pyx��� ������ �����. ������

```cython
def main():
    print('pcap')
```

�׸��� �̰� ������ ���ϰ�, �׽�Ʈ�� ������ �����.

��������δ� �Ʒ� �������� setup.py ������ �����.

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

�׽�Ʈ�� �Ʒ� �������� test.py�� �����.

```python
import _pcap

_pcap.main()
```

�׸��� ���� ����� �Ʒ� ����� ��� �ؾ� �ϴµ� �������ϱ� ��ġ������ �ϳ� �����. build.bat��.
 
```sh
py -3 setup.py build_ext --inplace
```

����� �Ǿ��ٸ� _pcap.cp35-win_amd64.pyd�� ���������. �׸��� test.py�� ���� �� ȭ�鿡 'pcap'�� ������ �ϴ� ù ���� ���� ���̴�.

### pcap_findalldevs_ex �Լ� ȣ���ϱ� 

���� _pcap.pyx�� �Ʒ��� �ڵ带 �߰��Ѵ�.

```python
cdef extern from 'pcap.h':
    cdef struct pcap_rmtauth:
        pass
    cdef struct pcap_if:
        pass
    int pcap_findalldevs_ex(char* source, pcap_rmtauth* auth, pcap_if** alldevs, char* errbuf)
```

pcap_findalldevs_ex �Լ��� ����ϱ� ���� �غ� �ܰ��� ���� �ȴ�. �� �ڵ带 ����� �������Ϸ��� pcap.h�� ��� �ִ��� �׸��� ���������� � ���̺귯���� ��ũ�ؾ� �ϴ��� ���� �˷� ��� �Ѵ�. �̸� ���ؼ� setup.py�� �Ʒ�ó�� ��ģ��.

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

winpcap_dir�� �ռ� ���� ���̺귯���� Ǭ ���丮��. �׸��� 'WIN32'��� ��ũ�θ� �߰��� ������ ��� �Ѵ�. �⺻������ ���ǵǴ� ��ũ�δ� '_WIN32'�� �տ� ������ �ִµ�, pcap.h������ ������ ���� WIN32�� ����ϰ� �����Ƿ� �߰��� �ش�. ������� �ϰ� �������� ���尡 �Ǵ����� Ȯ���Ѵ�.

���� �Լ��� ȣ���� ����!

```c
    pcap_if_t *alldevs;
    char errbuf[PCAP_ERRBUF_SIZE];
    /* Retrieve the device list from the local machine */
    if (pcap_findalldevs_ex(PCAP_SRC_IF_STRING, NULL /* auth is not needed */, &alldevs, errbuf) == -1)
```

C������ �̷��� �Ǿ� �ִ�. �̸� Cython���� ó���Ϸ��� ���� `#define`���� ���ǵ� 2���� ����� ���� �˷��־�� �Ѵ�. ������ PCAP_ERRBUF_SIZE�� `enum`�� ����ϸ� �ǰ�, ���ڿ��� PCAP_SRC_IF_STRING�� ������ �����ϵ��� �ϸ� �ȴ�.

```cython
cdef extern from 'pcap.h':
    enum:
        PCAP_ERRBUF_SIZE  # 256
    char* PCAP_SRC_IF_STRING  # "rpcap://"
```

�׸��� ���� ȣ���� main �Լ� �ȿ��� ������ ���� �Ѵ�. 

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

�ּ��� �����, `cdef`�� �߰��� �͸� �ٸ��� ���� ����. `NULL`�� Cython�� �˾Ƽ� �Ѵ�. printf�� exit�� C ǥ�� ���̺귯���� ����Ϸ��� _pcap.pyx�� ���� �ڵ嵵 �Բ� �־�� �Ѵ�.

```cython
from libc.stdio cimport printf, fprintf, stderr
from libc.stdlib cimport exit
```

�̴�� ���� �ϸ� PCAP_SRC_IF_STRING�� ���ٴ� ������ �߻��Ѵ�. PCAP_SRC_IF_STRING�� remote-ext.h ��� ���Ͽ� ���ǵǾ� �ִµ�, HAVE_REMOTE�� ���ǵǾ� �ִ� ��쿡�� ���Եȴ�. �׷��Ƿ� ���� ���� �������� �Ϸ��� setup.py�� define_macros�� ('HAVE_REMOTE', None)�� �߰��ؾ� �Ѵ�.

������� �ϰ� �����ϰ� test.py���� �� 'result 0'�� ������ �ȴ�.

