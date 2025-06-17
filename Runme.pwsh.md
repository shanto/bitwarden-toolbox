---
runme:
  id: 01JXXSNE3R5X2P6XRY9X0RYX5H
  version: v3
---

# Bitwarden Toolbox

#### 1. Setup BW Session

Loads session data and checks login/lock status. If necessary, prompts for login/password. Upon successful unlock, it saves the session in a temporary Session.ps1 file.

```bash {"closeTerminalOnSuccess":"false","id":"01JXXQXE264H61JE4646BZ1VZY"}
Get-Content Session.ps1 | Invoke-Expression
bw status | jq -r ".status" | Set-Variable -Name "status"
Set-Variable -Name locked -Value locked
Set-Variable -Name unauthenticated -Value unauthenticated
if($status -eq $unauthenticated) {
    Write-Host Login...
    bw login
}
if($status -eq $locked) {
    Write-Host Unlocking...
    bw unlock | Select-String -Pattern "env:BW_SESSION" | ForEach-Object { $_.Line.SubString(1) } | Set-Content ./Session.ps1
    Get-Content Session.ps1 | Invoke-Expression
}
bw status | jq -r '.status' | Set-Variable -Name status
Write-Host "Bitwarden is $status"
```

#### 2. Feed search results to Work.json

Loads items into temporary work file. Edit the search query below according to your needs.

```sh {"id":"01JXXQXE264H61JE464BGT6HCJ"}
Get-Content Session.ps1 | Invoke-Expression
bw list items --search www. | jq > Work.json
bw list collections | jq > Collections.json
bw list folders | jq > Folders.json
bw list organizations | jq > Organizations.json
```

[Open Work.json](./Work.json) and make changes within VS Code. E.g. search/replace or manual JSON edits.

#### 3. Process items in Work.json

Items with `"delete": true` will be deleted. Remaining items will be updated.

```sh {"closeTerminalOnSuccess":"false","id":"01JXXVHTAK6S1FS5CR02XX5ZSP"}
Get-Content Session.ps1 | Invoke-Expression
ForEach ($id in (jq -r '.[] | .id' Work.json)) {
    Set-Variable -Name query -Value ('.[ ] | select(.id==\"{0}\")' -f $id)
    jq $query ./Work.json | jq -r '.name' | Set-Variable -Name name
    jq $query ./Work.json | jq -r '.delete' | Set-Variable -Name delete
    Set-Variable -Name "truth" -Value "true", 1
    If($truth -contains $delete) {
        Write-Host "DEL: $name - $id"
        bw delete item $id | out-null
    } else {
        Write-Host "MOD: $name - $id"
        jq $query ./Work.json | jq 'del(.revisionDate)' | bw encode | bw edit item $id | Out-Null
    }
}
```

#### 4. Cleanup & Lock

Cleans up work files and session data. Finally, locks Bitwarden vault.

```sh
Set-Content -Value "" -Path ./Work.json
Set-Content -Value "" -Path ./Session.ps1
bw lock
```

*Read the docs on [runme.dev](https://runme.dev/docs/intro) to learn how to get most out of Runme notebooks!*
