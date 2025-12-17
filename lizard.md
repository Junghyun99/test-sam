Python `lizard` 모듈을 import하여 파일의 **NLOC(Non-Comment Source Lines of Code)**를 확인하는 방법은 다음과 같습니다.

`lizard.analyze_file()` 함수를 사용하면 파일 분석 결과가 담긴 객체를 반환받으며, 이 객체의 `.nloc` 속성을 통해 전체 코드 라인 수를 확인할 수 있습니다.

### 1. 전체 파일의 NLOC 확인하기

`lizard.analyze_file("파일경로")`를 호출하여 분석 정보를 가져온 뒤, `nloc` 속성을 조회합니다.

```python
import lizard

# 분석할 파일 경로
file_path = "target_file.py"

# 파일 분석 수행
result = lizard.analyze_file(file_path)

# 전체 파일의 NLOC 출력
print(f"전체 NLOC: {result.nloc}")
```

### 2. 함수별 NLOC 확인하기

분석 결과 객체(`result`)에는 `function_list`라는 리스트가 포함되어 있습니다. 이를 순회하면 파일 내 개별 함수의 NLOC도 확인할 수 있습니다.

```python
import lizard

file_path = "target_file.py"
result = lizard.analyze_file(file_path)

print(f"[{file_path}] 분석 결과")
print(f"Total NLOC: {result.nloc}\n")

# 함수별 정보 출력
for func in result.function_list:
    print(f"함수명: {func.name}")
    print(f"  - NLOC: {func.nloc}")
    print(f"  - 복잡도(CCN): {func.cyclomatic_complexity}")
    print("-" * 20)
```

### 3. 소스 코드 문자열로 NLOC 확인하기 (파일 없이)

파일이 아니라 메모리에 있는 소스 코드 문자열을 바로 분석하고 싶다면 `analyze_source_code`를 사용합니다.

```python
import lizard

code_snippet = """
def hello():
    print("Hello")
    return True
"""

# (가상 파일명, 소스코드 문자열) 전달
result = lizard.analyze_file.analyze_source_code("memory_file.py", code_snippet)

print(f"NLOC: {result.nloc}")
```

**참고:** `result` 객체는 `lizard.FileInformation` 클래스의 인스턴스로, `nloc` 외에도 `token_count`, `average_cyclomatic_complexity` 등의 속성을 제공합니다.




`lizard` 모듈을 import해서 사용할 때 CLI의 `-m` (Modified Cyclomatic Complexity, 수정된 순환 복잡도) 옵션을 적용하려면, **`analyze_file` 함수의 `extensions` 파라미터**를 사용해야 합니다.

`-m` 옵션은 내부적으로 `lizard.ModifiedComplexity` 클래스를 사용하므로, 이 클래스의 인스턴스를 리스트에 담아 전달하면 됩니다.

### 사용 방법

```python
import lizard

file_path = "target_file.py"

# 1. ModifiedComplexity 인스턴스 생성 (-m 옵션 역할)
extensions = [lizard.ModifiedComplexity()]

# 2. analyze_file에 extensions 파라미터 전달
result = lizard.analyze_file(file_path, extensions=extensions)

# 3. 결과 확인
for func in result.function_list:
    print(f"함수명: {func.name}")
    # 이제 이 복잡도는 switch/case(또는 elif) 문을 1개의 복잡도로 계산한 값입니다.
    print(f"  - Modified CCN: {func.cyclomatic_complexity}") 
```

---

### 비교 예시 (일반 vs `-m` 적용)

`if/elif`가 많은 코드로 차이를 확인해 보겠습니다.

```python
import lizard

# 많은 조건문이 있는 코드 (일반 CCN은 높게 나옴)
code_snippet = """
def complex_decision(x):
    if x == 1: return "one"
    elif x == 2: return "two"
    elif x == 3: return "three"
    elif x == 4: return "four"
    elif x == 5: return "five"
    return "other"
"""

# 1. 일반 분석 (Default)
result_normal = lizard.analyze_file.analyze_source_code("test.py", code_snippet)
ccn_normal = result_normal.function_list[0].cyclomatic_complexity

# 2. -m 옵션 적용 분석 (Modified)
result_modified = lizard.analyze_file.analyze_source_code(
    "test.py", 
    code_snippet, 
    extensions=[lizard.ModifiedComplexity()] # 핵심 부분
)
ccn_modified = result_modified.function_list[0].cyclomatic_complexity

print(f"일반 복잡도(CCN): {ccn_normal}")      # 결과: 6 (조건 분기마다 +1)
print(f"수정된 복잡도(Modified): {ccn_modified}") # 결과: 1 (switch/elif 나열은 1로 계산)
```

**요약:**
*   `extensions=[lizard.ModifiedComplexity()]`를 `analyze_file`에 인자로 넘겨주면 `-m` 옵션과 동일하게 동작합니다.