---
date: 2024-06-24 
categories:
  - Computer Science
---

# OpenMP

OpenMP를 사용하면 병렬 프로그래밍 코드를 빠르고 직관적으로 작성할 수 있다.

<!-- more -->

## Parallel construct

```cpp
#pragma omp parallel
{
	// enclosing block is parallel
}
```

`parallel` 블록 안에 작성된 코드는 `N` 개의 쓰레드가 동시에 병렬적으로 실행한다. 쓰레드 개수는 실행 환경에 따라 런타임에서 결정되지만 `num_threads(N)` 구문을 추가하면 임의로 정할 수 있다.

## Loop construct

```cpp
#pragma omp parallel num_threads(5)
#pragma omp for
for (int i = 0; i < omp_get_num_threads(); ++i) {
	// loop 블록을 실행중인 쓰레드의 id를 가져옴.
	// 각 순회(iteration)를 처리하는 쓰레드 순서는 비결정적이다
	int tid = omp_get_thread_num();
	std::cout << "Hello from thread #" << tid << endl;
}

// Output:
// Hello from thread #3
// Hello from thread #1
// Hello from thread #2
// Hello from thread #0
// Hello from thread #4
```

`#pragma omp for` 를 사용하면 for 반복문 내부에서 수행될 작업 전체를 각 쓰레드가 균등하게 맡아 처리하도록 OpenMP가 자동으로 처리한다.

### Caveat

for 블록을 진입하면 OpenMP는 “존재하는 모든 쓰레드가 idle 상태에 있다” 가정하고 반복문 작업을 분배한다. 만일 같은 parallel 블록 안에서 다른 작업을 수행하는 쓰레드가 존재하면 해당 작업이 끝나기 전까지 for 블록 작업이 종료되지 못하고 지체되는 문제가 발생하는데, 이 경우 for construct를 사용하지 않고 작업을 직접 배분하도록 하자.

> Each work-distribution region must be encountered by all threads
> in the binding thread set or by none at all unless cancellation
> has been requested for the innermost enclosing parallel region.
> 
> - OpenMP Specification chapter 10
    

```cpp
#pragma omp parallel
{
	#pragma omp single nowait
	{
		bool finish = false;
		
		while (!finish) {
			// 무거운 작업 A를 수행하는 쓰레드 1개...
			finish = work_A();
		}
	}
	
	#pragma omp for
	for (int i = 0; i < 100; ++i) {
		// 혼자서 빡세게 일하는 쓰레드 1개는 nowait 구문이 있음에도 불구하고
		// 작업 A가 끝나기 전까지 이쪽 반복문 작업을 수행할 수 없다
	}
}
```
