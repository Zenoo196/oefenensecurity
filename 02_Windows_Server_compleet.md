# 02 · Windows Server compleet (vinden, oplossen, testen)

> Proeftoets: server **SRV001**, map **`C:\Fileserver`** met submappen Administratie, Iedereen, Magazijn, Management, Sales.
> PowerShell **als Administrator**: Start → typ `powershell` → rechtermuis → *Run as administrator*.
> CMD als Administrator werkt ook voor de `net`-commando's.

**Inhoud**
1. [Snelkoppelingen (Win+R)](#1-snelkoppelingen-winr)
2. [Gebruikers en groepen vinden](#2-gebruikers-en-groepen-vinden)
3. [Rechten op mappen vinden (Share + NTFS)](#3-rechten-op-mappen-vinden-share--ntfs)
4. [Logs: Event Viewer](#4-logs-event-viewer)
5. [Oplossen: klikpaden](#5-oplossen-klikpaden)
6. [Oplossen: overige afwijkingen uit W1](#6-oplossen-overige-afwijkingen-uit-w1)
7. [Oplossen: kant-en-klaar in PowerShell](#7-oplossen-kant-en-klaar-in-powershell)
8. [Testen](#8-testen)
9. [Controlescript: alles in één keer](#9-controlescript-alles-in-één-keer)
10. [Domeincontroller? (Active Directory)](#10-domeincontroller-active-directory)

---

## 1. Snelkoppelingen (Win+R)

| Typ | Opent | Waarvoor |
|---|---|---|
| `lusrmgr.msc` | Local Users and Groups | gebruikers, groepen, Guest |
| `compmgmt.msc` | Computer Management | gebruikers + **Shared Folders** + Event Viewer in één |
| `fsmgmt.msc` | Shared Folders | alle shares, sessies, open bestanden |
| `eventvwr.msc` | Event Viewer | wie heeft ingelogd |
| `dsa.msc` | AD Users and Computers | alleen bij domeincontroller |
| `gpedit.msc` | Local Group Policy | auditing aanzetten |
| `secpol.msc` | Local Security Policy | auditing, gastaccount-policy |
| `explorer C:\Fileserver` | Verkenner | mappen |

Server Manager → **Tools** (rechtsboven) heeft ze ook allemaal. Server Manager → **File and Storage Services** → **Shares** toont alle shares + pad.

---

## 2. Gebruikers en groepen vinden

### Klikken
1. `lusrmgr.msc` → **Users**: lijst met Name, Full Name, Description. Pijltje-omlaag op icoon = uitgeschakeld.
2. Dubbelklik gebruiker → **General**: Full name, ☐ *Account is disabled*. → **Member Of**: groepen.
3. **Groups** → dubbelklik groep → **Members**.

In de proeftoets zie je:
```
Administrator, AnnieStryker, AyoubOzcan, BjornHentges, BramBurg, DefaultAccount, Guest,
HannekeCool, SalmaDemirci, TjeerdNickson, WDAGUtilityAccount, YounesBulut
Groep Administratie: SalmaDemirci, TjeerdNickson, YounesBulut      <- Younes hoort hier niet
```
`Administrator`, `DefaultAccount`, `WDAGUtilityAccount` zijn standaard Windows-accounts → **geen** afwijking (wel kun je noemen dat Administrator een sterk wachtwoord moet hebben).

### CMD
```cmd
net user                          :: alle gebruikers
net user Guest                    :: "Account active  Yes" = FOUT
net user BramBurg                 :: details + "Local Group Memberships"
net localgroup                    :: alle groepen
net localgroup Administratie      :: leden
net localgroup Sales
whoami /all                       :: wie ben ik
wmic useraccount get name,fullname,disabled
```

### PowerShell
```powershell
Get-LocalUser | Format-Table Name, FullName, Enabled, LastLogon, PasswordRequired
Get-LocalUser -Name Guest | Select *
Get-LocalGroup
Get-LocalGroupMember -Group Administratie
# alle afdelingsgroepen met leden in één keer:
foreach ($g in "Administratie","Magazijn","Management","Sales","Iedereen") { "== $g"; (Get-LocalGroupMember $g).Name }
# in welke groepen zit één gebruiker:
Get-LocalGroup | ? { (Get-LocalGroupMember $_ -EA SilentlyContinue).Name -match "YounesBulut" } | select Name
```

---

## 3. Rechten op mappen vinden (Share + NTFS)

> **Twee lagen rechten.** Via het netwerk geldt de **strengste** van Share- en NTFS-rechten. Bekijk **allebei**, voor **alle 5** mappen.

### Klikken
**Share-rechten:** Verkenner → `C:\Fileserver\Sales` → rechtermuis → **Properties** → **Sharing** → **Advanced Sharing...** → **Permissions**.
Proeftoets Sales: `Management (SRV001\Management)`, `Sales (SRV001\Sales)`, `AnnieStryker (SRV001\AnnieStryker)` ← losse gebruiker = fout.

**NTFS-rechten:** zelfde map → **Properties** → **Security** → (Edit / **Advanced**).
In Advanced: tabblad **Effective Access** → *Select a user* → bv. Guest → **View effective access** = wat iemand echt mag.

**Alle shares in één scherm:** `fsmgmt.msc` → **Shares** (rechtermuis share → Properties → Share Permissions / Security).
Ook: **Sessions** (wie is nu verbonden) en **Open Files**.

### CMD / PowerShell
```cmd
net share                                    :: alle shares + pad
net share Management                         :: details incl. "Permission"
icacls C:\Fileserver\Management              :: NTFS
icacls C:\Fileserver\*                       :: alle mappen
```
```powershell
Get-SmbShare                                                       # alle shares
Get-SmbShareAccess -Name Management                                # share-rechten
Get-SmbShare | ? Name -notmatch '\$$' | Get-SmbShareAccess | ft Name, AccountName, AccessControlType, AccessRight -AutoSize
Get-ChildItem C:\Fileserver -Directory | % { "== $($_.Name)"; (Get-Acl $_.FullName).Access | ft IdentityReference, FileSystemRights, AccessControlType -AutoSize }
Get-SmbSession                                                     # wie is verbonden
Get-SmbOpenFile                                                    # welke bestanden staan open
```
(`$` aan het eind = verborgen admin-shares als `C$`, `ADMIN$`, `IPC$` → negeren.)

Waar let je op:
| Zie je | Fout? |
|---|---|
| `Everyone` – Full Control | ❌ te ruim |
| `Guest` | ❌ |
| losse gebruiker (AnnieStryker) | ❌ rechten direct aan persoon |
| groep van andere afdeling (Sales op Administratie) | ❌ |
| eigen afdelingsgroep ontbreekt | ❌ (afdeling kan er niet bij) |
| Management op alles | ✔ volgens beleid |
| Administrators / SYSTEM in NTFS | ✔ normaal |

---

## 4. Logs: Event Viewer

`eventvwr.msc` → **Windows Logs** → **Security** → rechts **Filter Current Log...** → bij `<All Event IDs>` invullen: `4624,4625` → OK. Dan **Find...** → `Guest`.

| ID | Betekenis |
|---|---|
| 4624 | Succesvol aangemeld (Logon Type **3** = via netwerk/share, 2 = lokaal, 10 = RDP) |
| 4625 | Mislukte aanmelding |
| 4720 / 4726 | Account aangemaakt / verwijderd |
| 4722 / 4725 | Account ingeschakeld / uitgeschakeld |
| 4738 | Account gewijzigd |
| 4732 / 4733 | Toegevoegd aan / verwijderd uit lokale groep |
| 5140 | Netwerkshare benaderd (alleen met auditing) |
| 5145 | Share-object gecontroleerd (alleen met auditing) |
| 4663 | Bestand benaderd (alleen met object-auditing) |

PowerShell:
```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} | ? { $_.Message -match 'Guest' } | select TimeCreated, Id -First 20
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624,4625} -MaxEvents 50 | ft TimeCreated, Id, Message -Wrap
```
Proeftoets: **Guest ingelogd op 8 mei 2023 om 08:21**. Bestandstoegang werd niet gelogd → niet vast te stellen wat er is ingezien → **mogelijk datalek**.

### Auditing aanzetten (advies én kun je doen)
`gpedit.msc` → Computer Configuration → Windows Settings → Security Settings → **Advanced Audit Policy Configuration** → System Audit Policies → **Object Access** → *Audit File Share* + *Audit File System* → ☑ Success ☑ Failure.
```cmd
auditpol /set /subcategory:"File Share" /success:enable /failure:enable
auditpol /set /subcategory:"File System" /success:enable /failure:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
```
(Voor File System ook op de map: Properties → Security → Advanced → **Auditing** → Add → Everyone.)

---

## 5. Oplossen: klikpaden

### 5.1 Kwetsbaarheid 1 · Guest actief → uitschakelen
`lusrmgr.msc` → Users → dubbelklik **Guest** → ☑ **Account is disabled** → **Apply** → OK. Screenshot.
(Of: `secpol.msc` → Local Policies → Security Options → *Accounts: Guest account status* → **Disabled**.)

### 5.2 Kwetsbaarheid 2 · Guest heeft rechten op mappen → weghalen
Per map (begin bij **Management**, daarna de andere 4 controleren):
1. Properties → **Sharing** → **Advanced Sharing** → **Permissions** → klik **Guest** (of Everyone) → **Remove** → Apply → OK.
2. Properties → **Security** → **Edit...** → Guest/Everyone → **Remove** → OK.
   (Grijs/kan niet? → **Advanced** → **Disable inheritance** → *Convert inherited permissions* → dan Remove.)

### 5.3 Kwetsbaarheid 3 · Management te ruim → alleen groep Management
1. Sharing → Advanced Sharing → Permissions: alleen **Management** (Full Control). Rest Remove. Ontbreekt Management: **Add...** → typ `Management` → **Check Names** → OK → vink Full Control.
2. Security → Advanced → **Disable inheritance** → Convert → verwijder Everyone/Users/Guest → houd **Management (Modify)**, **Administrators (Full)**, **SYSTEM (Full)**.
3. Eigenaar: Security → Advanced → bovenaan **Owner: Change** → `Administrators` → ☑ *Replace owner on subcontainers and objects* → OK.

---

## 6. Oplossen: overige afwijkingen uit W1

| Afwijking | Klikken | Commando |
|---|---|---|
| BramBurg staat niet op lijst | Users → BramBurg → ☑ Account is disabled (na akkoord: rechtermuis → Delete) | `net user BramBurg /active:no` / `net user BramBurg /delete` |
| YounesBulut in Administratie | Groups → Administratie → YounesBulut → Remove | `net localgroup Administratie YounesBulut /delete` |
| AnnieStryker direct op shares | Administratie, Iedereen, Sales → Share Permissions → AnnieStryker → Remove | `Revoke-SmbShareAccess -Name Sales -AccountName AnnieStryker -Force` |
| Groep Sales op Administratie | Administratie → Share Permissions → Sales → Remove | `Revoke-SmbShareAccess -Name Administratie -AccountName Sales -Force` |
| Afdeling mist op eigen share | Permissions → Add → groep → Check Names → Change+Read | `Grant-SmbShareAccess -Name Sales -AccountName Sales -AccessRight Change -Force` |
| Namen Salma/Ayoub | Users → dubbelklik → **Full name**: `Salma Aslan` | `Set-LocalUser -Name SalmaDemirci -FullName "Salma Aslan"` |
| Gebruikersnaam zelf | rechtermuis → **Rename** (SID/rechten blijven, profielmap niet) | `Rename-LocalUser -Name SalmaDemirci -NewName SalmaAslan` |

Voorbeelduitwerking zegt: *"Gebruikersnamen kunnen niet zomaar aangepast worden; dan moet de gebruiker opnieuw aangemaakt worden en de data omgezet."* → Pas de **Full name** aan en geef advies over de gebruikersnaam.

---

## 7. Oplossen: kant-en-klaar in PowerShell

Controleer de namen eerst met `Get-LocalUser` / `Get-SmbShare`. Elke regel apart uitvoeren en screenshotten.

```powershell
Get-Date                                         # tijd logboek

# === W2 kwetsbaarheid 1: Guest uit ===
Disable-LocalUser -Name Guest
Get-LocalUser Guest | select Name, Enabled       # Enabled False

# === W2 kwetsbaarheid 2: Guest van alle shares + NTFS af ===
Get-SmbShare | ? Name -notmatch '\$$' | % { Revoke-SmbShareAccess -Name $_.Name -AccountName Guest -Force -EA SilentlyContinue }
Get-ChildItem C:\Fileserver -Directory | % { icacls $_.FullName /remove Guest }

# === W2 kwetsbaarheid 3: Management alleen voor Management ===
Revoke-SmbShareAccess -Name Management -AccountName Everyone -Force -EA SilentlyContinue
Grant-SmbShareAccess  -Name Management -AccountName Management -AccessRight Full -Force
icacls C:\Fileserver\Management /inheritance:r /grant "Management:(OI)(CI)M" "Administrators:(OI)(CI)F" "SYSTEM:(OI)(CI)F" /remove Everyone Users Guest
icacls C:\Fileserver\Management /setowner Administrators /T
Get-SmbShareAccess -Name Management ; icacls C:\Fileserver\Management

# === W1 overige ===
Disable-LocalUser -Name BramBurg
Remove-LocalGroupMember -Group Administratie -Member YounesBulut
foreach ($s in "Administratie","Iedereen","Sales") { Revoke-SmbShareAccess -Name $s -AccountName AnnieStryker -Force }
Revoke-SmbShareAccess -Name Administratie -AccountName Sales -Force
Set-LocalUser -Name SalmaDemirci -FullName "Salma Aslan"
Set-LocalUser -Name AyoubOzcan  -FullName "Ayoub Ahmadi"
```
`(OI)(CI)` = geldt ook voor submappen en bestanden. `M` = Modify, `F` = Full, `RX` = lezen.
Share-rechten: `-AccessRight Full | Change | Read`.

---

## 8. Testen

```cmd
net user Guest | findstr /i active                   :: Account active   No
net use \\SRV001\Management /user:SRV001\Guest ""   :: System error 1331 (account disabled) = goed
net use \\localhost\Management /user:SRV001\AnnieStryker *     :: wachtwoord vragen -> Access is denied
net use * /delete /y                                 :: verbindingen opruimen
```
- **Effective Access**: map → Properties → Security → Advanced → **Effective Access** → *Select a user* → `Guest` / `AnnieStryker` → **View effective access** → alles ✘ = screenshot.
- Hanneke moet er **wel** in: Effective Access voor `HannekeCool` op Management → ✔.
- Uitloggen, inloggen als AnnieStryker → `\\SRV001\Management` → *"You do not have permission"* → screenshot.

---

## 9. Controlescript: alles in één keer

Maak in PowerShell ISE (`powershell_ise`) of Notepad: `C:\audit.ps1`
```powershell
"===== $env:COMPUTERNAME - $(Get-Date) ====="
"`n== 1. GEBRUIKERS =="
Get-LocalUser | ft Name, FullName, Enabled, LastLogon -AutoSize
"== 2. GROEPEN =="
foreach ($g in Get-LocalGroup | ? Name -in "Administratie","Magazijn","Management","Sales","Iedereen") {
  "$($g.Name): " + ((Get-LocalGroupMember $g).Name -join ", ") }
"`n== 3. SHARE-RECHTEN =="
Get-SmbShare | ? Name -notmatch '\$$' | Get-SmbShareAccess | ft Name, AccountName, AccessControlType, AccessRight -AutoSize
"== 4. NTFS-RECHTEN =="
Get-ChildItem C:\Fileserver -Directory | % { "-- $($_.Name)"; (Get-Acl $_.FullName).Access | ft IdentityReference, FileSystemRights, AccessControlType -AutoSize }
"== 5. GUEST LOGINS =="
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -EA SilentlyContinue | ? Message -match 'Guest' | select TimeCreated -First 10
```
Uitvoeren:
```powershell
Set-ExecutionPolicy -Scope Process Bypass -Force
C:\audit.ps1 | Out-File C:\audit_voor.txt ; notepad C:\audit_voor.txt
```
Na de fixes nog eens → `C:\audit_na.txt` → bewijs voor logboek.

---

## 10. Domeincontroller? (Active Directory)

Werkt `lusrmgr.msc` niet ("This snap-in may not be used with this edition / on a domain controller")? Dan zitten gebruikers in **AD**:
- `dsa.msc` → domein → **Users** (of een OU per afdeling). Guest → rechtermuis → **Disable Account**.
- Groep → dubbelklik → **Members**. Gebruiker → **Member Of**.
```powershell
Get-ADUser -Filter * -Properties DisplayName, Enabled, LastLogonDate | ft Name, DisplayName, Enabled, LastLogonDate
Get-ADGroupMember -Identity Sales | select Name
Disable-ADAccount -Identity Guest
Disable-ADAccount -Identity BramBurg
Remove-ADGroupMember -Identity Administratie -Members YounesBulut -Confirm:$false
Set-ADUser SalmaDemirci -DisplayName "Salma Aslan" -Surname "Aslan"
```
