# FARMETRY 공개 대시보드 (Render 정적 배포)

뷰 전용, 카메라/제어/설정 없음. 센서 값·차트·AI 메모만 공개.

## 구조

- `index.html` — 단일 HTML 대시보드 (MQTT 아닌 JSON polling)
- `data/latest.json` — farmer1이 5분마다 갱신하는 센서 스냅샷 (crop별 history + 메모)

## 배포 순서

### 1. 이 폴더를 GitHub 새 레포로

```bash
cd public/
git init
git add .
git commit -m "initial public dashboard"
gh repo create shinho-o/farmetry-public --public --source=. --push
```

### 2. Render 접속 → New → Static Site

- Repository: `shinho-o/farmetry-public`
- Branch: `main`
- Publish directory: `.`
- Build command: (비움)
- 자동 HTTPS, 무료 플랜으로 OK

배포되면 `https://farmetry-public.onrender.com` 같은 URL 생성.

### 3. farmer1에서 자동 스냅샷 publish

farmer1에 같은 레포 clone (push 권한 있는 SSH키 또는 PAT 필요):

```bash
ssh theysh0312@farmer1.local
git clone git@github.com:shinho-o/farmetry-public.git ~/farmetry-public
# (또는 HTTPS + PAT)
```

cron 등록 (`crontab -e`):

```
*/5 * * * * cd /home/theysh0312/smartfarm/ai_analyzer && /usr/bin/env bash -c "set -a; source .env; set +a; PUBLIC_REPO_DIR=/home/theysh0312/farmetry-public /usr/bin/python3 publish_public_snapshot.py" >> /tmp/public_snapshot.log 2>&1
```

5분마다 farmer1이:
1. InfluxDB에서 72h 센서 history 조회
2. `data/latest.json` 덮어쓰기
3. git commit + push
4. Render가 자동 재빌드 (30~60초)

### 4. 확인

Render URL 접속 → 브라우저 캐시 무시(Ctrl+Shift+R)하면 바로 반영됨.

## 커스터마이징

- `index.html`에서 `setInterval(loadData, 60000)`이 클라이언트 폴링 주기 (60초)
- `publish_public_snapshot.py`의 `fetch_history(hours=72)`로 보관 기간 조절
- crop별 zone 분리되면 `fetch_history(crop=...)` 추가하고 각 crop의 `history` 분리 기록

## 주의

- `data/latest.json`은 **공개됩니다**. 민감한 필드가 있으면 필터링 추가.
- API 키/토큰이 stdout에 찍히지 않도록 cron 로그 권한 주의.
- Render 무료 플랜은 빌드 분·대역폭 제한 있으니 5분 주기 이상으로.
