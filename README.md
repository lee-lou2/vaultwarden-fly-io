# Fly.io를 이용한 Vaultwarden 셀프 호스팅

몇 단계만으로 [Fly.io](https://fly.io/)에 나만의 비밀번호 관리자 [Vaultwarden](https://github.com/dani-garcia/vaultwarden)을 배포하세요.

-----

## ✅ 1. 사전 준비

  * **Fly.io 계정 생성**: [Fly.io 웹사이트](https://fly.io/docs/hands-on/start/)에서 가입합니다.

  * **Docker 설치**: 관리자 토큰 생성을 위해 [Docker](https://www.docker.com/products/docker-desktop/)가 필요합니다. 미리 설치해주세요.

  * **`flyctl` 설치 및 로그인**: 터미널에서 아래 명령어를 실행하여 Fly.io CLI를 설치하고 로그인합니다.

    ```bash
    # macOS / Linux
    curl -L https://fly.io/install.sh | sh

    # Windows
    iwr https://fly.io/install.ps1 -useb | iex
    ```

    ```bash
    flyctl auth login
    ```

## ⚙️ 2. 프로젝트 설정 및 앱 생성

1.  **저장소 복제 및 이동**

    ```bash
    git clone https://github.com/lee-lou2/vaultwarden-fly-io.git # 본인 저장소 주소로 변경
    cd vaultwarden-fly-io
    ```

2.  **Fly.io 앱 생성**
    앱 이름을 정하고 아래 명령어로 앱을 먼저 생성합니다. **앱 이름은 전 세계에서 고유해야 합니다.** 원하는 지역(예: `nrt` - 도쿄)을 지정할 수 있습니다.

    ```bash
    # <YOUR_UNIQUE_APP_NAME>을 원하는 고유한 앱 이름으로 변경하세요.
    flyctl apps create <YOUR_UNIQUE_APP_NAME> --org personal --machines
    ```

## 🛠️ 3. `fly.toml` 파일 수정

`fly.toml` 파일을 열어 아래 항목들을 **방금 생성한 앱 이름에 맞게** 수정합니다.

```toml
# fly.toml

app = "<YOUR_UNIQUE_APP_NAME>" # 2번 단계에서 생성한 앱 이름으로 변경
primary_region = "nrt"       # 2번 단계에서 선택한 지역으로 변경 (예: nrt, icn, sin)

[build]
  image = "vaultwarden/server:latest"

[env]
  # ✅ 중요: 아래 도메인 주소의 앱 이름을 본인 것으로 변경하세요.
  DOMAIN = "https://<YOUR_UNIQUE_APP_NAME>.fly.dev"
  SIGNUPS_ALLOWED = "true" # 회원가입 허용 여부 (초대 전용 시 "false"로 변경)
  WEBSOCKET_ENABLED = "true" # 웹소켓 활성화 (클라이언트-서버 실시간 동기화)

# [[services]] ... 이하 부분은 수정할 필요 없습니다.
```

## 💾 4. 볼륨 및 비밀(Secrets) 설정

1.  **데이터 저장을 위한 볼륨 생성**
    `--region`에는 `fly.toml`에 설정한 `primary_region`과 동일한 지역 코드를 입력합니다.

    ```bash
    # 1GB 볼륨 생성 (무료 등급은 총 3GB까지 지원)
    flyctl volumes create vaultwarden_data --app <YOUR_UNIQUE_APP_NAME> --region nrt --size 1
    ```

2.  **관리자 토큰(ADMIN\_TOKEN) 생성 및 설정**
    보안을 위해 관리자 페이지 접근에는 암호화된 토큰이 필요합니다.

      * **(1) 사용할 비밀번호 정하기**: 관리자 페이지에서 사용할 비밀번호를 정합니다. (예: `my-super-secret-123`)

      * **(2) Docker로 해시 값 생성**: 아래 명령어를 실행하고, 위에서 정한 비밀번호를 입력하여 해시 값을 발급받습니다.

        ```bash
        docker run --rm -it vaultwarden/server /vaultwarden hash
        ```

      * **(3) Secrets 설정**: 발급받은 `$`로 시작하는 긴 해시 값을 `ADMIN_TOKEN`으로 설정합니다.

        ```bash
        # '...' 부분에 발급받은 해시 값을 붙여넣으세요.
        flyctl secrets set ADMIN_TOKEN='...' --app <YOUR_UNIQUE_APP_NAME>
        ```

## 🚀 5. 배포 및 확인

1.  **배포**
    모든 설정이 완료되었습니다. 아래 명령어로 배포를 시작합니다.

    ```bash
    flyctl deploy --app <YOUR_UNIQUE_APP_NAME>
    ```

2.  **배포 확인**
    배포가 완료되면 `flyctl status` 명령어로 앱 상태를 확인할 수 있습니다.

    ```bash
    flyctl status --app <YOUR_UNIQUE_APP_NAME>
    ```

3.  **서비스 접속**

      * **Vaultwarden**: `https://<YOUR_UNIQUE_APP_NAME>.fly.dev`
      * **관리자 페이지**: `https://<YOUR_UNIQUE_APP_NAME>.fly.dev/admin`

    > **중요**: 관리자 페이지(`/admin`)에 로그인할 때는 해시 값이 아닌, **4-2 단계에서 직접 정했던 원래의 비밀번호**(예: `my-super-secret-123`)를 사용해야 합니다.

-----

## ✉️ 이메일(SMTP) 설정 (선택 사항)

사용자 초대, 비밀번호 재설정 등을 위해 SMTP 설정이 필요합니다. `fly.toml`에 직접 변수를 추가하는 것보다 `secrets`로 설정하는 것이 훨씬 안전합니다.

아래는 **Google SMTP** 예시이며, [Google 계정 2단계 인증 및 앱 비밀번호 설정](https://support.google.com/accounts/answer/185833) 후 발급받은 **16자리 앱 비밀번호**를 사용해야 합니다.

```bash
flyctl secrets set \
  --app <YOUR_UNIQUE_APP_NAME> \
  SMTP_HOST="smtp.gmail.com" \
  SMTP_FROM="<YOUR_GMAIL_ADDRESS>" \
  SMTP_PORT="587" \
  SMTP_SECURITY="starttls" \
  SMTP_USERNAME="<YOUR_GMAIL_ADDRESS>" \
  SMTP_PASSWORD="<YOUR_16_DIGIT_APP_PASSWORD>"
```

설정 후에는 `flyctl deploy`를 다시 실행하여 변경사항을 적용해야 합니다.

## 🩺 문제 해결

배포나 실행 중 문제가 발생하면 아래 명령어로 로그를 확인할 수 있습니다.

```bash
flyctl logs --app <YOUR_UNIQUE_APP_NAME>
```
