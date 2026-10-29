# Datadog Secret Backend for KeyControl Vault

Keep database and API credentials out of Datadog configuration files. With this tool the Datadog Agent fetches them from your KeyControl vault when it starts.

## How it works

```text
Datadog Agent  --(asks for "dbpassword")-->  datadog-secret-backend.exe  --(login + checkout)-->  KeyControl vault
Datadog Agent  <--(gets the real value)----  datadog-secret-backend.exe  <--(secret value)---------  KeyControl vault
```

1. In a Datadog config file you write `ENC[dbpassword]` instead of the real password.
2. When the Agent starts, it runs `datadog-secret-backend.exe` and asks for `dbpassword`.
3. The executable logs in to the vault with a vault user, checks out the secret, and hands the value back.
4. The Agent uses the value in memory. It is never written to disk.

## Before you start

Run every command in this guide in **PowerShell as Administrator** on the machine where the Datadog Agent is installed.

| You need | How to check |
|---|---|
| Datadog Agent installed and running | `Get-Service datadogagent` shows `Running` |
| The Agent's user account | `Get-LocalUser ddagentuser` shows `Enabled : True` |
| Access to the vault | `Test-NetConnection <vault-host> -Port 443` shows `TcpTestSucceeded : True` |
| The three files below | `datadog-secret-backend.exe`, `datadog-secret-backend.yaml`, and the vault CA certificate (a `.pem` file) |
| A vault user | A user that can check out secrets from your box (a dedicated user with no other rights is best) |
| Secrets in a vault box | Type **Text**, **Password** or **File**, in a box that does **not** use Secondary Approval |
| Go (only to build the `.exe` yourself) | `go version` shows 1.26 or newer. See Step 0. |

If the check for `ddagentuser` fails on a domain controller, the Agent runs as a domain account instead. Use that account name wherever this guide says `ddagentuser`.

## Step 0. Get the executable (skip this if you were given the `.exe`)

If you only have the source code, build the `.exe` yourself. You can build on any Windows PC; the result is one file that you copy to the Datadog host. Building needs Go 1.26 or newer.

The project file `go.mod` also pins `toolchain go1.26.8`. A machine with an older Go (1.21 or newer) and internet access downloads that toolchain automatically. A machine without internet access needs Go 1.26 installed.

### 0.1 Install Go

First check whether Go is already installed:

```powershell
go version
```

If it prints `go1.26` or newer, go to 0.2. If PowerShell says `go` is not recognized, install Go with one of the options below.

**Option A: winget (easiest)**

```powershell
winget install --id GoLang.Go -e
```

Close PowerShell, open a new window and run `go version` again.

If winget fails, use Option B. Typical failures: `0x80072efd` (network error while downloading) and `exit code: 1602` (the installer was cancelled or blocked).

**Option B: zip file (no installer, no admin rights needed)**

```powershell
# download the Go zip
Invoke-WebRequest https://go.dev/dl/go1.27.0.windows-amd64.zip -OutFile "$env:TEMP\go.zip"
# unpack it to a folder in your profile
Expand-Archive "$env:TEMP\go.zip" -DestinationPath "$env:USERPROFILE\go-sdk" -Force
# make `go` available in this PowerShell window
$env:Path = "$env:USERPROFILE\go-sdk\go\bin;" + $env:Path
go version
```

The `$env:Path` line only lasts for the current window. If you open a new window, run it again, or make it permanent:

```powershell
[Environment]::SetEnvironmentVariable("Path", "$env:USERPROFILE\go-sdk\go\bin;" + [Environment]::GetEnvironmentVariable("Path", "User"), "User")
```

You should see something like `go version go1.27.0 windows/amd64`.

### 0.2 Build the executable

```powershell
cd <path-to-repo>\third-party-plugin\datadog-vault-plugin-v1.0
New-Item -ItemType Directory -Path C:\datadog_go -Force
go build -o C:\datadog_go\datadog-secret-backend.exe .
```

| Command | What it does |
|---|---|
| `cd ...` | Goes to the folder that contains `main.go`. |
| `New-Item ...` | Creates the target folder if it does not exist. |
| `go build -o <file> .` | Compiles the code in this folder into the `.exe`. The first build downloads a dependency, so it needs internet access. No output means success. |

Check that the file exists:

```powershell
Get-Item C:\datadog_go\datadog-secret-backend.exe
```

## Step 1. Put the files in one folder

The CA certificate must be in the same folder as the executable. The executable also looks for `datadog-secret-backend.yaml` there by default. You can keep the YAML file somewhere else and point to it with `--config <path>`, but keeping all three files together is simpler.

```powershell
New-Item -ItemType Directory -Path C:\datadog_go -Force
# copy the 3 files into C:\datadog_go, then check:
Get-ChildItem C:\datadog_go
```

You should see the `.exe`, the `.yaml` and the `.pem` file listed.

## Step 2. Lock down who can use the executable

The Agent refuses to run the executable if other users can access it. Only `ddagentuser` (plus `Administrators` and `SYSTEM`) may have rights.

**Option A: with the mouse**

1. Right-click `datadog-secret-backend.exe` and choose **Properties**.
2. Open the **Security** tab and click **Advanced**.
3. Click **Disable inheritance** and remove all existing permissions.
4. Click **Add**, select `ddagentuser`, give it **Read & execute**, and save.


**Option B: with commands**

```powershell
$exe = "C:\datadog_go\datadog-secret-backend.exe"
icacls $exe /inheritance:r              # stop copying permissions from the folder
icacls $exe /grant "ddagentuser:(RX)"   # give the Agent user read and execute
icacls $exe                             # show the result
```

You should see only `ddagentuser:(RX)` in the final listing.

**Also protect the YAML file.** It holds the vault password in plain text.

```powershell
$cfg = "C:\datadog_go\datadog-secret-backend.yaml"
icacls $cfg /inheritance:r
icacls $cfg /grant "ddagentuser:(R)" "Administrators:(F)"   # Agent can read, admins have full control
```

The Agent user must also be able to read the CA certificate. If you removed inheritance on the folder, grant it as well:

```powershell
icacls C:\datadog_go\<ca-file>.pem /grant "ddagentuser:(R)"
```

## Step 3. Fill in the YAML file

Open `C:\datadog_go\datadog-secret-backend.yaml` in a text editor and replace each `<...>` with your own value. Change only what is between the quotes. Keep the key names and the indentation.

**`vault_uid` is required.** If the YAML file you were given has no `vault_uid` line, add it. Without it the program stops with a Go type-conversion panic (`interface conversion`), not a readable error message.

```yaml
backends:
  KeyControlVault:
    backend_type: 'KeyControl.vault'
    vault_url: 'https://<vault-host>'
    vault_uid: '<vault-uuid>'
    ca_cert: '<ca-certificate-file-name>'
    boxName: '<box-name>'
    vault_username: '<vault-user>'
    vault_password: '<vault-user-password>'
```

![config-file](./img/configfile.png)

| Key | What to enter | Example |
|---|---|---|
| `backend_type` | Leave as is. | `KeyControl.vault` |
| `vault_url` | Address of the vault, no path. The program adds `/vault/1.0/...` itself. | `https://vault.example.com` |
| `vault_uid` | ID of the vault. It is the last part of the vault login URL `.../vault/1.0/Login/<vault_uid>/`. | `0a1b2c3d-...` |
| `ca_cert` | File name of the vault CA certificate, stored in the same folder as the `.exe`. | `vault_ca.pem` |
| `boxName` | Name of the box that holds the secrets. | `box1` |
| `vault_username` | Vault user that can check out secrets. | `datadog-reader` |
| `vault_password` | Password of that user. | |

If the password contains characters such as `#`, `:` or `!`, keep it inside the quotes.

## Step 4. Test the executable on its own

Test it before involving Datadog. The Agent talks to the executable with a small JSON message, so we send the same message by hand.

```powershell
cd C:\datadog_go
'{"version":"1.0","secrets":["<secret-name>"]}' | .\datadog-secret-backend.exe
```

If the YAML file is in another folder, add `--config <path-to-yaml>`.

| Command part | Meaning |
|---|---|
| `'{...}'` | The request: asks for the secrets named in the list. Replace `<secret-name>` with a real secret from the box. |
| `\|` | Sends that text to the executable. |
| `.\datadog-secret-backend.exe` | Runs the executable from the current folder. |

You should see the secret's value and no error:

```json
{"<secret-name>":{"Value":"<secret-value>","Error":null}}
```

| Output | Meaning |
|---|---|
| `"Error":null` and a `Value` | Works. `$LASTEXITCODE` is `0`. |
| `... status_code=404, res={"error": "Secret not found"}` and exit code `1` | Wrong secret name or wrong `boxName`. |
| `... Secondary Approval has been enabled ...` and exit code `1` | The box requires approval. Use a box without it. |
| `"Error":"Secret Not Found"` | Same causes as above (older builds report errors this way). |
| `<name>__<key>` instead of `<name>` in the output | The secret is a key/value secret. Use a Text or Password secret. |
| `panic: interface conversion` | A key is missing from the YAML, most often `vault_uid`. |
| A TLS or certificate message | Wrong `ca_cert` or `vault_url`. |
| A login error | Wrong `vault_username` or `vault_password`. |

This test runs as **you**, not as the Agent, so it does not prove Step 2 is right. Step 6 does.

## Step 5. Tell Datadog to use it

Make a copy of the Datadog config first, so you can go back:

```powershell
Copy-Item C:\ProgramData\Datadog\datadog.yaml C:\ProgramData\Datadog\datadog.yaml.bak
```

Open `C:\ProgramData\Datadog\datadog.yaml` and add:

```yaml
secret_backend_command: C:\datadog_go\datadog-secret-backend.exe
```

Then, in the configuration of the check that needs the credentials, replace the plain values with secret names:

```yaml
user: "ENC[<secret-name-for-user>]"
password: "ENC[<secret-name-for-password>]"
```

The name inside `ENC[...]` must match the secret name in the vault box exactly, including upper and lower case.

## Step 6. Restart and verify

```powershell
Restart-Service datadogagent
& "C:\Program Files\Datadog\Datadog Agent\bin\agent.exe" secret
```

| Command | What it does |
|---|---|
| `Restart-Service datadogagent` | The Agent only reads its configuration at start, so restart it. |
| `agent.exe secret` | Asks the Agent to report on the executable and the secrets it found. Secret values are not printed. |

A healthy result looks like this:

```text
=== Checking executable rights ===
Executable path: C:\datadog_go\datadog-secret-backend.exe
Check Rights: OK, the executable has the correct rights

=== Secrets stats ===
Number of secrets decrypted: 2
Secrets handle decrypted:
- <secret-name-for-user>: from <check>.yaml
- <secret-name-for-password>: from <check>.yaml
```

To confirm the check works, run `agent.exe status` and look for your check marked `OK`. If something is wrong, read the end of `C:\ProgramData\Datadog\logs\agent.log`:

```powershell
Get-Content C:\ProgramData\Datadog\logs\agent.log -Tail 50
```

## Something went wrong?

| What you see | What it means | What to do |
|---|---|---|
| `other users/groups than LOCAL_SYSTEM, Administrators or ddagentuser have rights on it` | The executable can be used by more accounts than allowed | Repeat Step 2 |
| `could not query ACLs` | `ddagentuser` cannot read the executable | Give `ddagentuser` **Read & execute** in Step 2 |
| `not a valid Win32 application` | `secret_backend_command` points to a script | Point it to the `.exe` |
| `panic: interface conversion` | A required key, usually `vault_uid`, is missing from the YAML | Add the missing key (Step 3) |
| Certificate or TLS error | Wrong CA file or vault address | Check `ca_cert` and `vault_url`; the `.pem` must be in the same folder as the `.exe` |
| Login failure | Wrong user or password | Try the same login in the vault web page |
| `Secret not found` or `Secret Not Found` | Wrong name or box, or Secondary Approval | Check the secret name and box in the vault |
| Datadog gets no secrets at all | One secret request failed (any response other than 200), so the program exits before returning any secret | Run the Step 4 test for each secret name and fix the one that fails |
| `No mapping between account names and security IDs` (from `icacls`) | The account name is wrong or does not exist on this machine | Run `Get-LocalUser ddagentuser`; on a domain controller use the domain account |
| Works in Step 4 but not in the Agent | The Agent runs as `ddagentuser`, not as you | Check file permissions in Step 2 |

## Good to know

- **Changing a secret in the vault:** the Agent keeps the values until it restarts. Restart it to pick up a change.
- **One failure stops everything:** if any secret request fails, the program exits and Datadog receives no secrets, not just the missing one.
- **Existing secret settings:** if `secret_backend_command` is set, Datadog ignores `secret_backend_type` and `multi_secret_backends`.
- **Where `ENC[...]` works:** in check and integration settings, not in `secret_*` settings.
- **Key/value secrets:** they come back as `<name>__<key>`, which does not match the `ENC[<name>]` name the Agent asks for. Use Text or Password secrets.
- **Special characters:** a `%` in a secret value can corrupt the output, because the secret is printed with `fmt.Printf`.
- **Keep the YAML private:** it contains the vault password. Do not share it or commit it.
- **Example files:** replace any real address, UUID, user name and password with placeholders before you share a YAML file.
- **Least privilege:** give the vault user only the right to check out secrets from this box.

## All commands in order

A short copy-paste version of this guide. Replace the `<...>` parts. Run it in PowerShell as Administrator on the Datadog host.

```powershell
# 0. Build the exe (skip if you already have it)
go version
winget install --id GoLang.Go -e        # or use the zip option in Step 0.1
cd <path-to-repo>\third-party-plugin\datadog-vault-plugin-v1.0
New-Item -ItemType Directory -Path C:\datadog_go -Force
go build -o C:\datadog_go\datadog-secret-backend.exe .

# 1. Copy datadog-secret-backend.yaml and the .pem file into C:\datadog_go, then check
Get-ChildItem C:\datadog_go

# 2. Permissions
icacls C:\datadog_go\datadog-secret-backend.exe /inheritance:r
icacls C:\datadog_go\datadog-secret-backend.exe /grant "ddagentuser:(RX)"
icacls C:\datadog_go\datadog-secret-backend.yaml /inheritance:r
icacls C:\datadog_go\datadog-secret-backend.yaml /grant "ddagentuser:(R)" "Administrators:(F)"
icacls C:\datadog_go\<ca-file>.pem /grant "ddagentuser:(R)"

# 3. Edit C:\datadog_go\datadog-secret-backend.yaml (see Step 3, vault_uid is required)

# 4. Test the exe on its own
cd C:\datadog_go
'{"version":"1.0","secrets":["<secret-name>"]}' | .\datadog-secret-backend.exe

# 5. Back up and edit datadog.yaml (see Step 5)
Copy-Item C:\ProgramData\Datadog\datadog.yaml C:\ProgramData\Datadog\datadog.yaml.bak

# 6. Restart and verify
Restart-Service datadogagent
& "C:\Program Files\Datadog\Datadog Agent\bin\agent.exe" secret
```

## More information

- [Datadog secrets management](https://docs.datadoghq.com/agent/configuration/secrets-management/)
