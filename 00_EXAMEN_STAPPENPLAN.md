# 00 · Stappenplan examen P2-K2 (begin hier!)

> Uitgangspunt: het examen is (bijna) hetzelfde als de proeftoets. Casus **Handelsbedrijf Conley**, directeur **Hanneke Cool**, server **SRV001**, map `/home/fileserver` (Linux) of `C:\Fileserver` (Windows).
> Klopt een naam niet? Dan volg je gewoon dezelfde stappen met de namen uit jouw opdracht.
>
> Alle commando's staan uitgebreid in [01_Linux_Samba_compleet.md](01_Linux_Samba_compleet.md) en [02_Windows_Server_compleet.md](02_Windows_Server_compleet.md).
> Alle teksten die je kunt overnemen staan in [03_Kant-en-klare_teksten.md](03_Kant-en-klare_teksten.md).
> Wat je **zelf moet bedenken** (wordt niet gevraagd, wel beoordeeld): [04_Denk-eraan_checklist.md](04_Denk-eraan_checklist.md).

---

## Wat je verwacht te vinden (proeftoets = examen)

### De personeelslijst uit het interview (= de waarheid)

| Medewerker | Afdeling | Linux-gebruiker | Windows-gebruiker |
|---|---|---|---|
| Tjeerd Nickson | Administratie | `tjeerdnickson` | `TjeerdNickson` |
| Salma **Aslan** | Administratie | `salmademirci` ❌ | `SalmaDemirci` ❌ |
| Bjorn Hentges | Magazijn | `bjornhentges` | `BjornHentges` |
| Ayoub **Ahmadi** | Magazijn | `ayoubozcan` ❌ | `AyoubOzcan` ❌ |
| Hanneke Cool | Management | `hannekecool` | `HannekeCool` |
| Younes Bulut | Sales | `younesbulut` (ook in administratie ❌) | `YounesBulut` (ook in Administratie ❌) |
| Annie Stryker | Sales | `anniestryker` | `AnnieStryker` |
| *niet op lijst* | – | `bramburg` ❌ | `BramBurg` ❌ |
| *niet op lijst* | – | `gast` ❌ | `Guest` (actief) ❌ |

Interview: **Salma** (Administratie) houdt zich bezig met de AVG. Onderlegger zegt: datalek melden bij **Bjorn Hentges**. → Lees in de opdracht wie het is!

### Groepen zoals ze op de server staan (Linux `/etc/group`)
```
administratie:x:1010:tjeerdnickson,younesbulut,salmademirci      <- younesbulut hoort hier NIET
magazijn:x:1011:bjornhentges,ayoubozcan
management:x:1012:hannekecool
sales:x:1013:younesbulut,anniestryker,bramburg                    <- bramburg staat niet op de lijst
iedereen:x:1014:tjeerdnickson,younesbulut,salmademirci,bjornhentges,ayoubozcan,hannekecool,anniestryker,bramburg
```

### Rechten op de shares zoals ze op de server staan

| Share | Linux `valid users` (smb.conf) | Windows Share Permissions | Fout |
|---|---|---|---|
| administratie | `@administratie, @management, anniestryker` | Administratie, Management, **Sales**, **AnnieStryker** | Annie direct; Sales-groep (Win) |
| iedereen | `@iedereen, anniestryker` | Iedereen, **AnnieStryker** | Annie direct |
| magazijn | `@magazijn, @management` | Magazijn, Management | ✔ |
| management | `@management` + **gast** | Management, **Guest** | Gast! |
| sales | `@sales, @management, @administratie, anniestryker` | Management, Sales, **AnnieStryker** | Annie direct; @administratie (Linux, "op verzoek van Salma") |

Linux extra: `ls -la /home/fileserver` → map **management** heeft `drwxrwxrwx` (777) = te ruim.

---

## WERKPROCES 1 (opdracht 1): Auditrapport (± 2 uur)

### Stap 1 · Voorbereiden (5 min)
- [ ] Interview lezen, personeelslijst overnemen (tabel hierboven).
- [ ] Auditrapport-sjabloon openen, algemene gegevens invullen (Opdrachtgever: Hanneke Cool namens Handelsbedrijf Conley, Auteur: jouw naam, Datum, Versie 1.0).
- [ ] Logboek mag ook al open (handig voor W2).

### Stap 2 · Linux: gebruikers, groepen, rechten (20 min) — maak van ELK commando een screenshot
```bash
sudo -i
awk -F: '$3>=1000 && $3<65534 {print $1}' /etc/passwd    # 1. echte gebruikers
grep -E 'gast|guest' /etc/passwd                          #    is er een gastaccount?
pdbedit -L -v | grep -E 'Unix username|Account Flags'     # 2. Samba-gebruikers (D = uit)
grep -E '^(administratie|magazijn|management|sales|iedereen):' /etc/group   # 3. groepen
ls -la /home/fileserver                                   # 4. bestandsrechten (777?)
testparm -s 2>/dev/null                                   # 5. ALLE shares + valid users + guest
grep -n -E '^\[|valid users|guest|#' /etc/samba/smb.conf  #    snel overzicht incl. commentaar
smbclient -L //localhost -N                               # 6. test: lukt het zonder wachtwoord?
```
Of in één keer het controlescript uit [01_Linux_Samba_compleet.md §9](01_Linux_Samba_compleet.md#9-controlescript-alles-in-één-keer).

### Stap 3 · Windows: gebruikers, groepen, rechten (20 min) — screenshot van elk venster
1. `Win+R` → `lusrmgr.msc` → **Users** → screenshot (let op: Guest, BramBurg, SalmaDemirci, AyoubOzcan).
2. Dubbelklik **Guest** → staat ☐ *Account is disabled* UIT? Dan is gast actief → screenshot.
3. **Groups** → dubbelklik Administratie, Magazijn, Management, Sales, Iedereen → screenshot van leden.
4. Verkenner → `C:\Fileserver\<map>` → rechtermuis **Properties** → **Sharing** → **Advanced Sharing** → **Permissions** → screenshot. **Voor alle 5 de mappen.**
5. Zelfde map → tabblad **Security** → screenshot (NTFS).
6. Of in PowerShell (als Administrator), in één keer:
```powershell
Get-LocalUser | ft Name,FullName,Enabled
foreach ($g in "Administratie","Magazijn","Management","Sales","Iedereen") { "== $g"; Get-LocalGroupMember $g | select -Expand Name }
Get-SmbShare | ? Name -notmatch '\$$' | Get-SmbShareAccess | ft Name,AccountName,AccessControlType,AccessRight
```

### Stap 4 · Afwijkingen opschrijven (10 min)
Loop deze lijst af, elk "ja" is een probleem voor in je rapport:
- [ ] Iemand op de server die **niet op de lijst** staat? → Bram Burg
- [ ] **Gast/Guest** actief? Heeft hij rechten? → ja, op Management
- [ ] Iemand in een **groep van een andere afdeling**? → Younes in Administratie
- [ ] Naam op de server **anders** dan op de lijst? → Salma (Demirci), Ayoub (Özcan)
- [ ] Een **losse gebruiker** in rechten i.p.v. een groep? → Annie Stryker op 3 shares
- [ ] Een **verkeerde groep** op een share? → Sales op Administratie (Win), Administratie op Sales (Linux)
- [ ] Een map met **777 / Everyone Full Control**? → management (Linux)
- [ ] Management overal bij? → mag (directie), noem het als "conform beleid"

### Stap 5 · Rapport schrijven (45 min)
Neem de tekst over uit [03 §1 Auditrapport](03_Kant-en-klare_teksten.md#1-auditrapport-werkproces-1). Alle 5 onderdelen:
1. Overzicht te testen onderdelen (+ **gebruikte tools met motivatie**) met tabel gebruikers/groepen en tabel rechten op mappen + screenshots
2. Conclusie
3. Kwetsbaarheden en bedreigingen: **minstens 5**, met **impact**, **gesorteerd op prioriteit**
4. Verbetervoorstel: wat + waarom + hoe + wanneer (technisch én beleid)
5. Checklist testen (Ja/Nee)

### Stap 6 · Nakijken (5 min) met de rubric onderaan [W1_cheatsheet](W1_cheatsheet_auditrapport.md#7-zelf-nakijken-beoordelingsformulier-w1).

---

## WERKPROCES 2 (opdracht 2): Incident oplossen (± 2 uur)

Zwakheden (bijlage 2.3): *"Gast toegang staat open op alle mappen"* en *"Eigenaarschap van de mappen is verkeerd geconfigureerd"*.
Dat zijn volgens de onderlegger **3 kwetsbaarheden**:
1. Het **gastaccount is actief**
2. Het **gastaccount heeft rechten op mappen**
3. De rechten op de map **Management** zijn te ruim

### Stap 1 · Logboek starten — schrijf bij ELKE actie de tijd op (`date` in Linux, klok in Windows)

### Stap 2 · Vóór-screenshots (bewijs dat het probleem er is)
Linux:
```bash
date
pdbedit -L -v gast | grep -E 'Unix username|Account Flags'   # geen D = actief
passwd -S gast                                               # P = actief
grep -n -E '^\[|valid users|guest|map to guest' /etc/samba/smb.conf
ls -la /home/fileserver
smbclient -L //localhost -N
smbclient //localhost/management -U gast -c 'ls'
```
Windows:
```powershell
Get-Date
net user Guest | findstr /i "active"
Get-SmbShare | ? Name -notmatch '\$$' | Get-SmbShareAccess | ft Name,AccountName,AccessRight
icacls C:\Fileserver\Management
```

### Stap 3 · Is er al misbruik geweest? (dit wordt je niet gevraagd, wel belangrijk!)
Linux: `last gast`, `grep -i gast /var/log/auth.log`, `ls -la /var/log/samba/`
Windows: `eventvwr.msc` → Windows Logs → Security → Filter **4624** → zoek Guest. Proeftoets: **Guest ingelogd 8 mei 2023 08:21**.

### Stap 4 · Oplossen — kant-en-klaar

**Linux (alles):**
```bash
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak            # backup!

# Kwetsbaarheid 1: gast actief -> uit
smbpasswd -d gast
usermod -L -s /usr/sbin/nologin gast

# Kwetsbaarheid 2: gast rechten op mappen -> weg
nano /etc/samba/smb.conf
#   [global]       : map to guest = never
#   elke share     : guest ok = no
#   [management]   : valid users = @management     (gast weg)
testparm -s
systemctl restart smbd nmbd

# Kwetsbaarheid 3: management te ruim / eigenaarschap fout -> 770 met groep management
chown -R root:management /home/fileserver/management
chmod 2770 /home/fileserver/management
# en meteen alle afdelingsmappen goed zetten:
for a in administratie magazijn sales; do chown -R root:$a /home/fileserver/$a; chmod 2770 /home/fileserver/$a; done
chown -R root:iedereen /home/fileserver/iedereen; chmod 2770 /home/fileserver/iedereen
```
**Windows (alles, PowerShell als Administrator):**
```powershell
# Kwetsbaarheid 1
Disable-LocalUser -Name Guest
# Kwetsbaarheid 2 (op elke share proberen; foutmelding = stond er niet op)
Get-SmbShare | ? Name -notmatch '\$$' | % { Revoke-SmbShareAccess -Name $_.Name -AccountName Guest -Force -EA SilentlyContinue }
Revoke-SmbShareAccess -Name Management -AccountName Everyone -Force -EA SilentlyContinue   # alleen als Everyone erop staat
# LET OP: haal Everyone niet blind van ALLE shares; staat er geen afdelingsgroep op, dan kan niemand er meer in.
icacls C:\Fileserver\Management /remove Guest Everyone
# Kwetsbaarheid 3: Management alleen voor groep Management
icacls C:\Fileserver\Management /inheritance:r /grant "Management:(OI)(CI)M" "Administrators:(OI)(CI)F" "SYSTEM:(OI)(CI)F"
```
Liever klikken? Zie [02 §5](02_Windows_Server_compleet.md#5-oplossen-klikpaden).

### Stap 5 · Testen (ná-screenshots) — zonder test geen volle punten!
Linux:
```bash
pdbedit -L -v gast | grep 'Account Flags'          # [DU    ] -> D = disabled
smbclient -L //localhost -N                        # NT_STATUS_ACCESS_DENIED / LOGON_FAILURE
smbclient //localhost/management -U gast           # geweigerd
ls -ld /home/fileserver/management                 # drwxrws--- root management
sudo -u anniestryker ls /home/fileserver/management   # Permission denied
smbclient //localhost/management -U hannekecool -c 'ls'   # Hanneke kan er WEL in
```
Windows:
```powershell
net user Guest | findstr /i active                 # Account active  No
Get-SmbShareAccess -Name Management                # geen Guest/Everyone meer
icacls C:\Fileserver\Management                    # alleen Management, Administrators, SYSTEM
```
+ map → Properties → Security → Advanced → **Effective Access** → Guest → *View effective access* → alles ✘ → screenshot.

### Stap 6 · Extra fixes (als tijd over: laat zien dat je meedenkt)
Bram Burg uitschakelen, Younes uit Administratie, Annie direct weg, namen aanpassen. Commando's: [01 §7](01_Linux_Samba_compleet.md#7-oplossen-overige-afwijkingen-uit-w1) / [02 §6](02_Windows_Server_compleet.md#6-oplossen-overige-afwijkingen-uit-w1).

### Stap 7 · Documenten (45 min) — tekst in [03_Kant-en-klare_teksten.md](03_Kant-en-klare_teksten.md)
- [ ] **Onderzoeksverslag**: aanleiding, uitgevoerd onderzoek, analyse + categorie per kwetsbaarheid, conclusie, aanpassingen met screenshots, communicatiematrix
- [ ] **Logboek**: datum + tijden + actie + screenshot per regel
- [ ] **E-mail**: Van (jouw adres) / Aan (Hanneke Cool) / CC (AVG-verantwoordelijke) / onderwerp / **bijlage onderzoeksverslag** / zin "deze mail is verstuurd aan … omdat …"
