# SmartContract deploy-and-sync 및 Key Vault 자동 동기화 검증 작업기록

- 작성일: 2026-05-28
- 작성자: Copilot 작업 로그 기반 자동 정리
- 대상 파이프라인: Azure DevOps iwon-vm-cd (pipeline id: 2)
- 핵심 목표:
  1. smart-smartcontract zip 아티팩트 선행 publish 경로 확인
  2. operation=deploy-and-sync 재실행
  3. sync_keyvault.js 성공 로그 및 Key Vault updated 시각 최종 검증

---

## 0) 작업 전 상태 요약

- GitHub Actions deploy-smartcontract 워크플로우와 ADO 연동 경로는 이미 구성된 상태.
- 이전 실행(run 145)에서 Sync Key Vault 단계 실패 이력 존재.
- 실패 원인 후보: Key Vault RBAC 권한 부족.

---

## 1) run 145 최종 상태 확인

실행 명령:

```powershell
az pipelines runs show --organization https://dev.azure.com/iteyes-ito --project iwon-smart-ops --id 145 -o json | ConvertFrom-Json | Select-Object id,status,result,startTime,finishTime | ConvertTo-Json -Depth 4
```

핵심 결과:

- status: inProgress (초기 확인 시점)
- 이후 timeline 상세 조회에서 Sync 단계 실패 확인

---

## 2) run 145 단계별(timeline) 결과 확인

실행 명령:

```powershell
$b = az devops invoke --organization https://dev.azure.com/iteyes-ito --area build --resource builds --route-parameters project=iwon-smart-ops buildId=145 --api-version 7.1 -o json | ConvertFrom-Json
$plan=$b.orchestrationPlan.planId
az devops invoke --organization https://dev.azure.com/iteyes-ito --area build --resource timeline --route-parameters project=iwon-smart-ops buildId=145 timelineId=$plan --api-version 7.1 -o json | ConvertFrom-Json | Select-Object -ExpandProperty records | Where-Object { $_.name -match 'Run Ansible deploy playbook|Ensure Azure CLI for KV Sync|Sync Key Vault|Deploy target service|Precheck Maven package availability' } | Select-Object name,state,result,startTime,finishTime,log | ConvertTo-Json -Depth 6
```

핵심 결과:

- Precheck Maven package availability: succeeded
- Run Ansible deploy playbook: succeeded
- Ensure Azure CLI for KV Sync: succeeded
- Sync Key Vault (sync_keyvault.js) via ADO Service Connection: failed (log id: 22)

---

## 3) run 145 Sync Key Vault 실패 로그 원인 분석

실행 명령:

```powershell
az devops invoke --organization https://dev.azure.com/iteyes-ito --area build --resource logs --route-parameters project=iwon-smart-ops buildId=145 logId=22 --api-version 7.1 --accept-media-type "text/plain" --out-file run145_log22_v2.txt
```

확인 포인트(로그 핵심):

- Running sync_keyvault.js...
- Error: az keyvault secret set 실패 [IWON-COMPANY-PRIVATE-KEY]
- Code: Forbidden
- Inner error: ForbiddenByRbac
- Action: Microsoft.KeyVault/vaults/secrets/setSecret/action
- Assignment: (not found)

판단:

- ADO 서비스 커넥션 서비스 프린시펄에 Key Vault secret set 권한이 없어 동기화 실패.

---

## 4) run 145 시점 Key Vault updated baseline 확인

실행 명령:

```powershell
$names=@('IWON-COMPANY-PRIVATE-KEY','IWON-ADMIN-PRIVATE-KEY','IWON-ADMIN-ADDRESS','IWON-OPERATOR-ADDRESS','IWON-TOKEN-ADDRESS')
foreach($n in $names){
  az keyvault secret show --vault-name iwonsvckvkrc001 --name $n --query "{name:name,updated:attributes.updated,enabled:attributes.enabled}" -o json
}
```

핵심 결과:

- IWON-COMPANY-PRIVATE-KEY: 2026-05-07T06:34:02+00:00
- IWON-ADMIN-PRIVATE-KEY: 2026-05-07T06:34:00+00:00
- IWON-TOKEN-ADDRESS: 2026-05-07T06:33:57+00:00
- IWON-ADMIN-ADDRESS: SecretNotFound
- IWON-OPERATOR-ADDRESS: SecretNotFound

---

## 5) 권한 문제 해결을 위한 대상 주체/모드 확인

실행 명령:

```powershell
az devops service-endpoint list --organization https://dev.azure.com/iteyes-ito --project iwon-smart-ops -o json
az keyvault show --name iwonsvckvkrc001 --query "{id:id,resourceGroup:resourceGroup,enableRbacAuthorization:properties.enableRbacAuthorization}" -o json
```

핵심 결과:

- Azure RM 서비스 커넥션: iwon-smart-ops-sc
- serviceprincipalid: fef01f57-4763-428d-b19d-9f31b0490213
- 실제 호출 주체 oid(로그 기준): 86cbf4bc-58a1-4e0a-8401-aa515e08687a
- Key Vault enableRbacAuthorization: true

판단:

- Access Policy 방식이 아닌 RBAC 역할 할당 필요.

---

## 6) Key Vault RBAC 역할 할당

실행 명령:

```powershell
$scope='/subscriptions/51be5183-cf60-4f1f-8b9f-fb4b31daa579/resourceGroups/iwon-svc-rg/providers/Microsoft.KeyVault/vaults/iwonsvckvkrc001'
az role assignment list --scope $scope --query "[].{principalId:principalId,role:roleDefinitionName,principalType:principalType}" -o table
az role assignment create --assignee fef01f57-4763-428d-b19d-9f31b0490213 --role "Key Vault Secrets Officer" --scope $scope -o json
az role assignment list --scope $scope --assignee fef01f57-4763-428d-b19d-9f31b0490213 --query "[].{role:roleDefinitionName,principalId:principalId}" -o table
```

핵심 결과:

- principalId 86cbf4bc-58a1-4e0a-8401-aa515e08687a 에
  - 기존: Key Vault Secrets User
  - 추가: Key Vault Secrets Officer

조치 완료:

- secret set 권한 확보.

---

## 7) deploy-and-sync 재실행 (run 147)

실행 명령:

```powershell
az pipelines run --organization https://dev.azure.com/iteyes-ito --project iwon-smart-ops --id 2 --branch main --parameters deployTarget=smartcontract operation=deploy-and-sync mavenPackageVersion=latest -o json | ConvertFrom-Json | Select-Object id,status,result,queueTime,url | ConvertTo-Json -Depth 4
```

핵심 결과:

- run id: 147 생성

---

## 8) run 147 진행/결과 모니터링

실행 명령:

```powershell
az pipelines runs show --organization https://dev.azure.com/iteyes-ito --project iwon-smart-ops --id 147 -o json | ConvertFrom-Json | Select-Object id,status,result,startTime,finishTime | ConvertTo-Json -Depth 4

$b = az devops invoke --organization https://dev.azure.com/iteyes-ito --area build --resource builds --route-parameters project=iwon-smart-ops buildId=147 --api-version 7.1 -o json | ConvertFrom-Json
$plan=$b.orchestrationPlan.planId
az devops invoke --organization https://dev.azure.com/iteyes-ito --area build --resource timeline --route-parameters project=iwon-smart-ops buildId=147 timelineId=$plan --api-version 7.1 -o json | ConvertFrom-Json | Select-Object -ExpandProperty records | Where-Object { $_.name -match 'Run Ansible deploy playbook|Ensure Azure CLI for KV Sync|Sync Key Vault|Precheck Maven package availability|Deploy target service' } | Select-Object name,state,result,startTime,finishTime,log | ConvertTo-Json -Depth 6
```

최종 결과:

- Run 147 status: completed
- Run 147 result: succeeded
- Sync Key Vault 단계: succeeded

---

## 9) run 147 sync_keyvault.js 성공 로그 증빙

실행 명령:

```powershell
az devops invoke --organization https://dev.azure.com/iteyes-ito --area build --resource logs --route-parameters project=iwon-smart-ops buildId=147 logId=22 --api-version 7.1 --accept-media-type "text/plain" --out-file run147_log22.txt
Select-String -Path run147_log22.txt -Pattern 'Running sync_keyvault\.js|Key Vault 동기화 완료|\[KV\] set|##\[error\]|Forbidden|failed' | ForEach-Object { $_.Line }
```

로그 핵심 라인:

- Running sync_keyvault.js...
- [KV] set IWON-COMPANY-PRIVATE-KEY
- [KV] set IWON-ADMIN-PRIVATE-KEY
- [KV] set IWON-ADMIN-ADDRESS
- [KV] set IWON-OPERATOR-ADDRESS
- [KV] set IWON-TOKEN-ADDRESS
- Key Vault 동기화 완료.

판정:

- sync_keyvault.js 단계 정상 성공.

---

## 10) run 147 이후 Key Vault updated 시각 최종 검증

실행 명령:

```powershell
$names=@('IWON-COMPANY-PRIVATE-KEY','IWON-ADMIN-PRIVATE-KEY','IWON-ADMIN-ADDRESS','IWON-OPERATOR-ADDRESS','IWON-TOKEN-ADDRESS')
foreach($n in $names){
  az keyvault secret show --vault-name iwonsvckvkrc001 --name $n --query "{name:name,updated:attributes.updated,enabled:attributes.enabled}" -o json
}
```

최종 updated 결과:

- IWON-COMPANY-PRIVATE-KEY: 2026-05-28T00:25:27+00:00
- IWON-ADMIN-PRIVATE-KEY: 2026-05-28T00:25:28+00:00
- IWON-ADMIN-ADDRESS: 2026-05-28T00:25:29+00:00
- IWON-OPERATOR-ADDRESS: 2026-05-28T00:25:29+00:00
- IWON-TOKEN-ADDRESS: 2026-05-28T00:25:30+00:00

비교 결론:

- run 145 baseline 대비 run 147 이후 updated 시각이 모두 최신으로 갱신됨.
- 기존 SecretNotFound였던 ADMIN/OPERATOR ADDRESS도 정상 생성/반영됨.

---

## 최종 결론

- 요구사항 3가지 모두 충족:
  1. smart-smartcontract 아티팩트 publish 경로 확인
  2. operation=deploy-and-sync 재실행 완료
  3. sync_keyvault.js 성공 로그 및 Key Vault updated 시각 최종 검증 완료

- 핵심 해결 포인트:
  - 실패 원인은 Key Vault RBAC 권한 부족(ForbiddenByRbac)
  - 서비스 커넥션 주체에 Key Vault Secrets Officer 역할 부여 후 동일 시나리오 성공
