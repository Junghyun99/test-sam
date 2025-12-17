Cline은 기본적으로 **하나의 Hook 이벤트(`post_tool_use`)당 하나의 파일**만 실행합니다.

따라서 각 툴마다 별도의 스크립트를 실행하려면, **'메인 Hook 파일'을 라우터(Router)나 디스패처(Dispatcher)처럼 만들어야 합니다.**

즉, `post_tool_use`라는 메인 스크립트가 JSON 입력을 받아서, **어떤 툴이 사용되었는지 확인(`if/switch`)하고, 그에 맞는 다른 스크립트를 호출**하는 방식입니다.

Python이 JSON 파싱과 조건 분기에 유리하므로 Python을 사용한 **Router 패턴** 예시를 보여드리겠습니다.

---

### 방법: 메인 훅을 "관리자"로 만들기

#### 1. 폴더 구조 예시
`.cline/hooks/` 폴더를 아래와 같이 구성한다고 가정해 봅시다.

```text
.cline/hooks/
├── post_tool_use          (메인 훅 파일 - 실행 권한 필수)
└── scripts/               (실제 로직이 담긴 스크립트들)
    ├── on_command.py      (execute_command 툴 실행 후 돌릴 스크립트)
    ├── on_browser.js      (browser_action 툴 실행 후 돌릴 스크립트)
    └── logger.sh          (모든 툴 실행 후 공통으로 돌릴 스크립트)
```

#### 2. 메인 `post_tool_use` 작성 (Router)

이 파일이 "교통정리"를 담당합니다. 들어온 데이터를 분석해서 적절한 하위 스크립트로 토스합니다.

```python
#!/usr/bin/env python3
import sys
import json
import subprocess
import os

# 1. Cline에서 보낸 JSON 입력 받기
raw_input = sys.stdin.read()
if not raw_input:
    sys.exit(0)

data = json.loads(raw_input)

# 2. 현재 실행된 툴 이름 파악하기
# (Cline Hook 데이터 구조에 따라 'call' -> 'name' 확인)
tool_name = data.get("call", {}).get("name")

# 스크립트들이 있는 디렉토리 경로 설정
current_dir = os.path.dirname(os.path.abspath(__file__))
scripts_dir = os.path.join(current_dir, "scripts")

# 3. 툴 이름에 따라 다른 스크립트 실행 (라우팅 로직)
try:
    if tool_name == "execute_command":
        # 예: 터미널 명령 후에는 파이썬 스크립트 실행
        subprocess.run([os.path.join(scripts_dir, "on_command.py")], input=raw_input, text=True, check=True)

    elif tool_name == "browser_action":
        # 예: 브라우저 액션 후에는 Node.js 스크립트 실행
        subprocess.run(["node", os.path.join(scripts_dir, "on_browser.js")], input=raw_input, text=True, check=True)
    
    # (선택) 툴 상관없이 무조건 실행할 공통 로직
    # subprocess.run([os.path.join(scripts_dir, "logger.sh")], input=raw_input, text=True)

except Exception as e:
    # 하위 스크립트 에러가 메인 훅을 멈추게 할지, 무시할지 결정 (여기선 stderr에 로그만 남김)
    sys.stderr.write(f"[Error in sub-script] {e}\n")

# 4. Cline에게 최종 결과 반환 (필수)
# 하위 스크립트가 결과를 조작하지 않는다면, 원본 데이터를 그대로 혹은 빈 성공 응답을 보냅니다.
print(json.dumps({"result": "success"}))
```

#### 3. 하위 스크립트 작성 시 주의사항

하위 스크립트(`on_command.py` 등)를 작성할 때 가장 중요한 점은 **표준 출력(stdout) 관리**입니다.

*   **Cline은 메인 훅(`post_tool_use`)의 stdout만 기다립니다.**
*   하위 스크립트에서 `print("hello")` 등을 막 하면, 그 내용이 메인 훅의 stdout에 섞여 들어가 JSON 파싱 에러(`SyntaxError`)를 일으킬 수 있습니다.
*   **해결책:** 하위 스크립트에서의 로그나 메시지는 반드시 **`sys.stderr`** 로 출력하거나 파일로 저장해야 합니다.

**예시 (`scripts/on_command.py`):**
```python
import sys
import json

# 메인 훅에서 넘겨준 stdin 읽기
data = json.load(sys.stdin)

# 로직 수행
command_result = data.get("result", "")

# 로그는 stderr로! (stdout 절대 금지 X)
sys.stderr.write(f"Command finished. Output length: {len(command_result)}\n")

# 파일 기록 등 사이드 이펙트 수행
with open("/tmp/cline_log.txt", "a") as f:
    f.write(f"Executed: {data['call']['name']}\n")
```

### 요약

1.  **진입점은 하나:** `post_tool_use` 파일 하나만 실행됩니다.
2.  **라우팅:** 그 파일 안에서 `if tool_name == "..."` 로직을 짜서 다른 파일들을 `subprocess`로 실행하세요.
3.  **데이터 전달:** 메인 훅이 받은 `stdin` 데이터를 하위 스크립트에게 그대로 `input=` 파라미터로 넘겨줄 수 있습니다.
4.  **출력 주의:** 하위 스크립트는 절대 `stdout`에 아무거나 찍지 말고, `stderr`를 사용해야 Cline과의 통신이 깨지지 않습니다.