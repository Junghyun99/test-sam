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