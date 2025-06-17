---
runme:
  id: 01JXXSNE3R5X2P6XRY9X0RYX5G
  version: v3
---

# Bitwarden Toolbox

#### 1. Setup BW Session

Loads session data and checks login/lock status. If necessary, prompts for login/password. Upon successful unlock, it saves the session in a temporary Session.sh file.

```sh {"closeTerminalOnSuccess":"false","id":"01JXXQXE264H61JE4646BZ1VZY"}
. Session.sh
status=$(bw status | jq -r ".status")
if [ "$status" == "unauthenticated" ]; then bw login; fi
if [ "$status" == "locked" ]; then echo -n
    bw unlock | grep "export BW_SESSION" | sed 's/$ //' > ./Session.sh
    . Session.sh
fi
echo "Bitwarden is $(bw status | jq -r '.status')"
```

#### 2. Feed search results to Work.json

Loads items into temporary work file. Edit the search query below according to your needs.

```sh {"id":"01JXXQXE264H61JE464BGT6HCJ"}
. Session.sh
bw list items --search www. | jq > Work.json
bw list collections | jq > Collections.json
bw list folders | jq > Folders.json
bw list organizations | jq > Organizations.json
```

[Open Work.json](./Work.json) and make changes within VS Code. E.g. search/replace or manual JSON edits.

#### 3. Process items in Work.json

Items with `"delete": true` key set will be deleted. Remaining items will be updated.

```sh {"closeTerminalOnSuccess":"false","id":"01JXXVHTAK6S1FS5CR02XX5ZSP"}
. Session.sh
jq -r '.[] | .id' Work.json | while read id; do echo -n
    printf -v query '.[] | select(.id=="%s")' $(echo $id | sed 's/\r//;s/\n//')
    name=$(jq "$query" ./Work.json | jq -r '.name')
    delete=$(jq "$query" ./Work.json | jq -r '.delete')
    if [[ "$delete" == "1" || "$delete" == "true" ]]; then echo -n
        echo "DEL: $name - $id"
        bw delete item $id > /dev/null
    else echo -n
        echo "MOD: $name - $id"
        jq "$query" ./Work.json | jq 'del(.revisionDate)' | bw encode | bw edit item $id > /dev/null
    fi
done
```

#### 4. Cleanup & Lock

Cleans up work files and session data. Finally, locks Bitwarden vault.

```sh
cat /dev/null > Work.json
cat /dev/null > Session.sh
bw lock
```

*Read the docs on [runme.dev](https://runme.dev/docs/intro) to learn how to get most out of Runme notebooks!*
