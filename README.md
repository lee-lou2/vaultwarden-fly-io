# Fly.io를 이용한 Vaultwarden 셀프 호스팅

몇 단계만으로 [Fly.io](https://fly.io/)에 나만의 비밀번호 관리자 [Vaultwarden](https://github.com/dani-garcia/vaultwarden)을 배포하세요.

-----

## 🚀 배포 방법

### 1\. 사전 준비

  * **Fly.io 계정 생성**: [Fly.io 웹사이트](https://fly.io/docs/hands-on/start/)에서 가입합니다.
  * **flyctl 설치 및 로그인**: 터미널에서 아래 명령어를 실행하여 Fly.io CLI를 설치하고 로그인합니다.
    ```bash
    # macOS / Linux
    curl -L https://fly.io/install.sh | sh

    # Windows
    iwr https://fly.io/install.ps1 -useb | iex
    ```
    ```bash
    flyctl auth login
    ```

### 2\. 저장소 복제 및 설정

```bash
git clone https://github.com/lee-lou2/vaultwarden-fly-io.git # 본인 저장소 주소로 변경
cd vaultwarden-fly-io
```

`fly.toml` 파일을 열고, `app`과 `DOMAIN`의 `<YOUR_APP_NAME>` 부분을 원하는 앱 이름으로 수정합니다.

```toml
app = "my-vaultwarden" # 예시: 원하는 앱 이름으로 변경
# ...
[env]
  # ...
  DOMAIN = "https://my-vaultwarden.fly.dev" # 예시: 위 app 이름에 맞춰 변경
```

### 3\. 볼륨 및 비밀(Secrets) 설정

```bash
# 데이터 저장을 위한 1GB 볼륨 생성 (무료 등급은 총 3GB까지)
flyctl volumes create vaultwarden_data --size 1

# 관리자 토큰(ADMIN_TOKEN) 설정 (자세한 발급 방법은 아래 참조)
flyctl secrets set ADMIN_TOKEN='암호화된_해시값을_여기에_붙여넣기'

# (선택) 회원가입 허용 (초대 전용으로 운영 시 불필요)
flyctl secrets set SIGNUPS_ALLOWED="true"
```

### 4\. 배포

```bash
flyctl deploy
```

배포가 완료되면 설정한 `DOMAIN` 주소 (`https://<YOUR_APP_NAME>.fly.dev`)로 접속할 수 있습니다. 관리자 페이지는 `/admin` 경로입니다.

-----

## ⚙️ 상세 설정

### 🔑 관리자 토큰(ADMIN\_TOKEN) 발급

보안을 위해 관리자 페이지는 암호화된 비밀번호를 사용합니다.

1.  **사용할 비밀번호 정하기**: 먼저 `/admin` 페이지에서 사용할 자신만의 비밀번호를 정합니다 (예: `my-super-secret-123`).

2.  **해시(Hash) 값 생성하기**: Docker를 이용해 아래 명령어를 실행하고, 1번에서 정한 비밀번호를 입력하여 암호화된 해시 값을 발급받습니다.

    ```bash
    docker run --rm -it vaultwarden/server /vaultwarden hash
    ```

3.  **Secrets 설정**: 발급받은 `$`로 시작하는 긴 해시 값을 위의 `flyctl secrets set ADMIN_TOKEN=...` 명령어에 사용합니다.

> **중요**: `/admin` 페이지에 로그인할 때는 해시 값이 아닌, **1번에서 직접 정한 원래의 비밀번호**를 사용해야 합니다.

### ✉️ 이메일(SMTP) 설정 (선택 사항)

사용자 초대, 비밀번호 재설정 등을 위해 SMTP 설정이 필요합니다. 아래는 **Google SMTP** 예시이며, `fly.toml` 파일의 `[env]` 섹션에 추가하거나 `flyctl secrets set`으로 설정할 수 있습니다.

**Google의 "앱 비밀번호"를 사용해야 합니다.** [Google 계정 2단계 인증 및 앱 비밀번호 설정 방법](https://support.google.com/accounts/answer/185833)을 참고하세요.

```toml
# fly.toml의 [env] 섹션에 추가

[env]
  # ... 기존 설정 ...
  SMTP_HOST = "smtp.gmail.com"
  SMTP_FROM = "your-email@gmail.com"
  SMTP_PORT = "587"
  SMTP_SECURITY = "starttls"
  SMTP_USERNAME = "your-email@gmail.com"
  SMTP_PASSWORD = "your-google-app-password" # 16자리 앱 비밀번호
```

보안을 위해 `fly.toml`에 직접 추가하는 것보다 `secrets`로 설정하는 것을 권장합니다.

```bash
flyctl secrets set SMTP_HOST="smtp.gmail.com" SMTP_FROM="your-email@gmail.com" # ... 나머지 변수들도 동일하게 설정
```