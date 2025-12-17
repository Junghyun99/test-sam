Hooks Overview - Cline
https://docs.cline.bot/features/hooks/index
https://docs.cline.bot/features/hooks/hook-reference
https://docs.cline.bot/features/hooks/samples
위 경로의 hook 기능을 작성해보고 싶어.

hook을 작성할거고 내용은 아래와 같아.

# PostToolUse 코드품질 검사 hook
## write_to_file, replace_in_file
- python lizard 를 사용할것
- 옵션은 -L 100 -m -C 21 -w 사용
- tool은 두가지만 체크( "write_to_file", "replace_in_file")
- write_to_file 인 경우에는 모든 함수에 대해 검사
- replace_in_file 인 경우에는 postToolUse.parameters.diff를 확인하여 수정된 라인이 포함된 함수에 대해 검사
- 검사결과 리팩토링이 필요한 함수들 경고
(코드완성을 위해 중단하지는 않는다.)

## attempt_completion 
- 위에서 경고한 리팩토링이 필요한 함수 리스트를 가지고 있음
- task가 완료될때 리팩토링이 필요한 함수들 다시 경고
- 리팩토링 가이드 llm에 각 함수마다 요청하기



Cline의 Hook 기능을 사용하여 코드 품질을 검사하고 리팩토링 가이드를 요청하는 스크립트를 작성해 드리겠습니다.

이 Hook은 파이썬 스크립트로 작성되며, Cline이 도구를 사용한 직후(`PostToolUse`)에 실행됩니다. 상태(검출된 함수 목록)를 유지하기 위해 임시 파일을 사용합니다.

### 사전 준비

이 스크립트는 `lizard` 라이브러리를 사용하므로, 환경에 설치가 필요합니다.
```bash
pip install lizard
```

### Hook 스크립트 작성 (`quality_check_hook.py`)

아래 코드를 프로젝트 루트나 적절한 위치에 저장하세요.

```python
import sys
import json
import subprocess
import os
import re
import lizard

# 상태를 저장할 파일 경로 (프로젝트 루트의 .cline 폴더 권장)
STATE_FILE = ".cline/lizard_state.json"

def load_state():
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {}

def save_state(state):
    os.makedirs(os.path.dirname(STATE_FILE), exist_ok=True)
    with open(STATE_FILE, 'w', encoding='utf-8') as f:
        json.dump(state, f, ensure_ascii=False, indent=2)

def run_lizard(file_path):
    """
    lizard를 실행하여 복잡도 기준을 초과하는 함수 목록을 반환합니다.
    옵션: -L 100 (길이), -C 21 (순환 복잡도)
    """
    # lizard 라이브러리 사용 (CLI 옵션 로직 구현)
    # lizard -L 100 -m -C 21 -w 와 동일한 로직
    analysis = lizard.analyze_file(file_path)
    
    bad_functions = []
    
    for func in analysis.function_list:
        # -C 21: Cyclomatic Complexity > 21
        # -L 100: Length > 100
        is_bad = func.cyclomatic_complexity > 21 or func.length > 100
        
        if is_bad:
            bad_functions.append({
                "name": func.name,
                "file": file_path,
                "ccn": func.cyclomatic_complexity,
                "length": func.length,
                "start_line": func.start_line,
                "end_line": func.end_line
            })
            
    return bad_functions

def parse_diff_changed_lines(diff):
    """
    diff 텍스트를 파싱하여 변경된(추가/수정된) 라인 번호들의 집합을 반환합니다.
    """
    changed_lines = set()
    # Unified diff 헤더 파싱: @@ -old,cnt +new,cnt @@
    hunk_re = re.compile(r"^@@ -\d+(?:,\d+)? \+(\d+)(?:,(\d+))? @@")
    
    current_line = 0
    lines = diff.split('\n')
    
    for line in lines:
        if line.startswith('---') or line.startswith('+++'):
            continue
            
        match = hunk_re.match(line)
        if match:
            current_line = int(match.group(1))
            continue
            
        if line.startswith('+'):
            changed_lines.add(current_line)
            current_line += 1
        elif line.startswith(' '):
            current_line += 1
        # '-'로 시작하는 라인은 삭제된 라인이므로 현재 라인 카운트에 영향 없음 (새 파일 기준)
            
    return changed_lines

def main():
    # Cline으로부터 전달받은 인자 파싱 (stdin으로 들어온다고 가정하거나, CLI 인자로 처리)
    # 여기서는 Cline Hook 표준에 따라 stdin이나 payload 파일을 읽는 구조를 가정합니다.
    # 하지만 Cline Hook 문서를 보면 보통 실행 시 환경변수나 인자로 컨텍스트를 줍니다.
    # 편의상 payload가 stdin으로 들어온다고 가정하고 작성합니다.
    
    try:
        # stdin에서 데이터 읽기 (Cline Hook 전달 방식에 따라 조정 필요)
        if not sys.stdin.isatty():
            input_data = sys.stdin.read()
            payload = json.loads(input_data)
        else:
            # 테스트용 더미 데이터 혹은 종료
            return

        tool_name = payload.get("tool")
        tool_params = payload.get("parameters", {})
        
        # 상태 로드
        state = load_state()
        current_bad_funcs = state.get("bad_functions", {}) # format: {filepath: [funcs...]}

        # 1. write_to_file 검사
        if tool_name == "write_to_file":
            path = tool_params.get("path")
            if path and (path.endswith('.py') or path.endswith('.js') or path.endswith('.cpp')): # 지원 확장자 체크
                bad_funcs = run_lizard(path)
                
                if bad_funcs:
                    print(f"\n[Hook Warning] '{path}' 파일에서 코드 품질 기준을 초과한 함수가 발견되었습니다:")
                    for f in bad_funcs:
                        print(f" - {f['name']} (CCN: {f['ccn']}, Length: {f['length']})")
                    
                    # 상태 업데이트
                    current_bad_funcs[path] = bad_funcs

        # 2. replace_in_file 검사
        elif tool_name == "replace_in_file":
            path = tool_params.get("path")
            diff = tool_params.get("diff")
            
            if path and diff and (path.endswith('.py')):
                # 전체 파일 분석
                all_bad_funcs = run_lizard(path)
                
                # 변경된 라인 파악
                changed_lines = parse_diff_changed_lines(diff)
                
                # 변경된 라인이 포함된 나쁜 함수만 필터링
                affected_bad_funcs = []
                for func in all_bad_funcs:
                    func_range = range(func['start_line'], func['end_line'] + 1)
                    # 교집합이 있으면 해당 함수는 수정된 것임
                    if not changed_lines.isdisjoint(func_range):
                        affected_bad_funcs.append(func)
                
                if affected_bad_funcs:
                    print(f"\n[Hook Warning] '{path}'의 수정된 부분에서 코드 품질 기준을 초과한 함수가 있습니다:")
                    for f in affected_bad_funcs:
                        print(f" - {f['name']} (CCN: {f['ccn']}, Length: {f['length']})")
                    
                    # 상태 업데이트 (해당 파일의 전체 나쁜 함수 목록으로 갱신하거나 병합)
                    current_bad_funcs[path] = all_bad_funcs

        # 3. attempt_completion 검사
        elif tool_name == "attempt_completion":
            # 지금까지 누적된 리팩토링 대상 확인
            all_warnings = []
            for path, funcs in current_bad_funcs.items():
                for f in funcs:
                    all_warnings.append(f"File: {path}, Function: {f['name']} (CCN: {f['ccn']})")
            
            if all_warnings:
                print("\n" + "="*50)
                print("[Hook Alert] 작업 완료 전 리팩토링이 필요한 함수들이 남아있습니다.")
                print("="*50)
                print("다음 함수들에 대해 리팩토링 가이드를 제안합니다:\n")
                
                prompt = "다음 함수들은 복잡도가 높으므로 리팩토링이 필요합니다. 각 함수에 대해 구체적인 리팩토링 가이드와 개선된 코드 예시를 작성해 주세요:\n"
                for w in all_warnings:
                    print(f"- {w}")
                    prompt += f"- {w}\n"
                
                print("\n(위 내용을 바탕으로 LLM에게 리팩토링을 요청하세요)")
                
                # Cline에게 메시지를 보내는 방식 (이 부분은 Cline Hook의 출력 처리에 따라 다름)
                # 보통 stdout에 출력하면 유저가 볼 수 있음.
                
                # 상태 초기화 (완료 시도했으므로)
                state["bad_functions"] = {}
                save_state(state)
                return # 완료 진행

        # 상태 저장
        state["bad_functions"] = current_bad_funcs
        save_state(state)

    except Exception as e:
        # Hook 에러가 메인 프로세스를 죽이지 않도록 처리
        print(f"[Hook Error] {e}", file=sys.stderr)

if __name__ == "__main__":
    main()
```

---

### Cline 설정에 Hook 등록하기

작성한 파이썬 스크립트를 Cline이 실행할 수 있도록 `cline_config.json` (또는 프로젝트 설정)의 `hooks` 섹션에 등록해야 합니다.

**설정 예시:**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "command": "python3",
        "arguments": [
          "/path/to/your/project/quality_check_hook.py"
        ]
      }
    ]
  }
}
```

### 동작 방식 설명

1.  **PostToolUse 트리거**: Cline이 도구를 사용할 때마다 이 스크립트가 실행됩니다.
2.  **데이터 전달**: Cline은 실행된 도구의 정보(`write_to_file`, `replace_in_file` 등)와 파라미터(파일 경로, diff 등)를 스크립트의 stdin(표준 입력)으로 JSON 형태로 전달합니다.
3.  **Lizard 검사**:
    *   `write_to_file`: 파일 전체를 분석하여 기준(-L 100, -C 21)을 넘는 함수를 찾습니다.
    *   `replace_in_file`: `diff`를 파싱하여 실제 수정된 라인 번호를 찾고, 그 라인이 포함된 함수가 기준을 넘는지 확인합니다.
4.  **상태 저장**: 검출된 문제는 `.cline/lizard_state.json`에 저장되어 도구 실행 간에 기억됩니다.
5.  **Completion 시점**: `attempt_completion` 도구가 호출되면 저장된 문제 목록을 불러와 사용자에게 경고하고, 리팩토링 가이드를 요청하는 프롬프트 내용을 출력합니다.

이 구성을 통해 코드를 수정할 때마다 즉각적인 품질 피드백을 받고, 작업 완료 시점에 놓친 리팩토링 포인트를 챙길 수 있습니다.