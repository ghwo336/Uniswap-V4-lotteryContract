# Docker + Cloudflare Tunnel 배포

공개 포트를 열지 않고 Cloudflare Tunnel 로만 외부에 노출합니다.
컨테이너는 `127.0.0.1:3007` 에만 바인딩되므로 호스트 밖에서 직접 접근할 수 없습니다.

```
브라우저 → Cloudflare 엣지 → cloudflared(터널) → 127.0.0.1:3007 → nginx(컨테이너) → dist/
```

## 1. 빌드 & 기동

```bash
cd web && npm run sync-env && cd ..
docker compose up -d --build
```

`sync-env` 는 `deployments/sepolia.json` 에서 `web/.env` 를 만듭니다.

**중요:** Vite 는 `VITE_*` 값을 빌드 타임에 번들로 인라인합니다.
컨트랙트를 재배포하면 컨테이너 재시작이 아니라 **재빌드**가 필요합니다.
`.env` 가 없거나 훅 주소가 비어 있으면 Dockerfile 이 빌드를 실패시킵니다.
(조용히 데모 모드 이미지가 만들어지는 걸 막기 위함입니다.)

## 2. 터널 연결 (Cloudflare 대시보드)

실행 중인 터널은 `고속터미널`(`f638c31a-0452-4087-a9a2-37f53b8e13c1`) 입니다.

이 터널은 **remotely-managed** 방식입니다.
launchd 가 `TUNNEL_TOKEN` 환경변수로 실행하므로 (`/Library/LaunchDaemons/com.cloudflare.cloudflared.plist`),
ingress 규칙은 로컬 `config.yml` 이 아니라 Cloudflare 에 저장됩니다.
`/var/root/.cloudflared/config.yml` 이 비어 있는 것이 정상이며, **이 파일을 만들면 안 됩니다.**
재시작 시 이 파일이 채택되어 기존 호스트(baydev, chainlens 등)가 전부 끊길 수 있습니다.

대시보드에서 추가합니다.

> Zero Trust → Networks → Tunnels → `고속터미널` → Public Hostname → Add a public hostname

| 항목 | 값 |
|---|---|
| Subdomain | `lottery` |
| Domain | `pelicanlab.dev` |
| Type | `HTTP` |
| URL | `127.0.0.1:3007` |

DNS CNAME 은 대시보드가 자동으로 만듭니다. `cloudflared tunnel route dns` 는 필요 없습니다.
설정은 즉시 푸시되므로 데몬 재시작도 필요 없습니다.

nginx vhost 나 Let's Encrypt 인증서도 필요 없습니다.
TLS 는 Cloudflare 엣지에서 종료되고, 오리진 구간은 터널이 암호화합니다.

## 3. 확인

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:3007/healthz   # 200
curl -sI https://lottery.pelicanlab.dev | head -1                        # 200
```

## 재배포

```bash
cd web && npm run sync-env && cd ..
docker compose up -d --build
```
