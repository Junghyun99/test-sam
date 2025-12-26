pip install lizard

# tests 폴더와 venv 폴더를 제외하고 src 폴더만 검사
lizard -E duplicate -x "./tests/*" -x "./venv/*" src/


================================================
Duplicated code found:
  ./file_A.py:10-20
  ./file_B.py:10-20
------------------------------------------------   <-- 첫 번째 중복 그룹 끝
  ./logic_X.py:50-60
  ./logic_Y.py:100-110
  ./logic_Z.py:5-15
------------------------------------------------   <-- 두 번째 중복 그룹 끝

# 중복된 코드가 10줄 이상일 때만 보고해라 (현재 예제는 짧아서 아무것도 안 뜸)
lizard -Eduplicate --min_duplicate 10 dup_code.py


제시해주신 소스 코드를 직접 분석해 보니, 제가 처음에 말씀드린 `self.matches`는 이전 버전의 유산이거나 다른 확장 프로그램의 구조였던 것 같습니다.

제공해주신 코드 기준으로 보면, **`get_duplicates()`** 메서드가 정확히 우리가 원하는 데이터를 `yield`(제너레이터) 방식으로 반환하고 있습니다.

따라서 `print_result()`를 호출하는 대신, **`get_duplicates()`를 직접 호출하여 리스트로 변환**하면 변수에 담을 수 있습니다.

### 1. 변수로 리턴받는 코드 예시

```python
import lizard
from lizard_ext.lizardduplicate import LizardExtension

def get_duplication_data(paths, min_tokens=70):
    # 1. 확장 인스턴스 생성
    dup_ext = LizardExtension()

    # 2. 분석 실행 
    # lizard.analyze는 파일을 하나씩 분석하며 내부적으로 
    # dup_ext.cross_file_process를 호출하여 데이터를 쌓습니다.
    # 결과를 끝까지 소비하기 위해 list()로 감싸줍니다.
    list(lizard.analyze(paths, extensions=[dup_ext]))

    # 3. get_duplicates() 메서드를 호출하여 결과 데이터 가져오기
    # 이 메서드는 [CodeSnippet, CodeSnippet, ...] 형태의 리스트를 yield 합니다.
    raw_duplicates = list(dup_ext.get_duplicates(min_duplicate_tokens=min_tokens))

    # 4. 가공해서 변수에 저장
    final_results = []
    for dup_group in raw_duplicates:
        group_data = []
        for snippet in dup_group:
            group_data.append({
                "file": snippet.file_name,
                "start": snippet.start_line,
                "end": snippet.end_line
            })
        final_results.append(group_data)
    
    return final_results, dup_ext.duplicate_rate()

# --- 실행 ---
my_results, rate = get_duplication_data(["./src"])

for i, group in enumerate(my_results):
    print(f"중복 그룹 {i+1}:")
    for loc in group:
        print(f"  - {loc['file']} ({loc['start']}~{loc['end']}라인)")
print(f"전체 중복율: {rate * 100:.2f}%")
```

---

### 2. 코드 내부 동작 원리 (제공하신 소스 기준)

보내주신 `lizardduplicate.py` 파일의 구조를 보면 흐름은 이렇습니다:

1.  **`cross_file_process(self, fileinfos)`**:
    *   `lizard.analyze`가 실행될 때 호출됩니다.
    *   모든 파일의 토큰 정보(`hash_nodes`)를 `self.nodes`라는 하나의 큰 리스트에 합칩니다.
2.  **`get_duplicates(self, min_duplicate_tokens=70)`**:
    *   핵심 로직입니다. `DuplicateFinder`를 생성해서 합쳐진 `self.nodes`에서 중복을 찾아냅니다.
    *   찾아낸 중복 구간(인덱스)을 **`CodeSnippet` 객체의 리스트**로 만들어서 `yield` 합니다.
3.  **`CodeSnippet` 클래스**:
    *   `file_name`, `start_line`, `end_line` 속성을 가지고 있어 우리가 필요한 정보를 모두 담고 있습니다.

### 요약
제공해주신 소스 코드에서는 **`dup_ext.get_duplicates()`**를 호출하는 것이 정답입니다. 이 함수는 `print`를 하지 않고 데이터를 생성해서 던져주므로, 이를 `list()`로 묶어서 사용하시면 바로 변수화가 가능합니다.


하나의 파일 내에서 발생하는 중복 코드만 검사하고 싶을 때도 동일한 **`LizardExtension`**을 사용할 수 있습니다. Lizard의 중복 검출 로직은 파일이 하나든 여러 개든 상관없이, 입력된 전체 토큰 뭉치에서 반복되는 패턴을 찾기 때문입니다.

파일 하나만 대상으로 할 때의 코드 예시와 주의할 점을 정리해 드립니다.

### 1. 단일 파일 중복 검사 코드

```python
import lizard
from lizard_ext.lizardduplicate import LizardExtension

def find_internal_duplicates(file_path, min_tokens=50):
    # 1. 확장 인스턴스 생성
    dup_ext = LizardExtension()

    # 2. 분석 실행 (파일 경로 하나만 리스트에 담아 전달)
    # 한 파일만 분석하더라도 analyze를 거쳐야 내부 토큰화(Tokenizing)가 진행됩니다.
    list(lizard.analyze([file_path], extensions=[dup_ext]))

    # 3. 중복 결과 추출
    # min_duplicate_tokens: 중복으로 간주할 최소 토큰 수 (낮을수록 짧은 중복도 잡음)
    raw_duplicates = list(dup_ext.get_duplicates(min_duplicate_tokens=min_tokens))

    # 4. 결과 정리
    results = []
    for dup_group in raw_duplicates:
        # dup_group은 같은 코드가 나타난 지점(CodeSnippet)들의 리스트입니다.
        group_info = [
            {"line_start": s.start_line, "line_end": s.end_line}
            for s in dup_group
        ]
        results.append(group_info)
    
    return results

# --- 실행 예시 ---
file_to_check = "my_script.py"
duplicates = find_internal_duplicates(file_to_check, min_tokens=40)

if not duplicates:
    print("파일 내에 중복된 코드가 없습니다.")
else:
    for i, group in enumerate(duplicates):
        print(f"[{i+1}]번 중복 패턴 발견:")
        for loc in group:
            print(f"   - {loc['line_start']}라인 ~ {loc['line_end']}라인")
```

### 2. 왜 이렇게 작동하나요? (소스코드 기반 설명)

보내주신 `lizardduplicate.py` 코드의 핵심을 보면 이해가 빠릅니다.

1.  **`cross_file_process`**: 이 함수는 파일들을 돌면서 모든 토큰을 `self.nodes`라는 하나의 거대한 리스트에 다 합쳐버립니다.
    *   파일이 1개라면 `self.nodes`에는 그 파일의 토큰만 들어갑니다.
2.  **`DuplicateFinder`**: 이 클래스는 `self.nodes` 안에서 반복되는 구간을 찾습니다. 
    *   이때 소스코드 상의 `boundaries`는 파일의 시작과 끝 지점을 나타냅니다. 
    *   파일이 하나뿐이면 경계선이 시작과 끝(0과 끝 인덱스)뿐이므로, **그 안에서 자기 자신과 겹치는 구간**을 찾게 됩니다.

### 3. 단일 파일 검사 시 팁

*   **`min_duplicate_tokens` 조절**: 
    *   기본값인 70은 꽤 긴 코드 블록입니다. 파일 내부의 소소한 중복까지 잡고 싶다면 **30~50** 정도로 낮추는 것이 좋습니다.
*   **파일 크기**: 
    *   파일이 너무 짧으면 중복이 검색되지 않을 수 있습니다. (Lizard는 기본적으로 어느 정도 의미 있는 길이의 중복을 찾도록 설계됨)
*   **자기 복제**:
    *   동일한 파일 내에서 A라는 코드가 10번 라인과 50번 라인에 동시에 존재한다면, `get_duplicates`는 두 지점의 라인 정보를 담은 `CodeSnippet` 리스트를 반환합니다.

이렇게 하면 별도의 복잡한 설정 없이도 특정 파일 하나에 대한 "코드 복붙" 여부를 쉽게 확인할 수 있습니다.