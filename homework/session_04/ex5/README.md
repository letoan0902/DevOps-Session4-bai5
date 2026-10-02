# Bài 5 - Khôi phục trạng thái và đảo ngược commit (Reset vs Revert)

Session 04 - DevOps. Báo cáo này trình bày hai kỹ thuật sửa sai trong lịch sử Git: `git reset` (sửa lịch sử cục bộ) và `git revert` (đảo ngược an toàn bằng commit mới). Mọi lệnh dưới đây đã được chạy thật bằng Git Bash trên Windows 11 (git 2.49.0.windows.1, Python 3.13.3); output được chép nguyên văn từ `terminal_log.txt`.

## 1. Mục tiêu

- Dùng thành thạo `git reset` và `git revert` để sửa sai sót trong lịch sử dự án.
- Phân biệt "khôi phục lịch sử cục bộ" (reset, viết lại lịch sử) với "đảo ngược lịch sử an toàn" (revert, thêm commit mới) khi làm việc nhóm.
- Trường hợp 1: lùi HEAD một commit, giữ thay đổi ở dạng Modified (unstaged), không dùng `--hard`.
- Trường hợp 2: tạo commit `Revert "..."` ở đầu nhánh để đảo ngược một commit cũ.

## 2. Kịch bản

Repo thử nghiệm: `reset-revert-demo`, nhánh `main`, cấu hình local: `user.name = Lê Toàn`, `user.email = trinhhanh261293@gmail.com`, `core.autocrlf = false`. File `calculator.py` gồm các hàm `add`, `subtract`, `multiply`, `divide`.

Lịch sử xây dựng qua các bước:

| Bước | Commit | Ghi chú |
|------|--------|---------|
| 1 | `4b8df38 feat: them ham add va subtract` | commit đầu tiên |
| 2 | `5cc69d1 feat: them ham multiply` | phép nhân đúng |
| 3 (TH1) | `e717cb8 feat: them ham divide` | commit LỖI cục bộ: `divide` dùng sai toán tử `*` |
| 3 (TH1) | `8631d56 feat: them ham divide co kiem tra chia cho 0` | sau reset, viết lại đúng |
| 4 (TH2) | `3c27d09 feat: doi cong thuc multiply` | commit A LỖI: `multiply` thành `a + b` |
| 4 (TH2) | `00841ae docs: them README` | commit B bình thường, khiến commit lỗi KHÔNG phải commit cuối |
| 4 (TH2) | `a93b9ee Revert "feat: doi cong thuc multiply"` | commit revert |

Khởi tạo và hai commit đầu:

```
$ git init -b main
Initialized empty Git repository in D:/Rikkei/DevOps/Session4/_demo/bai5/reset-revert-demo/.git/

$ git config user.name "Lê Toàn"

$ git config user.email "trinhhanh261293@gmail.com"

$ git config core.autocrlf false

$ git commit -m "feat: them ham add va subtract"
[main (root-commit) 4b8df38] feat: them ham add va subtract
 1 file changed, 6 insertions(+)
 create mode 100644 calculator.py

$ git commit -m "feat: them ham multiply"
[main 5cc69d1] feat: them ham multiply
 1 file changed, 4 insertions(+)

$ python -c "import calculator as c; print(\"add(6, 3) =\", c.add(6, 3)); print(\"subtract(6, 3) =\", c.subtract(6, 3)); print(\"multiply(6, 3) =\", c.multiply(6, 3))"
add(6, 3) = 9
subtract(6, 3) = 3
multiply(6, 3) = 18

$ git log --oneline -n 5
5cc69d1 feat: them ham multiply
4b8df38 feat: them ham add va subtract
```

(Toàn bộ lệnh tạo file, `git add` và các bước trung gian có trong `terminal_log.txt`.)

## 3. Trường hợp 1 - Sửa đổi cục bộ: `git reset --mixed HEAD~1`

### 3.1. Tạo commit lỗi

Hàm `divide` được thêm vào nhưng dùng sai toán tử (`a * b` thay vì `a / b`) và chưa kiểm tra chia cho 0:

```
$ git commit -m "feat: them ham divide"
[main e717cb8] feat: them ham divide
 1 file changed, 4 insertions(+)

$ python -c "import calculator as c; print(\"divide(6, 3) =\", c.divide(6, 3), \"(sai, ky vong 2.0)\")"
divide(6, 3) = 18 (sai, ky vong 2.0)

$ git log --oneline -n 5
e717cb8 feat: them ham divide
5cc69d1 feat: them ham multiply
4b8df38 feat: them ham add va subtract
```

Kết quả `divide(6, 3) = 18` là sai (kỳ vọng 2.0), chứng minh commit `e717cb8` chứa mã lỗi. Commit này mới chỉ nằm ở local, chưa push, nên được phép viết lại lịch sử.

### 3.2. Lùi HEAD một commit, giữ thay đổi ở Working Directory

`--mixed` là chế độ mặc định của `git reset`; viết rõ ra để minh bạch. Lệnh này KHÔNG dùng `--hard`:

```
$ git reset --mixed HEAD~1
Unstaged changes after reset:
M	calculator.py
```

Lưu ý: phiên bản git 2.49.0.windows.1 in thông báo thành hai dòng (`Unstaged changes after reset:` rồi `M	calculator.py`), nội dung tương đương dạng một dòng `Unstaged changes after reset: M calculator.py` nêu trong đề.

### 3.3. Kiểm tra kết quả

```
$ git log --oneline -n 5
5cc69d1 feat: them ham multiply
4b8df38 feat: them ham add va subtract

$ git status
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   calculator.py

no changes added to commit (use "git add" and/or "git commit -a")

$ git diff
diff --git a/calculator.py b/calculator.py
index 3292da0..7c61a57 100644
--- a/calculator.py
+++ b/calculator.py
@@ -8,3 +8,7 @@ def subtract(a, b):
 
 def multiply(a, b):
     return a * b
+
+
+def divide(a, b):
+    return a * b

$ git reflog -n 4
5cc69d1 HEAD@{0}: reset: moving to HEAD~1
e717cb8 HEAD@{1}: commit: feat: them ham divide
5cc69d1 HEAD@{2}: commit: feat: them ham multiply
4b8df38 HEAD@{3}: commit (initial): feat: them ham add va subtract
```

Giải thích:

- `git log`: HEAD đã lùi về `5cc69d1`; commit `e717cb8` không còn trên nhánh `main`.
- `git status`: `calculator.py` ở trạng thái "Changes not staged for commit" (Modified, chưa staged).
- `git diff`: mã `divide` lỗi vẫn còn nguyên trong Working Directory, không bị mất, đúng ràng buộc "không làm mất mã nguồn hiện hành".
- `git reflog`: commit `e717cb8` vẫn được ghi nhận (`HEAD@{1}`), nên có thể phục hồi nếu cần (xem mục 8).

### 3.4. Viết lại code đúng và commit lại

```
$ git diff
diff --git a/calculator.py b/calculator.py
index 3292da0..4e9244b 100644
--- a/calculator.py
+++ b/calculator.py
@@ -8,3 +8,9 @@ def subtract(a, b):
 
 def multiply(a, b):
     return a * b
+
+
+def divide(a, b):
+    if b == 0:
+        raise ValueError("Khong the chia cho 0")
+    return a / b

$ git add calculator.py

$ git commit -m "feat: them ham divide co kiem tra chia cho 0"
[main 8631d56] feat: them ham divide co kiem tra chia cho 0
 1 file changed, 6 insertions(+)

$ python -c "import calculator as c; print(\"divide(6, 3) =\", c.divide(6, 3))"
divide(6, 3) = 2.0

$ git log --oneline -n 5
8631d56 feat: them ham divide co kiem tra chia cho 0
5cc69d1 feat: them ham multiply
4b8df38 feat: them ham add va subtract

$ git status
On branch main
nothing to commit, working tree clean
```

Lịch sử sạch: không còn dấu vết của commit lỗi `e717cb8` trên nhánh (chỉ còn trong reflog cục bộ).

### 3.5. Sơ đồ ASCII trước và sau reset

```
TRUOC reset (HEAD tro vao commit loi):

  4b8df38 --- 5cc69d1 --- e717cb8
                          (loi)   <- HEAD, main
  Index = e717cb8, Working Dir = e717cb8

SAU git reset --mixed HEAD~1:

  4b8df38 --- 5cc69d1 <- HEAD, main        e717cb8 (mo coi, chi con trong reflog)
  Index = 5cc69d1 (khong co thay doi staged)
  Working Dir = ma divide loi van con -> "modified, not staged"

SAU khi sua code va commit lai:

  4b8df38 --- 5cc69d1 --- 8631d56 <- HEAD, main
```

## 4. Trường hợp 2 - Sửa đổi công cộng: `git revert`

### 4.1. Tạo commit lỗi A và commit B bình thường

Commit A đổi công thức `multiply` thành `a + b` (sai). Sau đó commit B thêm `README.md` để commit lỗi KHÔNG còn là commit cuối, giống tình huống thực tế: lỗi đã nằm sâu trong lịch sử, đã có người làm tiếp phía trên.

```
$ sed -i 's/return a \* b/return a + b/' calculator.py

$ git diff
diff --git a/calculator.py b/calculator.py
index 4e9244b..c6f651b 100644
--- a/calculator.py
+++ b/calculator.py
@@ -7,7 +7,7 @@ def subtract(a, b):
 
 
 def multiply(a, b):
-    return a * b
+    return a + b
 
 
 def divide(a, b):

$ git commit -m "feat: doi cong thuc multiply"
[main 3c27d09] feat: doi cong thuc multiply
 1 file changed, 1 insertion(+), 1 deletion(-)

$ python -c "import calculator as c; print(\"multiply(6, 3) =\", c.multiply(6, 3), \"(sai, ky vong 18)\")"
multiply(6, 3) = 9 (sai, ky vong 18)

$ git commit -m "docs: them README"
[main 00841ae] docs: them README
 1 file changed, 3 insertions(+)
 create mode 100644 README.md

$ git log --oneline -n 5
00841ae docs: them README
3c27d09 feat: doi cong thuc multiply
8631d56 feat: them ham divide co kiem tra chia cho 0
5cc69d1 feat: them ham multiply
4b8df38 feat: them ham add va subtract
```

Commit lỗi là `3c27d09`, nằm dưới commit `00841ae`.

### 4.2. Đảo ngược commit lỗi bằng `git revert`

```
$ git revert 3c27d09 --no-edit
[main a93b9ee] Revert "feat: doi cong thuc multiply"
 Date: Fri Oct 2 12:48:45 2026 +0700
 1 file changed, 1 insertion(+), 1 deletion(-)
```

`--no-edit` giữ thông điệp tự sinh, không mở trình soạn thảo.

### 4.3. Kiểm tra đúng theo đề

```
$ git log --oneline -n 5
a93b9ee Revert "feat: doi cong thuc multiply"
00841ae docs: them README
3c27d09 feat: doi cong thuc multiply
8631d56 feat: them ham divide co kiem tra chia cho 0
5cc69d1 feat: them ham multiply
```

Đây là "ảnh chụp lịch sử commit" chứng minh đã tạo commit revert: commit `Revert "feat: doi cong thuc multiply"` nằm trên đầu nhánh, còn commit lỗi `3c27d09` vẫn còn nguyên trong lịch sử.

```
$ git show --stat HEAD
commit a93b9eedffda9c0f6dec4f05043a5ca3ca1c7609
Author: Lê Toàn <trinhhanh261293@gmail.com>
Date:   Fri Oct 2 12:48:45 2026 +0700

    Revert "feat: doi cong thuc multiply"
    
    This reverts commit 3c27d09a1b2cb215d35cad567f856565a5d88202.

 calculator.py | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

$ git show HEAD
commit a93b9eedffda9c0f6dec4f05043a5ca3ca1c7609
Author: Lê Toàn <trinhhanh261293@gmail.com>
Date:   Fri Oct 2 12:48:45 2026 +0700

    Revert "feat: doi cong thuc multiply"
    
    This reverts commit 3c27d09a1b2cb215d35cad567f856565a5d88202.

diff --git a/calculator.py b/calculator.py
index c6f651b..4e9244b 100644
--- a/calculator.py
+++ b/calculator.py
@@ -7,7 +7,7 @@ def subtract(a, b):
 
 
 def multiply(a, b):
-    return a + b
+    return a * b
 
 
 def divide(a, b):

$ git log -n 1
commit a93b9eedffda9c0f6dec4f05043a5ca3ca1c7609
Author: Lê Toàn <trinhhanh261293@gmail.com>
Date:   Fri Oct 2 12:48:45 2026 +0700

    Revert "feat: doi cong thuc multiply"
    
    This reverts commit 3c27d09a1b2cb215d35cad567f856565a5d88202.

$ git status
On branch main
nothing to commit, working tree clean

$ python -c "import calculator as c; print(\"multiply(6, 3) =\", c.multiply(6, 3), \"(dung tro lai)\")"
multiply(6, 3) = 18 (dung tro lai)

$ git log --oneline --graph --decorate -n 8
* a93b9ee (HEAD -> main) Revert "feat: doi cong thuc multiply"
* 00841ae docs: them README
* 3c27d09 feat: doi cong thuc multiply
* 8631d56 feat: them ham divide co kiem tra chia cho 0
* 5cc69d1 feat: them ham multiply
* 4b8df38 feat: them ham add va subtract
```

Giải thích:

- `git show HEAD`: diff của commit revert là diff NGƯỢC của commit lỗi (`a + b` đổi lại thành `a * b`).
- Thân thông điệp có dòng `This reverts commit <hash>.` ghi rõ commit bị đảo ngược, tạo dấu vết kiểm toán.
- Revert chỉ tác động vào dòng của `multiply`; `README.md` (commit B) và hàm `divide` (commit sửa ở Trường hợp 1) không bị ảnh hưởng, nên không có conflict.
- Chạy lại chương trình: `multiply(6, 3) = 18`, phép nhân đúng trở lại.

### 4.4. Sơ đồ ASCII trước và sau revert

```
TRUOC revert:

  5cc69d1 --- 8631d56 --- 3c27d09 --- 00841ae <- HEAD, main
                          (A: loi)    (B: README)

SAU git revert 3c27d09 --no-edit:

  5cc69d1 --- 8631d56 --- 3c27d09 --- 00841ae --- a93b9ee <- HEAD, main
                          (A: loi)    (B: README)  Revert "..."
                                                   (diff nguoc cua A)

  Khong co commit nao bi xoa hay doi hash: lich su chi duoc NOI DAI them.
```

## 5. Bổ sung: `git reset --soft HEAD~1` trên nhánh `demo-soft`

Nhánh thử `demo-soft` tạo từ `main`, thêm một commit `power`, rồi dùng `--soft` để thấy thay đổi nằm ở vùng Staged:

```
$ git switch -c demo-soft
Switched to a new branch 'demo-soft'

$ git commit -m "feat: them ham power"
[demo-soft 43c3ca4] feat: them ham power
 1 file changed, 4 insertions(+)

$ git log --oneline -n 3
43c3ca4 feat: them ham power
a93b9ee Revert "feat: doi cong thuc multiply"
00841ae docs: them README

$ git reset --soft HEAD~1

$ git log --oneline -n 3
a93b9ee Revert "feat: doi cong thuc multiply"
00841ae docs: them README
3c27d09 feat: doi cong thuc multiply

$ git status
On branch demo-soft
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	modified:   calculator.py


$ git diff --staged
diff --git a/calculator.py b/calculator.py
index 4e9244b..47a5ff5 100644
--- a/calculator.py
+++ b/calculator.py
@@ -14,3 +14,7 @@ def divide(a, b):
     if b == 0:
         raise ValueError("Khong the chia cho 0")
     return a / b
+
+
+def power(a, b):
+    return a ** b
```

Khác với `--mixed` (Modified, chưa staged), `--soft` giữ thay đổi ở trạng thái Staged ("Changes to be committed"), sẵn sàng `git commit` lại ngay (hữu ích để gộp hoặc sửa thông điệp commit). Sau đó commit lại và quay về `main`:

```
$ git commit -m "feat: them ham power (commit lai sau soft reset)"
[demo-soft 558b9b5] feat: them ham power (commit lai sau soft reset)
 1 file changed, 4 insertions(+)

$ git switch main
Switched to branch 'main'

$ git log --oneline -n 6
a93b9ee Revert "feat: doi cong thuc multiply"
00841ae docs: them README
3c27d09 feat: doi cong thuc multiply
8631d56 feat: them ham divide co kiem tra chia cho 0
5cc69d1 feat: them ham multiply
4b8df38 feat: them ham add va subtract

$ git status
On branch main
nothing to commit, working tree clean

$ git branch -v
  demo-soft 558b9b5 feat: them ham power (commit lai sau soft reset)
* main      a93b9ee Revert "feat: doi cong thuc multiply"
```

Lịch sử `main` không đổi (vẫn đầu nhánh là commit Revert), working tree sạch. Nhánh `demo-soft` chỉ là nhánh thử, bài nộp dùng `calculator.py` của `main`.

## 6. So sánh `git reset --soft` / `--mixed` / `--hard`

Ba chế độ đều di chuyển con trỏ nhánh (HEAD) về commit đích; khác nhau ở việc có đồng bộ lại Index và Working Directory hay không.

| Chế độ | HEAD (con trỏ nhánh) | Index (staging) | Working Directory | Thay đổi của commit bị bỏ nằm ở đâu |
|--------|----------------------|-----------------|-------------------|--------------------------------------|
| `--soft` | Di chuyển về commit đích | Giữ nguyên | Giữ nguyên | Staged (Changes to be committed) |
| `--mixed` (mặc định) | Di chuyển về commit đích | Đặt lại theo commit đích | Giữ nguyên | Modified, unstaged (Changes not staged) |
| `--hard` | Di chuyển về commit đích | Đặt lại theo commit đích | Đặt lại theo commit đích | MẤT hoàn toàn khỏi cây làm việc (chỉ cứu được commit qua reflog) |

Đề bài cấm `--hard` ở Trường hợp 1 vì nó ghi đè Working Directory, làm mất mã nguồn hiện hành chưa commit. Trong toàn bộ bài này không có lệnh `git reset --hard` nào được chạy.

## 7. Khác biệt khi làm việc cá nhân và làm việc nhóm

### 7.1. Bản chất hai cơ chế

- `git reset` DI CHUYỂN con trỏ nhánh về một commit cũ, nghĩa là VIẾT LẠI lịch sử: các commit phía sau bị gỡ khỏi nhánh (hash "biến mất" khỏi `git log` của nhánh, chỉ còn trong reflog cục bộ một thời gian).
- `git revert` TẠO COMMIT MỚI có nội dung đối lập với commit cũ. Lịch sử cũ không bị chạm vào; nhánh chỉ dài thêm.

### 7.2. Làm việc cá nhân (commit chưa push)

Reset an toàn và gọn: chưa ai khác thấy commit lỗi, nên có thể gỡ nó khỏi lịch sử, sửa code, rồi commit lại sạch sẽ (đúng Trường hợp 1). Không cần "dấu vết" cho một sai sót chỉ tồn tại trên máy mình.

### 7.3. Làm việc nhóm (commit đã nằm trên nhánh chung)

Nếu đã push commit lỗi lên nhánh chung (ví dụ `main`) và đồng đội đã `pull`:

- Dùng reset ở máy mình sẽ làm lịch sử local lệch với remote (non-fast-forward). Muốn đẩy lên phải `git push --force`, ghi đè lịch sử chung.
- Đồng đội đã có commit lỗi trong máy sẽ bị lệch: nhánh của họ chứa commit mà remote không còn, lần `pull` kế tiếp sinh conflict hoặc merge lộn xộn, thậm chí commit lỗi bị đẩy ngược lên lại.
- Commit của người khác dựa trên commit bị xóa có thể bị mất hoặc phải rebase lại.
- Mất dấu vết kiểm toán: không còn bằng chứng lỗi từng tồn tại và đã được sửa.

Dùng revert thì ngược lại:

- Chỉ thêm commit mới, nên đẩy lên bằng `git push` thường (fast-forward), không cần force.
- Đồng đội `git pull` bình thường, lịch sử của họ là tiền tố của lịch sử remote.
- Có dấu vết kiểm toán rõ ràng: ai đảo ngược, khi nào, đảo ngược commit nào (`This reverts commit <hash>.`).
- Hoàn tác được cả việc đảo ngược này (revert của revert) nếu cần.

### 7.4. Bảng tiêu chí chọn lệnh

| Tình huống | Nên dùng | Lý do |
|------------|----------|-------|
| Commit lỗi chưa push, chỉ mình biết | `git reset --mixed HEAD~1` | Gỡ commit, giữ code để sửa; lịch sử sạch |
| Muốn gộp/sửa thông điệp các commit local gần nhất | `git reset --soft HEAD~N` rồi `git commit` | Thay đổi nằm sẵn ở Staged |
| Muốn bỏ hẳn thay đổi chưa commit (chủ động, chấp nhận mất) | `git reset --hard` | Đồng bộ cả Index và Working Directory; nguy hiểm, cần chắc chắn |
| Commit lỗi đã push lên nhánh chung | `git revert <hash>` | Không viết lại lịch sử, không cần force push |
| Commit lỗi nằm sâu, phía trên đã có commit của người khác | `git revert <hash>` | Chỉ đảo ngược đúng commit đó, giữ nguyên các commit sau |
| Cần bằng chứng kiểm toán (production, release) | `git revert <hash>` | Lịch sử ghi lại cả lỗi lẫn bản sửa |
| Hotfix khẩn cấp trên `main` đã deploy | `git revert <hash>` | An toàn, nhanh, không ảnh hưởng đồng đội |

Quy tắc chung: lịch sử CHƯA chia sẻ thì có thể reset; lịch sử ĐÃ chia sẻ thì chỉ revert.

## 8. Lưu ý nâng cao

### 8.1. Revert một merge commit cần `-m`

Merge commit có hai parent nên Git không biết nên đảo ngược theo nhánh nào. Phải chỉ định parent cần giữ làm mainline:

```
git revert -m 1 <hash-merge-commit>
```

`-m 1` nghĩa là giữ parent thứ nhất (thường là nhánh nhận merge, ví dụ `main`) và đảo ngược các thay đổi mà nhánh được merge vào đã mang tới. Lưu ý: sau khi revert merge, muốn merge lại cùng nhánh đó phải revert commit revert trước, nếu không Git coi các thay đổi đã nằm trong lịch sử và bỏ qua. (Mục này là kiến thức tham khảo, chưa chạy trong bài thực hành.)

### 8.2. Revert bị conflict

Nếu các commit phía sau đã sửa cùng vùng code với commit bị revert, Git dừng lại và báo conflict. Cách xử lý:

```
git revert <hash>          # bao CONFLICT, dung giua chung
# mo file bi conflict, sua cac dau <<<<<<<, =======, >>>>>>>
git add <file-da-sua>
git revert --continue      # hoan tat, tao commit Revert
# hoac huy bo: git revert --abort
```

Trong bài này revert không conflict vì commit B chỉ thêm `README.md` và hàm `divide` nằm ở vùng khác với dòng bị đổi của `multiply`. (Mục này là kiến thức tham khảo, chưa chạy trong bài thực hành.)

### 8.3. Cứu khi reset nhầm bằng reflog

`git reflog` ghi lại mọi lần HEAD di chuyển ở local, kể cả sau reset. Thực hành trên nhánh `demo-soft`: cố ý reset nhầm commit `power`, rồi phục hồi.

```
$ git switch demo-soft
Switched to branch 'demo-soft'

$ git reset --mixed HEAD~1
Unstaged changes after reset:
M	calculator.py

$ git log --oneline -n 3
a93b9ee Revert "feat: doi cong thuc multiply"
00841ae docs: them README
3c27d09 feat: doi cong thuc multiply

$ git reflog -n 5
a93b9ee HEAD@{0}: reset: moving to HEAD~1
558b9b5 HEAD@{1}: checkout: moving from main to demo-soft
a93b9ee HEAD@{2}: checkout: moving from demo-soft to main
558b9b5 HEAD@{3}: commit: feat: them ham power (commit lai sau soft reset)
a93b9ee HEAD@{4}: reset: moving to HEAD~1

$ git reset --mixed HEAD@{1}

$ git log --oneline -n 3
558b9b5 feat: them ham power (commit lai sau soft reset)
a93b9ee Revert "feat: doi cong thuc multiply"
00841ae docs: them README

$ git status
On branch demo-soft
nothing to commit, working tree clean
```

Commit `558b9b5` quay lại đầu nhánh chỉ bằng cách trỏ HEAD về `HEAD@{1}` trong reflog. Điều kiện: commit đã từng tồn tại ở local và reflog chưa hết hạn (mặc định khoảng 90 ngày cho mục còn tham chiếu được, 30 ngày cho mục không còn tham chiếu). Thay đổi CHƯA commit bị `reset --hard` ghi đè thì reflog không cứu được, đây là lý do nên tránh `--hard`.

## 9. Bảng đối chiếu yêu cầu đề bài

| Yêu cầu | Đã làm | Bằng chứng |
|---------|--------|------------|
| TH1: lùi HEAD 1 commit | Có, `git reset --mixed HEAD~1` | Mục 3.2, 3.3: log chỉ còn tới `5cc69d1` |
| TH1: thay đổi ở Working Directory dạng Modified (unstaged) | Có | `git status` mục 3.3: "Changes not staged for commit", `git diff` còn mã lỗi |
| TH1: không dùng reset hard, không mất mã | Đúng | Không có `--hard` trong `terminal_log.txt`; `git diff` vẫn thấy `divide` lỗi |
| TH2: tạo commit mới đối lập để đẩy lên remote an toàn | Có, `git revert 3c27d09 --no-edit` | Mục 4.2; commit `a93b9ee`; không viết lại lịch sử, không cần force |
| TH2: commit mới tiêu đề bắt đầu bằng `Revert ...` trên đầu nhánh | Có | `git log --oneline -n 5` mục 4.3 |
| Kiểm tra `git log --oneline -n 5` | Có, cả hai trường hợp | Mục 3.3, 3.4, 4.1, 4.3 |
| README trình bày chi tiết hai trường hợp | Có | Mục 3 và 4 |
| Giải thích khác biệt cá nhân so với nhóm | Có | Mục 7 |
| "Ảnh chụp" lịch sử commit chứng minh commit revert | Có, dưới dạng output text nguyên văn | Mục 4.3 (`git log --oneline`, `--graph`, `git show`) |
| Đường dẫn `homework/session_04/ex5/` | Có | Cây thư mục bài nộp |

Ghi chú về "đẩy lên remote": đề yêu cầu trình bày cách đảo ngược an toàn để đẩy lên remote; bài nộp này không tạo repo GitHub và không push (theo hướng dẫn của buổi thực hành). Lệnh sẽ dùng khi có remote là `git push origin main` thường, không cần `--force`, vì revert chỉ nối dài lịch sử.

## 10. Kết luận

- `git reset` là công cụ sửa lịch sử CỤC BỘ: nhanh, sạch, nhưng viết lại lịch sử nên chỉ dùng cho commit chưa chia sẻ. Ba chế độ khác nhau ở độ "sâu" đồng bộ: `--soft` chỉ đổi HEAD, `--mixed` đổi thêm Index, `--hard` đổi cả Working Directory (và có thể làm mất mã).
- `git revert` là cách sửa lịch sử CÔNG CỘNG: thêm một commit đối lập, giữ nguyên toàn bộ lịch sử cũ, không cần force push, có dấu vết kiểm toán. Đây là lựa chọn mặc định cho nhánh chung.
- Khi lỡ tay, `git reflog` giúp phục hồi các commit đã bị reset khỏi nhánh.

Kết quả thực hành: Trường hợp 1 đưa HEAD từ `e717cb8` về `5cc69d1` với mã `divide` lỗi còn trong Working Directory, sau đó commit lại đúng thành `8631d56`. Trường hợp 2 tạo commit `a93b9ee Revert "feat: doi cong thuc multiply"` đảo ngược commit lỗi `3c27d09`, phép nhân trở lại đúng (`18`).

## 11. Cây thư mục bài nộp

```
DevOps-Session4-bai5/
|-- README.md
`-- homework/
    `-- session_04/
        `-- ex5/
            |-- README.md          (báo cáo này)
            |-- terminal_log.txt   (toàn bộ lệnh và output thật)
            `-- calculator.py      (bản cuối trên nhánh main)
```
