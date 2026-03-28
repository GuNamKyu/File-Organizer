---
description: 사용자가 제공한 경로의 파일을 확장자 기반으로 분류하여 넘버링된 폴더로 정리하는 워크플로우
---

# 📂 파일 정리 워크플로우 (organize-files)

// turbo-all

사용자가 `[경로 : 위치경로]` 형식으로 경로를 제공하면, 해당 경로의 파일을 스캔하고 분류하여 넘버링된 폴더로 정리한다.
정리 완료 후 `logs/` 폴더에 경로별 실행 기록을 자동 저장한다.

---

## 사전 준비

이 워크플로우를 실행하기 전에 다음 설정 파일을 반드시 로드한다:

1. `config/rules.json` — 분류 규칙 (최대 깊이, 넘버링 형식, 기본 카테고리)
2. `config/extensions.json` — 확장자 → 카테고리 매핑 정보

---

## 실행 단계

### 1단계: 입력 파싱 및 경로 추출

사용자 메시지에서 `[경로 : ...]` 패턴을 찾아 대상 경로를 추출한다.

- 형식: `[경로 : C:\Users\example\path]`
- 경로에서 앞뒤 공백 및 따옴표(`"`, `'`)를 제거한다.
- 경로가 추출되지 않으면 사용자에게 올바른 형식을 안내하고 중단한다.

### 2단계: 경로 유효성 검증

`list_dir` 도구를 사용하여 대상 경로가 존재하고 접근 가능한지 확인한다.

- 경로가 존재하지 않거나 접근 불가능하면 사용자에게 알리고 중단한다.
- 경로가 유효하면 다음 단계로 진행한다.

### 3단계: 전체 파일/폴더 스캔

PowerShell로 대상 경로의 전체 구조를 스캔하여 파일 수, 폴더 수, 확장자 분포를 파악한다.

```powershell
$base = "대상경로"
$folders = (Get-ChildItem -LiteralPath $base -Directory -Recurse).Count
$files   = (Get-ChildItem -LiteralPath $base -File -Recurse).Count
Get-ChildItem -LiteralPath $base -Recurse | Select-Object FullName, Extension | Sort-Object FullName
```

### 4단계: 설정 파일 로드 및 분류 계획 수립

- `config/rules.json`, `config/extensions.json`을 읽는다.
- 파일명·내용·확장자를 기반으로 분류 카테고리를 결정한다.
- 신규 폴더 구조 계획을 수립하고 사용자에게 제시한다.

**폴더 넘버링 규칙:**
- 형식: `{번호}_{카테고리명}` (예: `01_문서`, `02_이미지`)
- 번호는 `01`, `02`, ... 형식 (2자리 0-패딩)
- 기존 넘버링 폴더가 있으면 그 번호 체계를 유지한다.
- **폴더 깊이는 최대 5단계까지만 허용한다.**
- **파일명은 절대 변경하지 않는다.**

### 5단계: 사용자 승인 요청

정리 계획을 사용자에게 보여주고 승인을 요청한다.

```
📊 파일 정리 계획:
══════════════════════════════════════════
  📁 대상 경로: [경로]
  📄 정리 대상 파일: [N]개
══════════════════════════════════════════
  [분류 계획 요약]
══════════════════════════════════════════
  ※ 파일명은 변경되지 않습니다. 폴더만 번호가 부여됩니다.

계속 진행하시겠습니까?
```

- 승인 시 6단계로 진행
- 거부 시 중단하고 수정 사항을 확인한다

### 6단계: 정리 스크립트 생성 및 실행

승인된 계획을 바탕으로 PowerShell 스크립트(`organize_run.ps1`)를 생성하고 **한 번에 실행**한다.

**스크립트 구조:**

```powershell
$base = "대상경로"
$logPath = "c:\Users\namkyu-gu\workspace\File-Organizer\logs\{날짜}_{폴더명}.md"
$moved = 0; $errs = @(); $newFolders = @()

# Phase 1: 폴더 생성
@("01_카테고리1", "02_카테고리2", ...) | ForEach-Object {
    New-Item -ItemType Directory -Path (Join-Path $base $_) -Force | Out-Null
    $newFolders += $_
}

# Phase 2~N: 파일 이동 (각 분류 규칙에 따라)
# ... (분류 로직)

# Phase 마지막: 빈 폴더 정리
Get-ChildItem -LiteralPath $base -Directory -Recurse |
  Sort-Object { $_.FullName.Length } -Descending |
  ForEach-Object {
    if ($_.Name -notmatch "^[0-9]") {
      if ((Get-ChildItem -LiteralPath $_.FullName -Recurse).Count -eq 0) {
        Remove-Item -LiteralPath $_.FullName -Force -Recurse
      }
    }
  }

# 로그 자동 저장
$logContent = @"
# 파일 정리 실행 기록
- 실행일시: $(Get-Date -Format 'yyyy-MM-dd HH:mm')
- 대상 경로: `$base`
- 이동된 파일: `$moved 개
- 오류: `$($errs.Count) 건
[결과 구조]
"@
$logContent | Out-File -FilePath $logPath -Encoding UTF8

Write-Host "✅ 완료! 이동: $moved 개, 오류: $($errs.Count) 건"
Write-Host "📋 로그 저장: $logPath"
```

**실행 원칙:**
- 스크립트를 생성한 후 `run_command`로 **한 번에 전체 실행**한다.
- Phase를 나눠서 별도로 실행하지 않는다.
- 인코딩 문제 방지를 위해 `-File` 대신 인라인 실행을 사용한다.

### 7단계: 결과 보고 및 로그 저장

실행 완료 후 결과를 요약하여 보고한다.

```
✅ 파일 정리 완료!
══════════════════════════════════════════
  📁 대상 경로: [경로]
  📄 이동된 파일: [N]개
  📁 생성된 폴더: [M]개
  ⚠️ 오류: [K]건
  📋 로그: logs/{날짜}_{폴더명}.md
══════════════════════════════════════════
```

로그 파일은 `logs/` 폴더에 `{YYYYMMDD}_{대상폴더명}.md` 형식으로 저장한다.

---

## 엣지 케이스 처리

| 상황 | 처리 방법 |
|------|----------|
| 대상 경로에 파일이 없음 | "정리할 파일이 없습니다" 안내 후 중단 |
| 동일 파일명 충돌 | `-Force`로 덮어쓰기 (동일 이름 파일이 있는 경우 사용자에게 사전 알림) |
| 확장자 없는 파일 | `defaultCategory` (99_기타)로 분류 |
| 숨김 파일 (`.`으로 시작) | 정리 대상에서 제외 |
| 폴더 깊이 5단계 초과 | 5단계까지만 허용, 초과 시 경고 후 현재 위치 유지 |
| 이미 넘버링된 폴더 존재 | 기존 번호 체계를 유지하고 이어서 번호 부여 |
| 읽기 전용 파일 | 이동 실패 시 `errs`에 기록하고 계속 진행 |
| 인코딩 오류 | `-File` 대신 인라인 PowerShell 명령으로 실행 |
