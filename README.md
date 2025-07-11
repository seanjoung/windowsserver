# windowsserver



script 1. delete file with CSV
####################################################################
delete file with CSV

# CSV 파일 경로 설정
$csvFilePath = "C:\Scripts\Edrive-18.csv" # CSV 파일 경로를 실제 경로로 변경

# CSV 파일에서 "FullName" 열의 값을 가져옵니다.
$fullNames = Import-Csv -Path $csvFilePath | Select-Object -ExpandProperty 파일

# 삭제 대상 리스트 출력
Write-Host "삭제 대상 리스트:"
foreach ($fullName in $fullNames) {
    Write-Host $fullName
}

# 삭제 여부 확인
$confirm = Read-Host -Prompt "삭제하시겠습니까? (y/n)"

# 삭제 여부에 따라 처리
if ($confirm -eq "y") {
    foreach ($fullName in $fullNames) {
        if (Test-Path -Path $fullName) {
            Remove-Item -Path $fullName -Force # -WhatIf 옵션 제거, 실제 삭제 수행
            Write-Host "Deleted: $($fullName)"
        } else {
            Write-Warning "File or folder not found: $($fullName)"
        }
    }
    Write-Host "Script completed."
} elseif ($confirm -eq "n") {
    Write-Host "삭제가 취소되었습니다."
} else {
    Write-Warning "잘못된 입력입니다. y 또는 n을 입력하세요."
}

######################################################################

script 2.check old file dir, files
#######################################################################

$ErrorActionPreference = "Continue"

# E:\ 드라이브의 모든 하위 폴더를 가져옵니다.
Get-ChildItem -Path "E:\" -Directory | ForEach-Object {
    $currentFolder = $_.FullName
    $results = @()

    Get-ChildItem -Path $currentFolder -Recurse -File | ForEach-Object {
        try {
            # Mac 임시폴더, DS_Store만 예외
            if ($_.FullName.Contains("__MACOSX") -or $_.FullName.EndsWith(".DS_Store")) {
                Write-Warning "Skipping invalid path: $($_.FullName)"
                return
            }

            # 2015년 이전에 *생성된* 파일인지 확인합니다.
            if ($_.CreationTime -lt (Get-Date "2015-01-01")) { # CreationTime 사용
                $acl = Get-Acl $_.FullName -ErrorAction Stop

                $fileInfo = [PSCustomObject]@{
                    파일 = $_.FullName
                    용량 = $_.Length
                    소유자 = $acl.Owner
                    생성_시간 = $_.CreationTime
                }
                $results += $fileInfo
            }
        } catch {
            Write-Warning "Error processing file: $($_.FullName) - $($_.Exception.Message)"
        }
    }

    if ($results.Count -eq 0) {
        Write-Warning "No results found in $currentFolder. Check your filter or permissions."
    } else {
        # 현재 폴더 이름을 기반으로 CSV 파일 경로 생성
        $csvPath = "C:\Scripts\$($_.Name)_file_info.csv"

        # CSV 파일 저장 (각 폴더별로)
        $results | Export-Csv -Path $csvPath -NoTypeInformation -Encoding UTF8 -Append
    }
}

Write-Host "Finished processing all folders."

##############################################################################
