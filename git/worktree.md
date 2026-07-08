# git worktree - clone 없이 한 저장소를 여러 폴더에 동시 체크아웃

> 한 줄 요약: `git worktree add <폴더> <브랜치>` 하면 **같은 `.git` 저장소를 공유하는 작업 폴더가 하나 더** 생긴다. 이후엔 특별한 명령 없이 그냥 폴더처럼 쓰면 됨 — "clone인데 히스토리를 공유해서 가볍고, 커밋이 실시간 공유되는 버전". 핵심 함정: **같은 브랜치는 두 워크트리에 동시 체크아웃 불가**, 본체 저장소를 지우면 워크트리도 죽는다.

## 언제 쓰나
- 기능 브랜치 작업 중인데 **다른 브랜치(리뷰할 MR, 핫픽스)를 stash/checkout 없이** 옆에서 열어보고 싶을 때
- 두 브랜치의 코드를 **나란히 비교**하거나, 각각 IDE로 열어 동시에 보고 싶을 때
- AI 에이전트/스크립트가 병렬로 파일을 수정할 때 서로 격리시키고 싶을 때

## 사용 예시

```bash
# 생성: ../api-review 폴더에 release 브랜치 체크아웃
git worktree add ../api-review release

# 새 브랜치를 만들면서 생성
git worktree add -b hotfix/9999 ../api-hotfix origin/release

# 목록 (본체 + 워크트리 전부)
git worktree list
# C:/work/api          ca3b226 [feat/my-work]     ← 본체(main worktree)
# C:/work/api-review   f53c769 [release]          ← linked worktree

# 정리
git worktree remove ../api-review     # 폴더 + 관리 정보 삭제
git worktree prune                    # 폴더를 수동으로 지웠을 때 남은 찌꺼기 정리
```

### GitLab MR / GitHub PR을 브랜치 이름 없이 가져와서 열기
서버가 MR/PR마다 숨은 ref를 노출한다 — **브랜치 이름을 몰라도 번호로 접근** 가능:

```bash
# GitLab: refs/merge-requests/<iid>/head (merge 결과 미리보기는 /merge)
git fetch origin refs/merge-requests/15688/head:mr-15688
git worktree add ../api-mr-15688 mr-15688

# GitHub: refs/pull/<번호>/head
git fetch origin refs/pull/123/head:pr-123
```

리뷰 끝나면 `git worktree remove ../api-mr-15688` + `git branch -D mr-15688`로 흔적 없이 정리.

## clone과 뭐가 다른가 (공유 범위)

| | worktree | clone 하나 더 |
|---|---|---|
| 저장소(`.git`) | **하나를 공유** — 워크트리의 `.git`은 디렉토리가 아니라 본체를 가리키는 파일 1개 (`gitdir: .../.git/worktrees/<이름>`) | 통째로 복제 (디스크·다운로드 2배) |
| 커밋/브랜치 | 한쪽에서 커밋하면 **다른 쪽에서 즉시 조회됨** (push/pull 불필요) | push → pull 해야 보임 |
| `git fetch` | 한 번이면 전체 반영 | 각자 fetch |
| remote 설정·stash | 공유 (stash도 `refs/stash`라 공유됨) | 독립 |
| HEAD·index(스테이징) | **워크트리별 독립** — 그래서 서로 다른 브랜치를 체크아웃할 수 있는 것 | 독립 |
| 수명 | **본체에 종속** — 본체 삭제 시 같이 죽음 | 완전 독립 |

## ⚠️ 함정
1. **같은 브랜치를 두 워크트리에서 동시 체크아웃 불가.** `fatal: '...' is already used by worktree ...` 에러. 같은 저장소라 index/HEAD 충돌을 막기 위한 안전장치 (`--force`로 뚫을 수 있지만 하지 말 것).
2. **IDE는 별도 프로젝트다.** 폴더가 다르니 IntelliJ 등에서 워크트리마다 창을 따로 열고, 인덱싱도 각자 한다. clone 2개일 때와 동일 — worktree라고 IDE가 특별 취급하지 않는다.
3. **폴더를 그냥 `rm`으로 지우면 관리 정보가 남는다.** `git worktree remove`를 쓰거나, 이미 지웠으면 `git worktree prune`. 남아 있으면 그 브랜치가 계속 "체크아웃 중"으로 잠긴다.
4. 빌드 캐시(`target/`, `node_modules/` 등)는 공유 안 되므로 워크트리마다 새로 빌드해야 한다 — "디스크 절약"은 git 히스토리 얘기지 빌드 산출물 얘기가 아님.

## 💡 판단 기준
- **"지금 작업을 건드리지 않고 다른 브랜치 코드를 봐야 한다" → stash+checkout 말고 worktree.** 실제 케이스: 기능 브랜치에서 작업 중에 동료 MR(15688)을 코드로 분석해야 했는데, worktree로 MR ref를 옆 폴더에 체크아웃하니 내 브랜치의 미커밋 상태를 전혀 건드리지 않았고, 저장소가 공유라 내 폴더에서도 `git log mr-15688` 조회가 바로 됐다.
- 잠깐 diff만 볼 거면 worktree까지 갈 필요 없이 `git diff origin/develop...대상브랜치`로 충분. **"파일 트리를 탐색/실행해야 하는가"**가 worktree를 꺼내는 기준.

## 참고
- git worktree 공식 문서: https://git-scm.com/docs/git-worktree
- GitLab MR refs: https://docs.gitlab.com/ee/user/project/merge_requests/reviews/#checkout-merge-requests-locally-through-the-head-ref

---
학습 날짜: 2026-07-08
계기: 기능 브랜치 작업 중 동료의 스트리밍 채팅 MR을 코드 레벨로 분석할 일이 생겨, stash 없이 MR ref를 worktree로 체크아웃해 병행 분석하며 clone과의 차이(저장소 공유·실시간 커밋 공유·동일 브랜치 금지)를 정리.
