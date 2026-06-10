# 다이어그램 PNG (슬라이드용)

> [시퀀스_함수콜링.md](../시퀀스_함수콜링.md)의 Mermaid를 PNG로 export한 것. 발표 슬라이드 삽입용.

| 파일 | 내용 |
|---|---|
| `00-function-map.png` | 전체 함수맵 (발표 1장) |
| `01-npc-dialogue.png` | NPC 대화 풀 체인 시퀀스 |

## 재생성 (Mermaid → PNG)

```bash
# 요구: Node.js (npx). 헤드리스 Chromium 자동 사용.
cd .github/report/images

# .mmd 소스에서 PNG 렌더 (한글·고해상)
npx -y @mermaid-js/mermaid-cli@latest \
  -i 00-function-map.mmd -o 00-function-map.png \
  -p puppeteer.json -t neutral -b white --scale 3
```

- `.mmd` = Mermaid 소스 (문서 본문의 ```mermaid 블록과 동일)
- 새 다이어그램 추가 시: 문서의 mermaid 블록을 `*.mmd`로 저장 후 위 명령
- 옵션: `-t neutral`(테마) · `-b white`(배경) · `--scale 3`(해상도)
