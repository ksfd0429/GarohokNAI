# 가호록 v13 — 웹 배포판

파일 하나(`index.html`)로 돌아가는 정적 웹앱. 서버·설치 불필요.
같은 파일이 PC 로컬 서버(`가호록시작.bat`)에서 열리면 자동으로 **서버 모드**,
깃허브 Pages 등 웹 주소로 열리면 **웹 모드**로 동작한다.

| | 서버 모드 (PC bat) | 웹 모드 (깃허브 Pages) |
|---|---|---|
| API 키 | PC의 nai_key.txt | ⚙️ 설정에서 입력 (브라우저 저장 또는 세션만) |
| 프롬프트 슬롯 | slots.json 파일 | 브라우저(localStorage) + 내보내기/가져오기 |
| 생성 이미지 | output/ 자동 저장 | 세션 갤러리 보관 → 💾 눌러야 저장 |

## 깃허브 배포 절차 (1회)

1. github.com 로그인 → **New repository** → 이름 자유, **Public** 선택 → Create
2. **Add file → Upload files** → 이 폴더의 `index.html` 업로드 → Commit
   (`.gitignore`도 같이 올려두면 실수로 키 파일이 올라가는 것을 막아준다)
3. 저장소 **Settings → Pages** → Branch: `main` / `(root)` → Save
4. 1~2분 뒤 `https://<계정명>.github.io/<저장소명>/` 접속 → 완료

> 무료 계정의 Pages는 공개 저장소 전용 — 페이지 주소를 아는 사람은 누구나 열 수 있다.
> API 키는 각자 브라우저에만 저장되므로 저장소에는 아무 비밀도 올라가지 않는다.

## 아이패드에서 쓰기

1. 사파리로 배포 주소 접속 → 공유 → **홈 화면에 추가** (앱처럼 사용)
2. **⚙️ 설정** → NAI API 키 입력 → "이 브라우저에 저장" 선택 → 적용
   (공용 기기라면 "이번 세션만" — 탭을 닫으면 키가 지워진다)
3. 외형 탭에서 🎨 생성 → 이미지는 **세션 갤러리에만 보관**됨
4. 남기고 싶은 것만 **💾 저장** → 사파리 다운로드 폴더로 들어감
   - 저장 위치는 iOS **설정 → Safari → 다운로드**에서 한 번 지정해두면 항상 그곳으로
5. PC에서 쓰던 슬롯(화풍/네거티브)을 옮기려면: PC의 `slots.json`을
   ⚙️ 설정 → **가져오기**로 넣으면 된다 (이후 브라우저에 유지됨)

## PC 크롬/엣지에서 쓰기

⚙️ 설정 → **폴더 선택**으로 저장 폴더를 지정하면 💾 저장 시 묻지 않고 그 폴더에 바로 저장된다.
(사파리·파이어폭스는 폴더 지정 미지원 → 다운로드 폴더로 저장)

## 직통 호출이 차단될 때 (프록시)

웹 모드는 브라우저에서 NovelAI API를 직접 호출한다. 만약 생성 시
"직통 호출이 차단됨(CORS)" 오류가 뜨면, 5분짜리 Cloudflare Worker 프록시를 만들어 우회한다:

1. dash.cloudflare.com → Workers & Pages → **Create Worker** → 이름 아무거나
2. 코드를 아래로 교체 → Deploy:

```js
const CORS = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Headers': 'Authorization, Content-Type, Accept',
  'Access-Control-Allow-Methods': 'POST, OPTIONS',
};
export default {
  async fetch(req) {
    if (req.method === 'OPTIONS') return new Response(null, { headers: CORS });
    const r = await fetch('https://image.novelai.net/ai/generate-image', {
      method: 'POST',
      headers: {
        'Authorization': req.headers.get('Authorization') || '',
        'Content-Type': 'application/json',
        'Accept': 'application/x-zip-compressed',
      },
      body: req.body,
    });
    const h = new Headers(CORS);
    h.set('Content-Type', r.headers.get('Content-Type') || 'application/octet-stream');
    return new Response(r.body, { status: r.status, headers: h });
  },
};
```

3. 발급된 `https://<이름>.<계정>.workers.dev` 주소를
   가호록 **⚙️ 설정 → 고급 → 프록시 URL**에 넣고 적용

키는 요청을 스쳐갈 뿐 Worker에 저장되지 않는다.

## 주의

- `nai_key.txt`류 키 파일은 **어떤 경우에도 저장소에 올리지 말 것** (.gitignore가 1차 방어)
- 세션 갤러리는 페이지를 닫으면 사라진다 — 남길 이미지는 반드시 💾 저장
- 402 오류 = 구독/Anlas 문제, 429 = 연속 요청 과다 (잠시 후 재시도)
