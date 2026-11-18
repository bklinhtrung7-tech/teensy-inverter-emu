# 03 – Git workflow và bài tập

Nguyên tắc: **bạn tự gõ lệnh Git**. Claude Code chỉ được xem (`status`, `diff`, `log`), đề xuất commit message và giải thích. Quyền này đã khóa trong `.claude/settings.json` (deny `git push`, `reset --hard`, `clean`).

## 1. Quy ước

### Nhánh (GitHub Flow rút gọn)

```
main  ──●──────────●────────────●──────►   (luôn build được, chỉ vào qua PR)
         \        / \          /
          ●──●──●    ●──●──●──●            feature/p2-lpspi-driver, feature/p4-edma ...
```

- `main` được bảo vệ trên GitHub: bắt buộc PR, bắt buộc CI xanh (từ P6).
- Tên nhánh: `feature/p<N>-<topic>`, `fix/<topic>`, `docs/<topic>`, `exp/<topic>` (thử nghiệm, có thể bỏ).
- Mỗi phase = 1 **GitHub Milestone**; mỗi checkbox lớn = 1 **Issue**; PR ghi `Closes #12`.

### Commit (Conventional Commits)

```
<type>(<scope>): <mô tả ngắn, tiếng Anh, thì hiện tại>

<thân: vì sao thay đổi, số đo nếu có>

Refs: #12
```

`type`: `feat`, `fix`, `refactor`, `test`, `docs`, `build`, `ci`, `chore`, `perf`
`scope`: `spi`, `dac`, `dma`, `pit`, `siggen`, `can`, `com`, `ctrl`, `os`, `boot`, `tools`, `canoe`…

Ví dụ: `perf(dma): move tx buffer to DTCM, drop cache clean (CPU 7% -> 3%)`

### Tag & release

- Annotated tag theo SemVer: `v0.3.0` khi xong P2… `v1.0.0` khi xong P6.
- GitHub Release đính kèm `firmware.hex` + `firmware.map` + tóm tắt số đo.

## 2. Thiết lập một lần

```bash
git config --global user.name  "unknown"
git config --global user.email "<email dùng cho GitHub>"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global core.editor "code --wait"
git config --global core.autocrlf input        # Linux/macOS; Windows: true
ssh-keygen -t ed25519 -C "<email>" && gh auth login   # chọn SSH
```

## 3. Bài tập (gắn với phase)

| # | Phase | Bài tập | Lệnh chính | Kiểm tra |
|---|---|---|---|---|
| 1 | P0 | Tạo repo, commit bộ file khởi tạo | `git init`, `git add -p`, `git commit`, `gh repo create --source . --private --push` | `git log --oneline` có 1 commit; repo trên GitHub |
| 2 | P0 | Bảo vệ `main`, tạo Milestones P0–P6 và Issues từ roadmap | `gh api` hoặc web UI, `gh issue create` | Không push thẳng được vào `main` |
| 3 | P1 | Nhánh + PR đầu tiên (startup + linker) | `git switch -c feature/p1-startup`, `git push -u origin HEAD`, `gh pr create` | PR có mô tả theo template, merge bằng **squash** |
| 4 | P1 | `.gitignore` sai rồi sửa: lỡ commit `build/` → gỡ khỏi Git nhưng giữ trên đĩa | `git rm -r --cached build`, `git check-ignore -v` | `build/` không còn trong `git ls-files` |
| 5 | P2 | Sửa commit: thêm file quên, sửa message | `git commit --amend`, `git commit --fixup <sha>` | Lịch sử sạch trước khi push |
| 6 | P3 | **`git bisect`**: cố ý tạo một chuỗi 6–8 commit, một commit giữa làm sai lệch pha (ví dụ đổi `0x55555555` thành `0x55550000`). Viết script test host trả mã lỗi rồi để Git tự tìm | `git bisect start`, `git bisect bad/good`, `git bisect run ctest --test-dir build-host` | Git chỉ ra đúng commit lỗi |
| 7 | P3 | **Merge conflict có chủ ý**: 2 nhánh cùng sửa `SigGen_Cfg.h` (một nhánh đổi fs, một nhánh đổi kích thước LUT) | `git merge`, sửa conflict, `git mergetool` (VS Code) | Build và test xanh sau khi gộp |
| 8 | P4 | **Interactive rebase**: nhánh DMA có 15 commit "wip" → gộp còn 3–4 commit có nghĩa trước khi mở PR | `git rebase -i main` (squash, fixup, reword), `git push --force-with-lease` | Mỗi commit sau rebase đều build được |
| 9 | P5 | **Submodule** FreeRTOS-Kernel ghim theo tag; clone lại repo ở thư mục khác để thử | `git submodule add`, `git -C third_party/FreeRTOS-Kernel checkout <tag>`, `git clone --recurse-submodules` | Clone mới build được ngay |
| 10 | P6 | **Release**: annotated tag + GitHub Release có `.hex` | `git tag -a v1.0.0`, `git push origin v1.0.0`, `gh release create` | Release có file đính kèm |
| 11 | P6 | **CI**: GitHub Actions build firmware + chạy host test + format check; PR đỏ thì không merge được | `.github/workflows/ci.yml` (nhờ Claude viết – L4) | Badge xanh trong README |
| 12 | Bất kỳ | **Cứu dữ liệu**: cố ý `reset --hard` mất commit (tự gõ, Claude bị chặn lệnh này) → tìm lại; so sánh `revert` và `reset`; `stash` khi đang dở việc mà phải sửa gấp `main`; `cherry-pick` một bản fix sang nhánh khác | `git reflog`, `git revert`, `git stash push -m`, `git cherry-pick` | Khôi phục được commit đã "mất" |

### Bài thêm (khi đã quen)

- **`git worktree`**: mở 2 thư mục làm việc từ 1 repo (ví dụ `exp/interp-lut` và `feature/p6-can`), chạy 2 phiên Claude Code song song mà không giẫm chân nhau.
- **Hook `pre-commit`**: chặn commit nếu `clang-format --dry-run --Werror` báo lỗi.
- **`git blame` / `git log -L`**: truy lịch sử một hàm (`git log -L :SigGen_FillBlock:app/SigGen/SigGen.c`).
- **Signed commits** (SSH signing) – liên hệ ISO/SAE 21434 về toàn vẹn chuỗi cung ứng phần mềm.

## 4. Vòng lặp mỗi buổi

```bash
git switch main && git pull
git switch -c feature/p2-dac-driver        # hoặc git switch <nhánh đang làm>
# ... code, build, test ...
git status && git diff                     # tự đọc diff trước
# trong Claude Code: /review
git add -p                                 # stage từng phần, không 'git add .' mù
git commit                                 # message theo quy ước
git push -u origin HEAD
# cuối phase: gh pr create → review → merge → /milestone P2 → tag
```
