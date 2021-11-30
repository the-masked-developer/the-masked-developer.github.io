---
title: Distributed Programming in PyTorch
date: 2021-11-30 22:22:10
toc: true
category: 프로그래밍
tags:
  - 프로그래밍
math: true
author: wayexists02
---

# Distributed Programming in PyTorch

PyTorch는 멀티 GPU 학습을 위해 `torch.nn.DataParallel`을 제공한다. 이 클래스의 사용법은 상당히 간단한데, 모델을 만든 후, `torch.nn.DataParallel`을 씌워주면 된다. 예를들어, 다음과 같다.

```python
class Model(nn.Module):
	def __init__(self):
		# ...
	def forward(self, x):
		# ...

model = Model().cuda()
dmodel = nn.DataParallel(model).cuda()

for e in range(EPOCHS):
	for x, y in trainloader:
		logps = dmodel(x)   # 여기서, model 대신, dmodel을 사용하면 됨
											  # dmodel은 x를 GPU개수만큼 잘라서 각각 돌리고
												# 그 결과를 concat해서 반환해줌
		
		# loss 계산, acc 계산, back-prop 등등 처리
```

이처럼, `Model`이 들어가야 할 자리에 `torch.nn.DataParallel` 객체를 넣어주면 된다. 매우 간편하게 멀티 GPU를 사용할 수 있는 셈이다.

하지만, 이 `torch.nn.DataParallel` 은 치명적인 문제점이 존재한다.

- `torch.nn.DataParallel`은 Thread 기반이다.
    - 아시는분들은 아시겠지만, python의 멀티스레드 프로그래밍에서는 GIL이라는 큰 제약조건이 존재한다.
    
    <aside>
    💡 GIL(Global Interpreter Lock)이란, 파이썬 인터프리터의 구현 모델중 하나로, 파이썬 인터프리터의 동기화 문제 등으로 인해, 동시에 구동될 수 있는 파이썬 스레드 개수를 최대 1개로 제한해버린 구현 모델을 말한다. 즉, 파이썬 코드로 스레드를 많이 만들어서 코드를 구동하고, 컴퓨터에 프로세서가 많다고 해도, 파이썬은 한 번에 하나의 스레드만 돌린다.
    
    </aside>
    

따라서, `torch.nn.DataParallel`을 사용한다고 해서, GPU가 늘어나는 만큼 그 속도가 증가하지는 못한다. PyTorch에는 이러한 문제를 해결하기 위해 `torch.nn.parallel.DistributedDataParallel` 이라는 클래스를 제공한다.

`torch.nn.parallel.DistributedDataParallel`은 스레드기반이 아닌 프로세스 기반이다. 파이썬에서는 `threading`이라는 모듈을 제공하지만, 말했다시피, GIL으로 인해 `threading`의 성능은 그렇게 좋지 못하다. 파이썬에서는 다행히도 GIL을 피할 수 있게 `multiprocessing`이라는 모듈을 standard library로 제공하고 있다.

PyTorch에서는 이 파이썬 `multiprocessing` 모듈처럼, `torch.multiprocessing` 모듈과 `torch.distributed` 모듈을 제공하며, 이는 스레드 기반 분산처리보다 더 나은 분산처리를 구현하는 데 사용될 수 있다. 우리가 여기서 논의할 내용이 바로 PyTorch에서 제공하는 프로세스 기반 분산 프로그래밍이다.

# Distributed Programming in PyTorch

여기서 논의할 내용은 다음과 같다.

1. PyTorch에서의 프로세스 생성 방법
2. PyTorch의 프로세스 그룹과 `init_process_group` 함수
3. PyTorch 프로세스간의 blocking 통신
4. PyTorch 프로세스간의 non-blocking 통신

`torch.nn.parallel.DistributedDataParallel`의 경우, 여기서 다루지는 않고, 나중에 다른 문서에서 다루도록 할 것이다. 본 문서에서는 `torch.nn.parallel.DistributedDataParallel`를 공부하기 이전에 PyTorch에서 어떻게 프로세스 단위로 프로그래밍할 수 있는지 알아볼 것이다.

## Process Creation in PyTorch

PyTorch에서는 `torch.multiprocessing`이라는 모듈을 제공하는데, 파이썬 standard library에서의 `multiprocessing` 모듈과 유사하게 생겼다.

PyTorch에서 프로세스를 생성하는 방법은 다음과 같다.

```python
from torch.multiprocessing import Process

def func(args):
	# ...

args = []                                # func에 넘겨줄 인자 리스트
p = Process(target=func, args=args)      # 프로세스 생성. 이때, 아직,
																				 # 프로세스는 시작하지 않음(func을 구동하지 않음)

p.start()                                # 여기서 비로소 프로세스 시작
p.join()                                 # 프로세스가 끝날때까지 기다리고 자원 회수
```

이 형태는 파이썬 `threading` 라이브러리 사용법과도 매우 유사한데, PyTorch에서 프로세스를 생성하는 방법은 `Process` 객체를 생성하는 것이다. 이때, `Process` 생성자로 넘겨주는 파라미터 중 가장 중요한 두 개의 인자는 바로 `target`과 `args`이다. `target`인자는 해당 프로세스가 생성되고 실행할 함수를 의미하며, `args`는 그 함수에 넘겨줄 파라미터를 받는다. 여기서, `target`은 `func`으로 설정되어 있기 때문에, 프로세스는 시작할 때, `func` 함수를 실행하게 되며, `args`라는 파라미터를 `func`의 파라미터로 넘겨주게 된다.

`Process` 를 통해 프로세스를 생성한다고 해서 프로세스가 바로 시작하지는 않는다. `Process` 객체의 `start` 메소드를 호출해야 해당 프로세스가 비로소 `target`함수를 실행하게 된다.

Linux 시스템 프로그래밍 경험이 있으시다면, `join`함수의 중요성을 잘 알 것이다. 기본적으로 프로세스를 코드상에서 생성하게 되면, 그 프로세스는 코드를 구동하는 프로세스의 자식 프로세스로 생성된다. **그리고, 이것은, 코드를 실행하는 프로세스가 자식 프로세스의 자원을 회수할 의무가 있다는 것을 의마한다.**

코드상에서 `fork`, `Process`와 같은 코드를 사용하여 프로세스를 생성했다면, 코드에서 그 프로세스의 죽음을 지켜보고 그 자원을 회수할 의무가 있다. **그 역할을 하는 함수가 바로 `join`이다.** 만약, 코드를 실행하는 프로세스가 자식을 회수하지 않고 끝나버리면, 그 자식 프로세스는 데몬 프로세스가 되는데, 물론, 요즘 운영체제는 부모가 죽으면, 그 자식의 자원을 자동으로 회수해주긴 하지만, 명시적으로 `join`을 호출해주는 습관을 들이는게 좋다.

## Process Group

PyTorch의 멀티프로세싱에서는 프로세스 그룹이라는 개념이 있다. 프로세스 그룹이란, 말 그대로 여러 프로세스를 묶어 관리하는 것을 말하는데, 프로세스 그룹 안에 속해있는 프로세스들은 peer 프로세스가 몇개가 있고 누가 있는지 알 수 있다. 즉, 그에따라, peer 프로세스와 통신이 가능하다.

프로세스 그룹은 처음볼때는 이해하기가 좀 까다로운데, 프로세스 그룹이 형성되는 과정은 다음과 같다.

1. 루트프로세스에서 여러개의 자식 프로세스를 생성한다.
2. 각 자식 프로세스에서, 마스터(프로세스 중 하나를 마스터로 삼는다)의 주소를 알 수 있게 하는 환경변수를 설정한다.
    - `MASTER_ADDR`: 마스터의 ip주소
    - `MASTER_PORT`: 마스터의 포트번호
3. 개발자는 각 자식 프로세스 코드에서 `init_process_group` 함수를 호출하여, 해당 프로세스가 프로세스 그룹에 속한다는 것을 알려주어야 한다.
    - 즉, `init_process_group`은 해당 프로세스에게, 그룹에 소속되어있다는 사실을 알려주는 함수이다.
    - `init_process_group`이 실행되면, 해당 프로세스는 `MASTER_ADDR`와 `MASTER_PORT`를 사용하여 마스터와 통신을 시도한다.
    - 마스터는 여러 프로세스로부터 통신을 받게 되며, 마스터는 그 프로세스들을 서로 통신할 수 있도록 연결시켜준다.

여기서, 알아두어야 할 점은, `init_process_group`함수는 **프로세스를 생성하지는 않는다**. 프로세스 그룹을 생성하는 역할을 하는 것이다. 또한, **다른 peer 프로세스가 누가 있는지, 알 수 있게 해 주는 함수가 바로 `init_process_group`이며**, 즉, 다른 peer 프로세스의 존재를 알려줌으로써, 나중에 peer 프로세스와 메시지를 주고받을 수 있게 해 주는 함수도 `init_process_group`이다.

### Example of `init_process_group`

다음은 `init_process_group`를 사용하는 예제코드를 보여준다(Reference: [PyTorch 공식홈피](https://pytorch.org/tutorials/intermediate/dist_tuto.html#point-to-point-communication)).

```python
import os
import torch
import torch.distributed as dist
from torch.multiprocessing import Process

def run(rank, size):
	'''
	프로세스가 실행하는 메인 함수
	'''
	# some logic

def init_process(rank, size, fn, backend="gloo"):
	'''
	이 함수는 각 프로세스에서 실행되는 함수이다.
	이 함수는 각 프로세스가 만들어졌을 때, 프로세스 그룹을
	세팅하기 위한 함수로, 세팅이 끝나면, 마지막으로 메인 함수인 run을 호출한다.
	'''

	print(f"I am a {rank} process!")

	# 각 프로세스에서는 MASTER_ADDR와 MASTER_PORT 환경변수를 설정해준다.
	os.environ["MASTER_ADDR"] = "127.0.0.1"
	os.environ["MASTER_PORT"] = "24500"

	# 이 프로세스는 127.0.0.1:24500 을 마스터로 하는 프로세스 그룹에 속한다는 것을
	# 알려준다.
	dist.init_process_group(backend, rank=rank, world_size=size)

	# fn 함수 호출
	fn(rank, size)

if __name__ == "__main__":
	# GPU 개수 얻어오기
	size = torch.cuda.device_count()
	print("Device count:", size)
	
	processes = []

	# GPU개수만큼 프로세스 생성 후 각 프로세스에서 init_process 구동
	for rank in range(size):
		p = Process(target=init_process, args=(rank, size, run))
		p.start()
		processes.append(p)

	# 모든 자식이 죽을때까지 기다렸다가 자원 회수
	for p in processes:
		p.join()
```

각 프로세스가 생성되면, `init_process`라는 함수가 호출되는데, `init_process`는 해당 프로세스의 환경변수를 세팅하고 프로세스 그룹에 포함된다는 사실을 알려주는(즉, `init_process_group`을 호출하는) 역할을 수행하는 함수이다. 이러한 세팅이 끝나면, 곧이어 `run` 함수를 호출하게 되고, `run` 함수에서 로직을 수행하게 되는 것이다.

`init_process_group`함수를 자세히 살펴보자. `init_process_group`은 많은 파라미터를 받을 수 있지만, 이 예제에서는 `backend`, `world_size`, `rank` 세 가지의 파라미터만 사용한다. 다음은 `init_process_group`의 각 파라미터에 대한 설명이다.

- `backend` 파라미터
    - PyTorch에서 프로세스 그룹을 생성한다는 것은 여러 프로세스가 서로 통신하며 분산처리를 하겠다는 것을 의미한다. 따라서, 분산처리 백엔드 라이브러리를 사용해야 하며, `backend` 파라미터는 어떤 백엔드를 사용하여 분산처리를 할 것인지 명시하도록 한다.
    - PyTorch는 scratch부터 low-level부터 high-level까지의 분산처리 시스템을 구현한 것이 아니라, 다른 low-level 분산처리 라이브러리를 가저다 쓰는 형태이다. 즉, 백엔드는 다른 라이브러리를 사용한다. PyTorch에서는 세 가지의 백엔드를 지원한다.
        - **GLOO**: PyTorch에서 가장 잘 지원하는 라이브러리로, CPU 텐서와 GPU 텐서의 분산처리 모두를 지원한다. 다만, GPU 텐서 연산의 경우 NCCL 백엔드보다는 최적화가 덜 되어 있다고 한다.
        - **MPI**: Messaging passing interface를 사용하는 백엔드. 이것을 백엔드로 쓰려면, MPI 라이브러리의 추가적인 설치와 세팅이 필요하다. 자세한것은 [여기](https://pytorch.org/tutorials/intermediate/dist_tuto.html#communication-backends) 참고
        - **NCCL**: GPU 텐서연산에 대해 고도로 최적화된 백엔드이다. GPU에 올라간 텐서만 통신을 지원한다.
- `world_size` 파라미터
    - 해당 프로세스 그룹에 속할 프로세스의 총 개수를 의미하며, 각 프로세스에게 몇 개의 peer 프로세스가 있는지 알 수 있도록 해 주는 인자이다.
    - 프로세스가 마스터와 통신할때, 몇 개의 peer 프로세스가 접속하기를 기다려야 하는지 알려주는 인자이기도 하다. 즉, 프로세스가 `init_process_group`을 호출하게 되면, 마스터와 통신을 시도하고, 마스터는 그 프로세스에게 다른 peer 프로세스를 연결시켜 주는데, `world_size` 파라미터를 명시해주게 되면,  몇 개의 peer 프로세스가 자신과 연결되어야 하는지 알 수 있게 된다(`world_size - 1`개의 프로세스가 자신과 연결되어야 함).
      
        만약, 마스터가 자신에게 `world_size - 1`개의 프로세스를 아직 연결해주지 못했다면(다른 peer 프로세스가 아직 `init_process_group`을 호출하지 못했다거나의 이유로)  해당 프로세스는 마스터와 계속 연결을 유지하면서 peer를 기다린다.
    
- `rank` 파라미터
    - 해당 프로세스의 id값이라고 보면 된다.
    - 프로세스 그룹이 생성될때, 각 프로세스는 rank의 값으로 자신의 id를 설정한다. 이 값은 나중에 다른 프로세스와 통신할때 사용된다.
      
        즉, 3번 rank를 가진 프로세스에게 tensor를 전송하라! 라는 명령이 가능하다.
        

`os.environ["MASTER_ADDR"]`와 `os.environ["MASTER_PORT"]`의 설정, 그리고, `init_process_group`은 각각의 프로세스에서 모두 한번씩 실행되어야 함을 주의해야 한다.

## Blocking Communications in PyTorch

`init_process_group`을 이용하여 각각의 프로세스에게 peer 프로세스가 누가 있는지 알려줄 수 있었다. `init_process_group`을 통해 peer 프로세스가 누군지 알 수 있게 되면, peer 프로세스의 rank 번호를 이용하여 peer 프로세스와 통신이 가능하다.

통신에는 blocking 통신과 non-blocking 통신이 존재하는데, 먼저, blocking 통신에 대해 논의해보도록 한다.

PyTorch에서의 다른 프로세스와의 통신은, 텐서를 주고받는 것이다. 텐서를 보내는 함수로는 `torch.distributed.send`가 있고, 텐서를 받는 함수로는 `torch.distributed.recv`가 있다.

`torch.distributed.send`를 호출하게 되면, 상대방 프로세스가 `recv`를 호출하여 받을 때까지 blocking된다. 마찬가지로, `torch.distributed.recv`를 호출하게 되면, 상대방 프로세스가 `send`를 통해 무언가를 보낼때 까지 blocking된다. 따라서, 타이밍을 잘 맞춰서 사용하도록 하자.

### Example of Blocking Communication in PyTorch

다음은 init_process_group의 예제에다가 blocking communication을 구현한 예제이다.

```python
import os
# GPU개수는 2개라고 가정하자
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1"

import torch
import torch.distributed as dist
from torch.multiprocessing import Process

def run(rank, size):
	'''
	프로세스가 실행하는 메인 함수
	'''
	tensor = torch.FloatTensor([0.0])
	if rank == 1:
		tensor += 1
		dist.send(tensor=tensor, dst=0)
	elif rank == 0:
		dist.recv(tensor=tensor, src=1)
	print("Rank:", rank, ", Tensor:", tensor[0])

def init_process(rank, size, fn, backend="gloo"):
	'''
	이 함수는 각 프로세스에서 실행되는 함수이다.
	이 함수는 각 프로세스가 만들어졌을 때, 프로세스 그룹을
	세팅하기 위한 함수로, 세팅이 끝나면, 마지막으로 메인 함수인 run을 호출한다.
	'''

	print(f"I am a {rank} process!")

	# 각 프로세스에서는 MASTER_ADDR와 MASTER_PORT 환경변수를 설정해준다.
	os.environ["MASTER_ADDR"] = "127.0.0.1"
	os.environ["MASTER_PORT"] = "24500"

	# 이 프로세스는 127.0.0.1:24500 을 마스터로 하는 프로세스 그룹에 속한다는 것을
	# 알려준다.
	dist.init_process_group(backend, rank=rank, world_size=size)

	# fn 함수 호출
	fn(rank, size)

if __name__ == "__main__":
	# GPU 개수 얻어오기
	size = torch.cuda.device_count()
	print("Device count:", size)
	
	processes = []

	# GPU개수만큼 프로세스 생성 후 각 프로세스에서 init_process 구동
	for rank in range(size):
		p = Process(target=init_process, args=(rank, size, run))
		p.start()
		processes.append(p)

	# 모든 자식이 죽을때까지 기다렸다가 자원 회수
	for p in processes:
		p.join()
```

프로세스의 생성과 `init_process_group`까지는 이전 예제와 같다. 달라진점은 `run` 함수 내에 다른 peer 프로세스와 통신하는 코드를 넣었다는 점이다.

말했다시피, 통신은 `send`함수와 `recv`함수를 사용한다. send함수는 `tensor` 객체를 `dst` 프로세스에게 보내는데, `dst`에 들어갈 값은 메시지를 받을 프로세스의 rank값이다.

`recv`함수 역시, 비슷하게 사용하면 되는데, `src` 프로세스로부터 어떤 텐서를 받아서 `tensor` 객체에 저장한다. 이때, `tensor`는 미리 생성해두어야 하며, `recv`는 받은 텐서객체를 `tensor`에 덮어씌우게 된다.

실행결과, 텐서값은 두 프로세스 모두 1이 된다. 0번 프로세스의 경우, 직접 `tensor += 1`을 해주었고, 1번 프로세스의 경우, 0번으로부터 받은 텐서로 덮어씌웠기 때문이다.

## Non-blocking Communication in PyTorch

PyTorch에서 process간 통신은 blocking과 non-blocking방식 모두 존재한다고 말했었다. PyTorch는 `torch.distributed.send`와 `torch.distributed.recv`라는 blocking 통신을 제공했는데, 이와 비슷하게 `torch.distributed.isend`, `torch.distributed.irecv`라는 non-blocking 통신을 제공한다.

`torch.distributed.isend`를 사용하여 텐서를 그룹 내 어떤 프로세스에게 전송하게 되면, 그 텐서를 수신자가 받을 때까지 기다리지 않고, 바로 리턴되는데, 이때, `torch.distributed.isend`는 request 객체를 반환한다(`torch.distributed.send`는 아무것도 반환하지 않음). 메시지가 전달되었는지에 대한 여부는 request 객체의 `wait` 메소드를 호출하면 알 수 있다. `wait` 메소드를 호출했을 때, blocking되면 아직 전송이 안된 것이고, 리턴되면 전송된 것이다.

마찬가지로, `torch.distributed.irecv`를 호출하게 되면, 텐서가 올때까지 기다리는 게 아니라 request객체를 바로 반환하게 된다(백그라운드에서 송신자로부터 텐서를 계속 기다리고 있음). 텐서가 왔는지 확인하려면 request 객체의 `wait` 메소드를 호출하면 된다. 역시, blocking되면, 아직 텐서를 못받은 것이고, 리턴되면 텐서를 받은 것이다.

### Example of Non-blocking Communication in PyTorch

```python
import torch
import torch.distributed as dist
from torch.multiprocessing import Process

def run(rank, world_size):
  '''
  프로세스가 실행하는 메인 함수
  '''
  tensor = torch.FloatTensor([0.0])
  if rank == 0:
    tensor += 1
    req = dist.isend(tensor=tensor, dst=1)
  else:
    req = dist.irecv(tensor=tensor, src=0)
  req.wait()
  print("Rank:", rank, ", Tensor:", tensor[0])

def init_process(rank, world_size, fn, backend="gloo"):
  '''
  이 함수는 각 프로세스에서 실행되는 함수이다.
  이 함수는 각 프로세스가 만들어졌을 때, 프로세스 그룹을
  세팅하기 위한 함수로, 세팅이 끝나면, 마지막으로 메인 함수인 run을 호출한다.
  '''

  print(f"I am a {rank} process!")

  # 각 프로세스에서는 MASTER_ADDR와 MASTER_PORT 환경변수를 설정해준다.
  os.environ["MASTER_ADDR"] = "127.0.0.1"
  os.environ["MASTER_PORT"] = "24500"

  # 이 프로세스는 127.0.0.1:24500 을 마스터로 하는 프로세스 그룹에 속한다는 것을
  # 알려준다.
  dist.init_process_group(backend, rank=rank, world_size=world_size)

  # fn 함수 호출
  fn(rank, world_size)

if __name__ == "__main__":
  # GPU 개수 얻어오기
  world_size = torch.cuda.device_count()

  processes = []

  # GPU개수만큼 프로세스 생성 후 각 프로세스에서 init_process 구동
  for rank in range(world_size):
    p = Process(target=init_process, args=(rank, world_size, run))
    p.start()
    processes.append(p)

  # 모든 자식이 죽을때까지 기다렸다가 자원 회수
  for p in processes:
    p.join()
```

## Reduce & Broadcast

`dist.recv`, `dist.send`, `dist.irecv`, `dist.isend` 처럼 텐서 단위를 직접 주고받는 연산도 있지만, 각 프로세스에 있는 텐서를 자동으로 모아서 합쳐준다거나, 하나의 텐서를 모든 프로세스에게 자동으로 뿌려주는 연산도 있다. 전자는 `reduce`, 후자는 `broadcast`로써 PyTorch에서 함수로 지원한다.

### Reduce

Reduce는 각 프로세스에 있는 텐서를 모아서 aggregation해주는 함수로, `dist.reduce`로 존재한다. 다음 예시 코드를 보자.

```python
import torch
from torch.utils.data.distributed import DistributedSampler
from torch import distributed as dist
from torch import multiprocessing as mp
import time
import os

WORLD_SIZE = 4

def setup(rank, world_size):
    os.environ["MASTER_ADDR"] = "127.0.0.1"
    os.environ["MASTER_PORT"] = "25555"
    os.environ["RANK"] = f"{rank}"
    os.environ["WORLD_SIZE"] = f"{world_size}"
    
    dist.init_process_group("gloo", rank=rank, world_size=world_size)

def run(rank, world_size):
    setup(rank, world_size)

    while True:
        tensor = torch.randn(2, 2)

				# 4개의 머신에서 tensor를 모두 0번 머신으로 보내고,
				# 0번 머신은 tensor를 모두 모아서 SUM한 값을 tensor에 대입한다.
        dist.reduce(tensor, 0, dist.ReduceOp.SUM)
        print(tensor)

        time.sleep(1)
        

if __name__ == "__main__":
		mp.spawn(run, args=(WORLD_SIZE,), nprocs=WORLD_SIZE, join=True)
```

이 코드는 1초마다 4개의 프로세스에서 (2, 2) 크기의 랜덤 텐서를 생성하고 텐서를 0번 텐서에 모두 모아서 덧셈하는 것을 수행하는 코드이다.

각 머신이 `dist.reduce(tensor, 0, dist.ReduceOp.SUM)`라는 코드를 만났을 때 취하는 행동은 다음과 같다.

- 자신이 0번 프로세스인 경우
  
    다른 프로세스로부터 값을 모으고 모두 SUM해서 `tensor`에 저장한다.
    
- 자신이 0번 프로세스가 아닌 경우
  
    0번 프로세스에게 `tensor`값을 전송한다.
    

위 코드에서, 0번 머신은 `tensor`값이 변하지만, 나머지 프로세스는 `tensor`값이 변하지 않는다. 0번 머신은 다른 머신이 보낸 텐서를 모두 모아서 더하고 그 값을 `tensor`로 저장했지만, 나머지 머신은 그저 0번 머신에게 전송만 했기 때문이다.

### Broadcast

Broadcast는 PyTorch에서 `dist.boardcast`로 제공되는 기능으로, 하나의 프로세스에서 다른 프로세스로 텐서를 뿌려주는(전파시키는) 역할을 하는 함수이다. 다음 예제를 보자.

```python
from torch import distributed as dist

def average_gradient(model, world_size):
		for param in model.parameters():
				# gradient를 0번 머신이 모아서 평균냄
				dist.reduce(param.grad.data, 0, dist.ReduceOp.SUM)
				param.grad.data /= world_size

				# 0번 머신은 gradient를 모두에게 전송함
				dist.broadcast(param.grad.data, 0)
```

이 예제에서, `dist.broadcast`는 0번 머신이 자신의 데이터(gradient)를 모든 머신에게 전송하는 역할을 한다.

각 프로세스가 `dist.broadcast(tensor, 0)`라는 코드를 만났을때, 수행하는 행동은 다음과 같다.

- 자신이 0번 프로세스인 경우
  
    자신의 `tensor`데이터를 모두에게 전송한다.
    
- 자신이 0번 프로세스가 아닌 경우,
  
    0번 프로세스로부터 데이터를 받을때까지 기다리고, 데이터를 받으면 `tensor`에 저장한다.
    

## Machine-to-Machine Communication

PyTorch 통신은 프로세스끼리도 가능하지만 다른 컴퓨터끼리도 가능하다. 해답은 `MASTER_ADDR`를 다른 컴퓨터 IP 주소로 설정하는 것이다.

```python
def setup(rank, world_size):
    os.environ["MASTER_ADDR"] = "???.???.???.???"
    os.environ["MASTER_PORT"] = "25555"
    os.environ["RANK"] = f"{rank}"
    os.environ["WORLD_SIZE"] = f"{world_size}"
    
    dist.init_process_group("nccl", rank=rank, world_size=world_size)
```

MASTER_ADDR에 들어가는 주소는 다른 컴퓨터가 인지 가능한 주소여야 하며, **마스터 자신또한 자신이 해당 주소라는 것을 알 수 있어야 한다.**

"마스터 자신또한 자신이 해당 주소라는 것을 알 수 있어야 한다"는 점도 상당히 중요한데, 만약, GCP(Google Cloud Platform)와 같은 서비스를 이용해서 분산 처리 시스템을 구현한다면, 이게 문제가 될 수 있다.

만약, GCP의 같은 계정, 같은 region 안에 존재하는 서버끼리는(또는 같은 VPC 네트워크 내에 존재하는 서버끼리는) 문제없이 통신되지만, 다른 GCP 계정에 존재하는 서버와 PyTorch 통신을 하려는 경우, 이게 문제가 생긴다.

`dist.init_process_group`에서는 외부 IP로 연결이 가능하지만, 그 이후에 각 머신이 서로 통신하는 과정은 각 머신이 알고 있는 자신의 IP로 통신을 시도한다. 근데, GCP VM은 자신의 외부 IP를 모른다(ifconfig 입력해보면 자신의 외부 IP가 나오지 않음). 이것 때문에, `dist.init_proces_group`은 통과하지만, 그 이후 통신에서 connect가 안되는 문제가 발생할 수 있다.

만약, 본인 계정에서 머신 여러개를 만들고 통신하는건 문제가 되지 않는다. 하지만, 다른 VPC 네트워크에 존재하는 VM으로의 통신은 필자가 알기로는 방법이 없어 보인다. 따라서, 다른 VPC 네트워크에 속한 VM과 통신하고자 한다면, PyTorch 통신 이외에 다른 통신법을 찾아보기를 권한다.