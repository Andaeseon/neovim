##  소스 코드 개요
 
 
Neovim은 Vim에서 대부분의 코드를 상속 받았기 때문에 [README (https://raw.githubusercontent.com/vim/vim/master/src/README.txt)]의 
일부 정보가 계속 적용됩니다.

이 문서는 Vim과 다른 부분에 초점을 맞춰 Neovim이 내부적으로 어떻게 작동하는지에 대한 수준 높은 개요를 제공하는 것을 목표로 합니다.
특히 끊임없이 변화하는 부품에 대한 세부 정보를 아직 추가하지 않았기 때문에 현재 이 작업은 여전히 진행 중입니다. 
코드가 보다 체계적이고 안정적이 되면, 이 문서는 변경 사항을 반영하여 업데이트 될 것입니다.

각 모듈의 세부 정보를 찾으려면 소스 코드를 읽는 것이 가장 좋습니다. 
일부 파일은 상단에 댓글을 달았습니다 (예 : terminal.c, screen.c)

###  최상위 프로그램 루프

먼저 일반적인 편집 흐름을 분석하여 Vim과 유사한 프로그램이 하는 일을 이해해 보겠습니다.

01. Vim이 환영 화면을 보냅니다.
02. 사용자 유형 : `:`
03. Vim이 명령 행 모드로 들어갑니다. 
04. 사용자 유형 : `edit README.txt<CR>`
05. Vim이 파일을 열고 정상 모드로 돌아갑니다.
06. 사용자 유형 : `G`
07. Vim이 파일 끝으로 이동합니다.
08. 사용자 유형 : `5`
09. Vim이 count-pending 모드로 들어갑니다.
10. 사용자 유형 : `d`
11. Vim이 operator-pending 모드로 들어갑니다.
12. 사용자 유형 : `w`
13. Vim이 5개의 단어를 삭제합니다.
14. 사용자 유형 : `g`
15. Vim이 "g command" 모드로 들어갑니다.
16. 사용자 유형 : `g`
17. Vim이 파일의 시작 부분으로 갑니다.
18. 사용자 유형 : `i`
19. Vim이 insert 모드로 들어갑니다.
20. 사용자 유형 : `word<ESC>`
21. Vim이 처음에 "단어"를 삽입하고 정상 모드로 돌아갑니다.


우리는 사용자의 행동을 편집자의 상태를 변경하는 일련의 입력으로 분할했습니다.
"g 명령 모드"(16 단계)에 대한 문서는 없지만 내부적으로 "연산자 보류 모드"와 비슷하게 구현됩니다.

이것으로부터 우리는 Vim이 입력 구동 상태 머신
(좀 더 구체적으로 말하자면 pushdown automaton은 상태에서 다시 전환하기 위해 스택이 필요하기 때문입니다.)
의 동작을 가지고 있음을 볼 수 있습니다.

각 상태가 키를 처리하는 콜백을 담당한다고 가정하면 
이 의사 코드(파이썬과 비슷한 언어)는 기본 프로그램 루프를 잘 보여줍니다.

```py
def state_enter(state_callback, data):
  do
    key = readkey()                 # read a key from the user
  while state_callback(data, key)   # invoke the callback for the current state
```

즉, 각 상태는`state_enter`를 호출하고 상태 별 콜백과 데이터를 전달하여 입력됩니다.
위에서 설명한 흐름과 같은 것을 구현하는 프로그램의 상위 의사 코드는 다음과 같습니다.

```py
def main()
  state_enter(normal_state, {}):

def normal_state(data, key):
  if key == ':':
    state_enter(command_line_state, {})
  elif key == 'i':
    state_enter(insert_state, {})
  elif key == 'd':
    state_enter(delete_operator_state, {})
  elif key == 'g':
    state_enter(g_command_state, {})
  elif is_number(key):
    state_enter(get_operator_count_state, {'count': key})
  elif key == 'G'
    jump_to_eof()
  return true

def command_line_state(data, key):
  if key == '<cr>':
    if data['input']:
      execute_ex_command(data['input'])
    return false
  elif key == '<esc>'
    return false

  if not data['input']:
    data['input'] = ''

  data['input'] += key
  return true

def delete_operator_state(data, key):
  count = data['count'] or 1
  if key == 'w':
    delete_word(count)
  elif key == '$':
    delete_to_eol(count)
  return false  # return to normal mode

def g_command_state(data, key):
  if key == 'g':
    go_top()
  elif key == 'v':
    reselect()
  return false  # return to normal mode

def get_operator_count_state(data, key):
  if is_number(key):
    data['count'] += key
    return true
  unshift_key(key)  # return key to the input buffer
  state_enter(delete_operator_state, data)
  return false

def insert_state(data, key):
  if key == '<esc>':
    return false  # exit insert mode
  self_insert(key)
  return true
```

실제 코드는 훨씬 복잡하지만 위의 내용은 Neovim이 내부적으로 어떻게 구성되어 있는지에 대한 아이디어를 제공합니다.
`g_command_state` 나`get_operator_count_state`와 같은 몇몇 상태는 전용`state_enter` 콜백을 가지고 있지 않지만 
다른 상태로 암시적으로 내포됩니다 (이것은 업데이트를 계속할 때 나중에 바뀔 것입니다).
실제 코드를 읽으려면 다음의 권장 순서를 따르십시오.

1. `state_enter ()`함수 (state.c)는 실제 프로그램 루프입니다. 
    콜백 및 상태 데이터에 대한 함수 포인터를 포함하는`VimState` 구조가 사용됩니다.
2. `main ()`함수 (main.c)는 모든 시작 후 normal_enter는 정상 모드로 들어가기 위해 함수의 끝에서 호출됩니다.
3. `normal_enter()` 함수 (normal.c)는 NormalState 구조체를 설정하고`state_enter`를 호출하는 작은 래퍼입니다.
4. `normal_check()` 함수 (normal.c)는 일반 모드의 각 반복 전에 호출됩니다.
5.  normal_execute ()`함수 (normal.c)는 키가 일반 모드에서 읽힐 때 호출됩니다.

3, 4, 5의 일반 모드에서 설명한 기본 구조는 'state_enter` 루프에 의해 관리되는 다른 모드에 사용됩니다.

- command-line mode: `command_line_{enter,check,execute}()`(`ex_getln.c`)
- insert mode: `insert_{enter,check,execute}()`(`edit.c`)
- terminal mode: `terminal_{enter,execute}()`(`terminal.c`)

### Async event support

One of the features Neovim added is the support for handling arbitrary
asynchronous events, which can include:

- msgpack-rpc requests
- job control callbacks
- timers (not implemented yet but the support code is already there)

Neovim implements this functionality by entering another event loop while
waiting for characters, so instead of:

```py
def state_enter(state_callback, data):
  do
    key = readkey()                 # read a key from the user
  while state_callback(data, key)   # invoke the callback for the current state
```

Neovim program loop is more like:

```py
def state_enter(state_callback, data):
  do
    event = read_next_event()       # read an event from the operating system
  while state_callback(data, event) # invoke the callback for the current state
```

where `event` is something the operating system delivers to us, including (but
not limited to) user input. The `read_next_event()` part is internally
implemented by libuv, the platform layer used by Neovim.

Since Neovim inherited its code from Vim, the states are not prepared to receive
"arbitrary events", so we use a special key to represent those (When a state
receives an "arbitrary event", it normally doesn't do anything other update the
screen).
