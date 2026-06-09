# 작업 레포용 템플릿 키트

이 폴더의 `.github/`를 **각 작업 레포(`dokkaebi-app`, `dokkaebi-server`, `dokkaebi-ai`, `dokkaebi-infra`) 루트에 복사**하면, 네 레포가 동일한 이슈/PR 템플릿을 쓰게 됩니다.

## 들어있는 것
```
.github/
├── ISSUE_TEMPLATE/
│   ├── feature.yml   ✨ 작업(기능/세부) 이슈
│   ├── bug.yml       🐞 버그 리포트
│   └── config.yml    빈 이슈 비활성 + 상위 Task/가이드 링크
└── PULL_REQUEST_TEMPLATE.md
```

## 배포 방법 (레포마다 1회)

각 작업 레포를 클론한 뒤, 이 키트의 `.github` 폴더를 레포 루트에 복사:

```bash
# 예: dokkaebi-app 에 적용
cp -r templates/work-repo/.github  <dokkaebi-app 클론 경로>/
cd <dokkaebi-app 클론 경로>
git checkout -b chore/issue-templates/v1
git add .github
git commit -m "chore: 이슈/PR 템플릿 추가"
git push -u origin chore/issue-templates/v1
# → dev 로 PR
```

`dokkaebi-server`, `dokkaebi-ai`, `dokkaebi-infra` 도 동일하게 반복.

## 참고 — 왜 작업 레포마다 따로 두나?
- `.github` 레포에는 **Task(엄브렐러)** 템플릿만 둔다.
- 작업 레포가 **자기 템플릿을 가지면** `.github` 레포의 기본값을 상속하지 않으므로, 작업 레포에서는 Task 템플릿이 안 뜨고 **작업/버그 템플릿만** 보인다. (역할 분리)
- PR 템플릿은 `.github` 레포 것이 조직 기본값으로 적용되지만, 명시적으로 두고 싶으면 이 키트의 것을 복사해도 된다.
