# 백업 정책

이 문서는 `family-archive-appliance`의 백업 정책을 정리합니다.

---

## 1. 현재 상태

현재는 단일 HDD로 운영합니다. **백업이 없습니다.**

단일 HDD는 백업이 아닙니다. HDD 장애 시 데이터 복구가 불가능합니다. 백업 HDD가 추가되기 전까지 이 서버를 유일한 원본 저장소로 보지 않습니다. 사진 원본은 휴대폰, PC, 클라우드 등 다른 위치에도 유지합니다.

---

## 2. 백업 대상

백업 대상은 originals와 metadata(SQLite)입니다.

thumbnails와 medium은 원본으로부터 재생성 가능한 캐시입니다. 백업하지 않습니다.

metadata(SQLite)는 originals와 함께 백업합니다. HDD 장애 시 originals만 복구하면 원본 파일명, 업로드 시각, 삭제 상태 등 메타데이터가 손실됩니다.

---

## 3. 백업 방식

originals는 rsync를 사용합니다. `--delete` 옵션은 사용하지 않습니다.

`--delete` 없이 실행하면 신규 파일만 백업 HDD로 복사되며, 원본에서 삭제된 파일은 백업에 그대로 남습니다. originals는 변형되지 않는 파일이므로 rsync는 사실상 신규 파일만 추가 복사합니다.

metadata(SQLite)는 실행 중인 DB 파일을 직접 rsync하지 않습니다. `sqlite3 .backup` 명령으로 일관된 snapshot 파일을 먼저 생성한 뒤 rsync합니다. 실행 중 DB 파일을 직접 복사하면 WAL 모드 여부에 따라 정합성이 없는 파일이 백업될 수 있습니다.

**실행 순서:**

```
1. 백업 HDD 마운트 여부 확인
   → 마운트 안 된 경우 즉시 실패. rsync 미실행.

2. metadata snapshot 생성
   sqlite3 /srv/family-archive/metadata/family-archive.sqlite \
     ".backup /tmp/family-archive-YYYY-MM-DD.sqlite"

3. metadata snapshot rsync
   rsync -av /tmp/family-archive-YYYY-MM-DD.sqlite \
     <backup-mount>/metadata/

4. originals rsync
   rsync -av /srv/family-archive/originals/ <backup-mount>/originals/
```

metadata snapshot을 먼저 생성하는 이유: snapshot 생성 이후 originals rsync 중에 새 업로드가 발생하면 백업 originals에는 있지만 백업 metadata에는 없는 파일이 생깁니다. 이 상태는 UUID 기반 파일 스캔으로 reconcile 가능합니다. 반대 순서(originals 먼저)로 실행하면 백업 metadata에는 있지만 백업 originals에는 없는 파일이 생길 수 있으며, 이는 복구 불가능한 상태입니다.

metadata snapshot은 날짜별 파일명으로 누적 저장합니다.

```
<backup-mount>/metadata/
├── family-archive-2025-01-01.sqlite
├── family-archive-2025-01-02.sqlite
└── ...
```

originals가 `--delete` 없이 전체 누적 보존되는 것과 동일한 방향입니다. 수동 정리는 1~2년 단위로 수행합니다.

**마운트 검증:** backup-runner는 백업 실행 전 백업 HDD가 실제 마운트되어 있는지 확인합니다. 마운트 marker 파일 또는 mountpoint 여부를 확인하고, 마운트가 확인되지 않으면 rsync를 실행하지 않고 실패 로그를 남깁니다.

---

## 4. 백업 정합성 한계

백업은 완전한 트랜잭션 스냅샷을 보장하지 않습니다.

metadata snapshot과 originals rsync는 순차 실행됩니다. 그 사이에 업로드가 발생하면 백업 originals에는 있지만 백업 metadata에는 없는 파일이 생길 수 있습니다.

이 선택의 이유: 백업 중 API 쓰기 잠금을 구현하면 backup-runner와 API 사이에 의존이 생기며 운영 복잡도가 높아집니다. 단일 운영자 홈서버에서 잠금 해제 실패는 서비스 중단으로 이어집니다. 복구 가능한 불일치 상태(파일 있음 + metadata 없음)를 허용하는 것이 복구 불가능한 실패 케이스(metadata 있음 + 파일 없음)를 막기 위한 잠금보다 낫습니다.

복구 시 불일치가 발생하면 originals 디렉토리의 UUID 목록과 metadata 레코드를 대조하여 reconcile합니다. 파일명이 UUID이고 UUID가 DB primary key와 동일하므로 대조가 가능합니다.

---

## 5. 백업 주기

매일 03:00에 Cron으로 실행합니다.

Trash cleanup(매일 04:00)과 동시에 실행되지 않도록 1시간 간격을 둡니다.

---

## 6. 백업 HDD 마운트 경로

미확정입니다. `/srv/` 아래는 사용하지 않습니다.

---

## 7. 미적용 항목

다음은 현재 적용하지 않습니다.

- 오프사이트 백업
- 클라우드 백업
- 백업 검증 자동화
- 백업 HDD 공간 자동 관리
