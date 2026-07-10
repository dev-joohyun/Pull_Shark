Gitlab Version Up

> Gitlab MCP를 18버전 부터 지원해서, 18.11 업그레이드 진행
>
> 17.7.7 → 17.8.x → 17.11.x → 18.0 → 18.2 → 18.5 → 18.8 → 18.11

### 1. 백업

```
sudo docker exec gitlab gitlab-backup create
```

### 2. 백업 완료 체크

```
sudo docker exec gitlab ls /var/opt/gitlab/backups/
```

이런 식으로 파일이 보이면 완료된 겁니다:

```
1749958800_2026_06_15_17.7.7_gitlab_backup.tar
```

### 3. 17.7.7 → 17.8.x 업그레이드

```
image: gitlab/gitlab-ce:17.8.7-ce.0
```

```
sudo docker-compose pull
sudo docker-compose up --build -d
```

```
sudo docker logs -f gitlab
```

로그에서 `Reconfigured!` 나오면 완료

### 4-1. 17.8.x → 17.11.x 업그레이드

```
image: gitlab/gitlab-ce:17.11.7-ce.0
```

이미지 파일 변경 후 동일한 과정 진행

### 4-2. 18로 넘어가기전 PG 업그레이드

docker-compose up 상태에서 진행해야함

```
sudo docker exec gitlab gitlab-ctl reconfigure
sudo docker exec gitlab gitlab-ctl pg-upgrade -V 16

sudo docker exec gitlab gitlab-psql -c "SELECT version();" # 16버전 출력 확인
```

### 5. 17.11.x → 18.0.x 업그레이드

```
image: gitlab/gitlab-ce:18.0.0-ce.0
```

이미지 파일 변경 후 동일한 과정 진행

### 6. 18.0.x → 18.2.x 업그레이드

```
image: gitlab/gitlab-ce:18.2.0-ce.0
```

### 7. 18.2.x → 18.5.x 업그레이드

```
image: gitlab/gitlab-ce:18.5.7-ce.0
```

### 7. 18.5.x → 18.8.x 업그레이드

```
image: gitlab/gitlab-ce:18.8.0-ce.0
```

### 7. 18.8.x → 18.11.x 업그레이드

```
image: gitlab/gitlab-ce:18.11.5-ce.0
```

## 참고

postgresql 상태 확인

```
sudo docker exec gitlab gitlab-ctl status postgresql
```

postgresql 계속 안뜨면

```
sudo docker exec gitlab gitlab-ctl start postgresql
```

그래도 문제 생기면 postgresql 버전 업그레이드

```
sudo docker exec gitlab gitlab-ctl reconfigure
```






# GitLab 백업 및 롤백 가이드

---

## 사전 준비 (업그레이드 전 필수)

### 1. GitLab 백업 생성

`sudo docker exec gitlab gitlab-backup create`

### 2. 백업 파일명 확인

`sudo docker exec gitlab ls /var/opt/gitlab/backups/`

### 3. 백업 파일 안전한 곳에 복사

`sudo cp /home/lpsdev1/app/gitlab/data/backups/{백업파일명}_gitlab_backup.tar /home/lpsdev1/`

### 4. 설정 파일 별도 백업 (중요)

`sudo cp /home/lpsdev1/app/gitlab/config/gitlab.rb /home/lpsdev1/
sudo cp /home/lpsdev1/app/gitlab/config/gitlab-secrets.json /home/lpsdev1/`

> **주의**: `gitlab.rb`와 `gitlab-secrets.json`은 GitLab 백업에 포함되지 않으므로 반드시 별도로 보관해야 합니다.

| 파일 | 설명 |
|----|----|
| `gitlab.rb` | GitLab 메인 설정 파일 (external_url, SMTP, LDAP 등) |
| `gitlab-secrets.json` | 암호화 키 모음 (CI/CD 변수, 2FA, Token 등) 없으면 암호화된 데이터 복호화 불가 |

---

## 롤백 실행

### 5. 컨테이너 내리기

`sudo docker-compose down`

### 6. 볼륨 초기화

`sudo rm -rf /home/lpsdev1/app/gitlab/data
sudo rm -rf /home/lpsdev1/app/gitlab/logs
sudo rm -rf /home/lpsdev1/app/gitlab/config`

> **주의**: 볼륨 초기화 시 모든 데이터가 삭제됩니다. 반드시 3\~4번 과정을 먼저 완료하세요.

### 7. docker-compose.yml 롤백 버전으로 변경

`image: gitlab/gitlab-ce:17.7.7-ce.0`

### 8. 컨테이너 올리기

`sudo docker-compose up -d`

완전히 뜰 때까지 대기:

`sudo docker logs -f gitlab | grep -E "Reconfigured|ERROR"`

`Reconfigured!` 확인 후 다음 단계 진행

### 9. 설정 파일 복원

`sudo cp /home/lpsdev1/gitlab.rb /home/lpsdev1/app/gitlab/config/
sudo cp /home/lpsdev1/gitlab-secrets.json /home/lpsdev1/app/gitlab/config/`

### 10. 백업 파일 복원 위치로 이동 및 권한 변경

`sudo cp /home/lpsdev1/{백업파일명}_gitlab_backup.tar /home/lpsdev1/app/gitlab/data/backups/

sudo chmod 777 /home/lpsdev1/app/gitlab/data/backups/{백업파일명}_gitlab_backup.tar`

### 11. 백업 복원

`sudo docker exec gitlab gitlab-backup restore BACKUP={백업파일명} GITLAB_ASSUME_YES=1`

`Restore task is done` 확인 후 다음 단계 진행

### 12. 재시작

`sudo docker-compose restart gitlab`

### 13. 비밀번호 재설정

`sudo docker exec -it gitlab gitlab-rake "gitlab:password:reset[root]"`

---

## 주의사항

| 항목 | 내용 |
|----|----|
| 백업 파일 위치 | 볼륨 초기화 전 반드시 외부로 복사 |
| 설정 파일 | `gitlab.rb`, `gitlab-secrets.json` 별도 백업 필수 |
| 파일 권한 | `chmod 777` 안 하면 복원 실패 |
| PG 버전 | 볼륨 초기화하면 자동으로 구버전으로 초기화됨 |
| 복원 순서 | 설정 파일 복원 → 백업 복원 순서 지킬 것 |
