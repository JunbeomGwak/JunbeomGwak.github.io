# 0ctf babyheap(unsorted bin attack)

### 보호기법

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled.png)

Full RELRO - Got Overwrite가 불가능

Canary found - Canary 존재

NX enabled - stack이나, data영역의 실행을 막음 → 쉘코드 실행불가

PIE enabled - 매번 주소가 바뀜 → offset으로만 보임

### Unsorted bin attack

힙에서는 fastbin, smallbin, unsorted bin, large bin등으로 chunk를 관리한다.

unsorted bin은 malloc으로 인해 등록된 chunk가 다시 재할당될 때 리스트에서 삭제하면서 발생한다.

이때 Unsotred bin attack을 사용하기 위한 조건은

1. Unsorted bin을 생성할 수 있어야 한다.
2. Unsorted bin에 등록된 Free chunk의 값을 변경할 수 있어야 한다
3. Free Chunk와 동일한 크기의 chunk을 요청할 수 있어야 한다.

출처 - 

[[heap exploit] - Unsorted bin Attack](https://rninche01.tistory.com/entry/heap-exploit-Unsorted-bin-Attack)

Unsorted bin은 1개의 bin이 존재하며 double-linked list로 관리되는 bin이다. 다른 bin들과는 달리 Free 
Chunk의 크기에 상관없이 등록되며 large bin, small bin에 들어가기 전에 먼저 해당 bin에 등록된다.

이후에 malloc요청할 경우 fast bin, small bin, large bin에서 알맞은 크기의 Free Chunk를 찾지 못하면 Unsorted bin에서 사용 가능한 Chunk가 있는지 찾는다.

이 때, Unsorted bin attack은 Unsorted bin에서 사용가능한 Free Chunk를 찾고 Free Chunk를 재할당하기 위해 Unsorted bin list에서 제거하면서 발생한다

[[heap exploit] - Unsorted bin Attack](https://rninche01.tistory.com/entry/heap-exploit-Unsorted-bin-Attack)

### 공격 과정

우선 PIE가 걸려있으므로 symbol을 통해 함수의 주소값을 찾는건 불가능하다. 따라서 다른 방법으로 main함수의 주소를 찾아야 한다. Jsec님의 블로그를 참조하였다.

[10. GDB로 PIE 디버깅](https://m.blog.naver.com/PostView.nhn?blogId=yjw_sz&logNo=221460400687&referrerCode=0&searchKeyword=PIE)

main 주소 = 0x55555555511d

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%201.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%201.png)

main함수

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%202.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%202.png)

Allocate = 0x555555554d48

Fill = 0x555555554e7f

Free = 0x555555554f50

Dump = 0x555555555051

총 4번의 fastbin 사이즈를 할당하였다.(0x20)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%203.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%203.png)

32 = 0x20이지만, size는 0x30인데 이는 헤더정보나 여러 정보(0x10)을 더한 크기가 된다. 따라서 0x20사이즈를 할당한다고 하여도 추가적인 정보(0x10)으로 인해 실제로 할당한 크기보다 더 큰 사이즈가 할당이 된다.

small bin을 2개 할당(0x90) → 총 2개를 추가한 이유는, free시 top chunk와 인접한 청크를 같이 병합하는데 이때 원하는 청크가 top chunk와 합병되는 것을 막기 위해서

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%204.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%204.png)

unsorted bin을 해제 하게되면 fd와 bk에 특정한 값이 저장되는데, 이는 main_arena+88의 값이 저장이 된다. 이를 출력할수만 있다면, libc_leak을 할 수 있을 것이다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%205.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%205.png)

이를 출력하기 위해 Free chunk의 특성을 이용할 것이다.

Index 2 → Index 1의 순서대로 Free를 한 상태이다. Index 1의 fd에 Index 2의 주소가 들어가 있다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%206.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%206.png)

이때 Fill 함수를 이용, 인덱스 0에서 overflow를 일으켜 fd에 들어있는 값을 변경하여 인덱스 1의 fd값을 small bin인 인덱스 4로 변경하고,(0x5555557570c0), 인덱스 4의 size를 fastbin size로 변경한다.

그리고 Allocate를 이용하여 2번 할당을 받으면(fastbin dup) free된 1번 인덱스에 할당되고 다음 fd가 가리키는 인덱스2에 할당될 것이다. 

그럼 실행을 해보자.

0x20 크기의 힙을 4개 할당(fastbin), 0x90크기의 힙을 2개할당(small bin)후 인덱스 2를 free한 후 그 다음 인덱스 1을 free한 후의 힙 상태

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%207.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%207.png)

현재는 인덱스 1이 인덱스 2를 가리키고 있다.

Fd(forward pointer) = 해제된 다음 청크를 가리킨다. 즉 2 → 1로 해제가 되었으면, 1의 fd는 해제된 1의 다음 청크인 2를 가리키게 된다.

이때 동일한 크기(fastbin, 0x20)의 힙을 2개를 할당 받으면 어떻게 되는지 보자.

인덱스 1에 힙이 할당이 되고, 다음 fd가 가리키는 청크(index 2, 0x565266881060)에 할당이 되는 것을 볼 수 있다.

이때 할당을 하지 않고 인덱스를 2 →1로 해제 한 후, Fill을 이용하여 인덱스1의 fd를 4의 인덱스로 변경하고(c0), size를 fastbin의 크기로 변경해준다. → size를 fastbin의 크기로 변경해주는 이유는 할당할때 사이즈를 검사하는데, 이를 우회하기 위해서 size를 변경시켜준다.

성공적으로 잘 변경이 되었다.

또한 0x90(small size)청크를 두개 할당해 주었는데 하나만 있는 이유는 top chunk에 의해 top chunk와 인접한 청크는 같이 병합되어 사라짐

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%208.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%208.png)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%209.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%209.png)

하지만 왜 변경을 하였는데 index 4의 청크가 Freed되었는지는 잘 모르겠다. → 이점에 대해서는 더 연구하고 질문해볼 필요가 있는거 같다.

다음은 fastbin 크기만큼 다시 힙을 재할당 해준다.(0x20 * 2)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2010.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2010.png)

하지만 변경후, 에러가 생기게 되는데, 이는 원래 c0을 0x90만큼의 사이즈로 할당하였지만 0x31(0x30+패리티비트)로 바꾸어 줬기 때문에 에러가 생기게 된다.

따라서 이 값을 0x91로 변경 해주면, 에러가 사라질 것이다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2011.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2011.png)

또한 prev_use가 1이 되었기 때문에? Dump함수가 정상적으로 작동하고, Small bin인덱스인 4(c0)을 free 하면 fd와 bk에 main_arena+88의 주소가 남게 되고, 이를 Dump를 통해 출력하게 된다면, libc_leak가 가능해 질것이다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2012.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2012.png)

### Libc_leak

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2013.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2013.png)

 위와 같이 main_arena+88의 주소를 구할 수 있다. 

그리고 main_arena의 offset을 구하기 위해 해당 프로그램을 사용하여 main_arena_offset의 값을 구하였고, libc_leak을 하려면 main_arena+88 - main_arena_offset - 88을 한다면 libc_leak을 할 수 있을 것이다.

[bash-c/main_arena_offset](https://github.com/bash-c/main_arena_offset)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2014.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2014.png)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2015.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2015.png)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2016.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2016.png)

짜잔~

이제 쉘을 따는 방법만 남았다. 

쉘을 따는 방법은 fastbin duplication을 이용하고, __malloc_hook을 이용하여 쉘을 획득할 것이다..

청크의 fd를 원하는 주소로 조작하고, malloc이 실행될때 쉘을 따는 식으로 구성

### __malloc_hook?

프로그램이 malloc을 부르게 되면 C 라이브러리 malloc이 사용자가 등록한 hook 함수가 있는지 조사하고 hook가 있을 경우, 그 hook 함수를 대신 부릅니다. 물론 이 hook 함수는 원래의 malloc을 호출할 수 있습니다.

```c
void * malloc_hook(size_t SIZE, const void *CALLER)
{}
```

저 caller에 함수가 있으면 malloc이 호출될때 Caller함수가 실행이 된다.

그래서 저 Caller부분에 one_shot가젯+libc_base 주소를 넣고 malloc을 한다면 쉘이 따질것이다.

우선 malloc_hook의 offset을 구하면 다음과 같다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2017.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2017.png)

우리가 malloc을 이용하기 위해서는 fake chunk를 이용하여 malloc을 해야한다.

그러기 위해서는 fake chunk를 위한 주소를 찾아야 하는데, 

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2018.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2018.png)

__malloc_hook에서 0x23을 뺀 부분을 보면 0x7f부분이 있는데, 이부분을 fake chunk의 size로 사용한다면 힙이 잘 할당될것이다.

그러면 힙을 하나 할당하여 caed부분을 fake chunk의 시작주소로 잡고, 0x7f를 size로 하는 fake chunk를 생성하여 쉘을 따면 될 것이다.

이때 주의해야할 점은 malloc을 할때 size는 할당을 요청한 size보다 더 큰 사이즈를 할당하는데, 이는 할당한 사이즈 + 헤더의크기가 있기 때문이다

0x7f - 0x10 -1 = 104이다. 이때 1을 더 빼주는 이유는 prev_use가 1이 되어야 Dump 함수가 제대로 작동하기 때문

그럼 104크기의 힙을 할당하고 다시 free해준다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2019.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2019.png)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2020.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2020.png)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2021.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2021.png)

3의 인덱스에서 overflow를 이용(Fill)함수, 인덱스 4의 fd값을 fake_chunk의 값으로 변경해준다.

첫번째로 힙를 할당하면, fastbins에 free_chunk의 값이 들어간다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2022.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2022.png)

fastbin은 LIFO 구조이다. 맨 마지막으로 들어간 것이 맨 처음으로 나옴

두번째로 힙을 할당하면 index 6에 들어가는 것을 볼 수 있다. 이때 Index 6은 fake_chunk의 위치를 가리키고 있다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2023.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2023.png)

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2024.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2024.png)

이때 Fill을 하게되면, free chunk의 값에서 __malloc_hook까지의 값을 덮어씌울수 있다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2025.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2025.png)

```python
Fill(6, 27, "A"*3+p64(0x61)*2+p64(one_gadget[1]+libc_base))
```

_IO_wide_data_0+288의 주소가 fake chunk의 주소가 되고, +296이 size, 304부터는 data의 영역이 된다. free_chunk의 주소는 wide_data + 301이다.

그럼 __malloc_hook의 주소가 one_gadget의 주소로 덮였을테니, Allocate를 통해 힙을 한번더 재할당하게되면 hook으로 인해 one_gadget가 실행될 것이다.

![0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2026.png](0ctf%20babyheap(unsorted%20bin%20attack)%2025025c28746b430e84eecb8051b3e909/Untitled%2026.png)

### 후기

힙에 대해서 잘 몰랐기 때문에 write-up을 보고 실습하고 다시 풀어보았다.. 역시 어렵다

우선 힙에 대해서 더 공부를 해야겠다. fastbin dup, unsotred bin attack등등 기본 지식을 더 익혀야겠다는 생각이 들었으며 이와 관련된 문제(Hitcon training 12)도 풀어서 풀이를 남겨야겠다.

또한 0x7f부분을 size로 한다는게 정말 획기적이였던거같다.