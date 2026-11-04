# Cheatsheet W1: Geeft security advies en verbetert de security

> P2-K2 Controleert de security, opdracht 1. Wat je maakt: een **auditrapport** (sjabloon bijlage 1.2).
> Max 12 punten: 6 pt = 5,5 / 8 pt = 7,0 / 10 pt = 8,5 / 12 pt = 10.
> Casus in de proeftoets: SecureIT controleert fileserver SRV001 van Handelsbedrijf Conley (directeur Hanneke Cool).
> Op het echte examen zijn de namen en fouten waarschijnlijk anders. **Leer de methode, niet de antwoorden.**

---

## 0. Het examen in één oogopslag

Je krijgt:
1. Een **interview** met een **personeelslijst** (wie werkt er, op welke afdeling). Dit is "het beleid", de waarheid.
2. Een server (Linux met Samba en/of Windows Server) met een fileserver-map (`/home/fileserver` of `C:\Fileserver`).

Jij doet: **vergelijk de lijst met wat er écht op de server staat**. Elk verschil = een afwijking = een beveiligingsprobleem.

Je kijkt naar 3 dingen:

| # | Wat | Linux | Windows |
|---|---|---|---|
| 1 | Gebruikers | `/etc/passwd`, `pdbedit -L` | Local Users and Groups → Users |
| 2 | Groepen (wie zit waar in) | `/etc/group` | Local Users and Groups → Groups |
| 3 | Rechten op mappen | `/etc/samba/smb.conf` + `ls -la` | Map → Properties → Sharing → Advanced Sharing → Permissions (+ tabblad Security) |

**Grootste fout uit mijn vorige poging:** alleen `/etc/passwd`, `/etc/group` en `ls -la` gedaan. Ik had `smb.conf` **niet** bekeken. Daar stonden juist de fouten in de rechten (gebruiker direct op een share, gast op management, verkeerde groep op een share). **Altijd `smb.conf` lezen!**

---

## 1. Stappenplan op het examen (± 2 uur)

| Stap | Wat | Tijd |
|---|---|---|
| 1 | Interview lezen, personeelslijst overnemen in een tabel (naam, afdeling). Noteer wie verantwoordelijk is voor AVG. | 10 min |
| 2 | Gebruikers bekijken + screenshot | 10 min |
| 3 | Groepen + leden bekijken + screenshot | 10 min |
| 4 | Rechten op mappen bekijken (smb.conf / share permissions / NTFS) + screenshots | 20 min |
| 5 | Afwijkingenlijst maken: voor elke gebruiker, groep en map: klopt het met de lijst? | 15 min |
| 6 | Auditrapport invullen (alle 5 onderdelen) | 45 min |
| 7 | Nakijken met de rubric hieronder (paragraaf 7) | 10 min |

**Screenshot elke stap** en zet hem in het rapport met een zin eronder wat je ziet.

---

## 2. Linux + Samba commando's

> Eerst root worden: `sudo -i` (of `su -`). Werkt een commando niet, zet er `sudo` voor.

### 2.1 Gebruikers

```bash
cat /etc/passwd                          # alle gebruikers (ook systeemaccounts)
awk -F: '$3>=1000 {print $1, $5}' /etc/passwd   # alleen echte gebruikers + volledige naam
cut -d: -f1,5,7 /etc/passwd              # gebruikersnaam : volledige naam : shell
getent passwd                            # zelfde, ook LDAP/domein-gebruikers
id annie                                 # uid, gid en ALLE groepen van één gebruiker
groups younesbulut                       # in welke groepen zit iemand
sudo passwd -S gast                      # status: P = wachtwoord actief, L = geblokkeerd
sudo chage -l gast                       # verloopdatum account/wachtwoord
lastlog                                  # wanneer heeft iedere gebruiker voor het laatst ingelogd
last                                     # login-geschiedenis
```

Uitleg regel `/etc/passwd`:
```
salmademirci:x:1002:1002:Salma Demirci,,,:/home/salmademirci:/bin/bash
  naam      ww uid  gid  volledige naam    homemap          shell
```
- Kijk naar de **volledige naam** (5e veld): klopt die met het interview? (Salma *Demirci* ≠ Salma *Aslan* → afwijking)
- Shell `/usr/sbin/nologin` of `/bin/false` = kan niet inloggen op de shell.
- Gebruiker die **niet op de lijst** staat (bv. `bramburg`, `gast`) → afwijking.

### 2.2 Samba-gebruikers (die mogen inloggen op de shares!)

```bash
sudo pdbedit -L                          # alle Samba-gebruikers
sudo pdbedit -L -v                       # met details (Account Flags: [D] = disabled, [U] = user)
sudo pdbedit -L -v gast                  # details van één gebruiker
```
Let op: een Linux-gebruiker is **niet automatisch** een Samba-gebruiker. Check beide.

### 2.3 Groepen

```bash
cat /etc/group                           # alle groepen + leden (4e veld)
grep -E 'sales|administratie|magazijn|management|iedereen' /etc/group
getent group sales                       # leden van één groep
```
Uitleg regel `/etc/group`:
```
administratie:x:1010:tjeerdnickson,salmademirci,younesbulut
  groep       ww gid  leden
```
→ `younesbulut` hoort bij Sales, staat hier ook in Administratie = **afwijking**.

### 2.4 Rechten op mappen (bestandssysteem)

```bash
ls -la /home/fileserver                  # rechten, eigenaar en groep van iedere map
ls -ld /home/fileserver/management       # alleen die ene map
stat /home/fileserver/management         # rechten als getal (bv. 0777)
getfacl /home/fileserver/management      # extra ACL-rechten (als die er zijn)
```

Rechten lezen:
```
drwxrwxrwx 2 root management 4096 ... management
 |  |  |       eigenaar groep
 |  |  +-- anderen (others): rwx = IEDEREEN mag alles  -> FOUT
 |  +----- groep
 +-------- eigenaar
```
| Getal | Betekenis | Goed voor afdelingsmap? |
|---|---|---|
| `777` | iedereen alles | **Nee**, te ruim |
| `775` | anderen mogen lezen | Nee |
| `770` | alleen eigenaar + groep | **Ja** |
| `2770` | 770 + nieuwe bestanden krijgen de groep (setgid) | Ja, nog beter |

### 2.5 Samba shares: HIER STAAN DE MEESTE FOUTEN

```bash
cat /etc/samba/smb.conf                                   # hele config
grep -v -E '^\s*(#|;|$)' /etc/samba/smb.conf              # zonder commentaar/lege regels
sudo testparm -s                                          # nette, gecontroleerde config (+ fouten)
grep -A10 '^\[sales\]' /etc/samba/smb.conf                # één share bekijken
grep -n -E 'valid users|guest ok|public|map to guest|guest account|write list|force user' /etc/samba/smb.conf
```

Waar let je op per share:

| Regel | Goed | Fout (afwijking) |
|---|---|---|
| `valid users` | alleen groepen met `@`: `@sales, @management` | een **losse gebruiker** zonder `@` (bv. `anniestryker`) of een **verkeerde groep** (bv. `@administratie` op share sales) of `gast` |
| `guest ok` / `public` | `no` | `yes` → iedereen zonder wachtwoord |
| `[global] map to guest` | `never` (of niet aanwezig) | `bad user` → foute login wordt gast |
| `[global] guest account` | n.v.t. | wijst naar een account met rechten |
| `#commentaar` | – | lees het! Bijv. `#administratie toegevoegd ... op verzoek van Salma` → toegevoegd zonder toestemming directie |

Voorbeeld uit de proeftoets (smb.conf):
```ini
[administratie]
   path = /home/fileserver/administratie
   valid users = @administratie, @management, anniestryker     # <- Annie direct als gebruiker = FOUT
   guest ok = no

[sales]
   path = /home/fileserver/sales
   valid users = @sales, @management, @administratie, anniestryker   # <- @administratie + Annie direct = FOUT
#administratie toegevoegd per 3 januari 2023 op verzoek van Salma

[iedereen]
   valid users = @iedereen, anniestryker                       # <- Annie direct = FOUT (dubbel/onnodig)
```

### 2.6 Testen of je ergens bij kunt (laat zien dat je tools "gebruikt en configureert")

```bash
smbclient -L //localhost -N                       # shares tonen ZONDER wachtwoord (lukt dit = gasttoegang open!)
smbclient //localhost/management -U gast          # inloggen als gast op management
smbclient //localhost/sales -U younesbulut        # mag Younes erin?
sudo smbstatus                                    # wie is nu verbonden met welke share
sudo -u gast ls /home/fileserver/management       # kan gast de map lezen op bestandsniveau?
```

### 2.7 Logbestanden (bewijs van misbruik)

```bash
ls /var/log/samba/                                # per client een log
sudo tail -n 50 /var/log/samba/log.smbd
sudo journalctl -u smbd --since "2023-05-01"
sudo grep -i gast /var/log/auth.log
last gast                                         # heeft gast ingelogd?
```

### 2.8 Handige tools (voor punten bij "Tools kiezen": noem ze + waarom)

| Tool | Waarom gekozen |
|---|---|
| `cat /etc/passwd` / `getent passwd` | Ingebouwd, laat alle accounts zien → vergelijken met personeelslijst |
| `cat /etc/group` / `id` | Laat groepslidmaatschap zien → klopt de afdeling? |
| `ls -la` / `getfacl` | Rechten op bestandsniveau → is een map te ruim (777)? |
| `testparm -s` / `smb.conf` | Rechten per share → wie mag via het netwerk bij welke map |
| `pdbedit -L -v` | Welke accounts kunnen inloggen op Samba, zijn ze disabled? |
| `smbclient` | Praktijktest: kan een account er echt in? (bewijs) |
| `nmap -p 139,445 --script smb-enum-shares localhost` | Extern/gespecialiseerd: ziet een aanvaller shares en gasttoegang? |

---

## 3. Windows Server: waar vind je alles?

### 3.1 Gebruikers en groepen (GUI)

| Wat | Waar (klik) | Snel openen (Win+R) |
|---|---|---|
| Gebruikers | Server Manager → Tools → **Computer Management** → System Tools → **Local Users and Groups** → **Users** | `lusrmgr.msc` of `compmgmt.msc` |
| Volledige naam, uitgeschakeld? | Dubbelklik gebruiker → tabblad **General** (Full name, ☐ *Account is disabled*) | |
| In welke groepen zit iemand | Dubbelklik gebruiker → tabblad **Member Of** | |
| Leden van een groep | Local Users and Groups → **Groups** → dubbelklik groep (bv. Administratie) | |
| Gast-account | Users → **Guest** (pijltje omlaag op icoon = uitgeschakeld) | |
| **Is het een domeincontroller?** Dan bestaat Local Users and Groups niet. Gebruik **Active Directory Users and Computers** | `dsa.msc` |

### 3.2 Rechten op mappen (GUI): twee soorten rechten!

1. **Share-rechten** (via netwerk):
   `C:\Fileserver\Sales` → rechtermuis → **Properties** → tabblad **Sharing** → **Advanced Sharing** → **Permissions**
2. **NTFS-rechten** (op schijf):
   zelfde map → **Properties** → tabblad **Security** → **Advanced** (→ tabblad *Effective Access* om te testen voor één gebruiker)
3. **Alle shares in één overzicht**:
   Computer Management → System Tools → **Shared Folders** → **Shares** (`fsmgmt.msc`)
   Of: Server Manager → **File and Storage Services** → **Shares**.
   Onder *Shared Folders* staan ook **Sessions** en **Open Files** (wie is nu verbonden).

**Beide** bekijken. De strengste van de twee wint. Let op: `Everyone`, `Guest`, losse gebruikers (Annie Stryker) en verkeerde groepen.

### 3.3 Windows commando's (CMD)

```cmd
net user                               :: alle gebruikers
net user Guest                         :: details: "Account active  Yes/No", lokale groepen
net user annie.stryker                 :: details van één gebruiker
net localgroup                         :: alle groepen
net localgroup Administratie           :: leden van groep
net share                              :: alle shares + pad
net share Sales                        :: details share (incl. rechten)
whoami /groups                         :: groepen van jezelf
icacls C:\Fileserver\Management        :: NTFS-rechten
wmic useraccount get name,fullname,disabled
```

### 3.4 Windows commando's (PowerShell, als Administrator)

```powershell
Get-LocalUser | Select Name, FullName, Enabled, LastLogon     # gebruikers + aan/uit
Get-LocalGroup                                                  # groepen
Get-LocalGroupMember -Group "Sales"                             # leden
Get-SmbShare                                                    # alle shares
Get-SmbShareAccess -Name "Sales"                                # SHARE-rechten
Get-SmbShare | Get-SmbShareAccess                               # rechten van alle shares in één keer
Get-Acl C:\Fileserver\Management | Format-List                  # NTFS-rechten
Get-SmbSession                                                  # wie is verbonden

# Bij een domein (Active Directory):
Get-ADUser -Filter * -Properties DisplayName, Enabled | Select Name, DisplayName, Enabled
Get-ADGroupMember -Identity "Sales"
```

### 3.5 Logs in Windows (Event Viewer)

`eventvwr.msc` → **Windows Logs** → **Security** → rechts *Filter Current Log* → Event ID invullen.

| Event ID | Betekenis |
|---|---|
| 4624 | Succesvol ingelogd (Logon Type 3 = via netwerk/share) |
| 4625 | Mislukte login |
| 4720 / 4726 | Account aangemaakt / verwijderd |
| 4722 / 4725 | Account ingeschakeld / uitgeschakeld |
| 4732 / 4733 | Lid toegevoegd aan / verwijderd uit lokale groep |
| 5140 | Netwerkshare benaderd (alleen als auditing aan staat) |
| 4663 | Bestand benaderd (alleen als object auditing aan staat) |

---

## 4. Afwijkingen vinden: zo vergelijk je

Maak deze tabel tijdens het zoeken (kladje):

| Personeelslijst | Afdeling | Linux account / groepen | Windows account / groepen | Klopt? |
|---|---|---|---|---|
| Tjeerd Nickson | Administratie | tjeerdnickson / administratie | idem | ✔ |
| Salma Aslan | Administratie | salmademirci (Demirci) | Demirci | ✘ achternaam |
| ... | | | | |
| *(niet op lijst)* | | bramburg / sales | Bram Burg | ✘ |
| *(niet op lijst)* | | gast | Guest actief | ✘ |

Typische fouten die examinatoren inbouwen (checklist):
- [ ] Account van iemand die **niet (meer) op de lijst** staat (oud-medewerker)
- [ ] **Gast**-account actief en/of met rechten op mappen
- [ ] Iemand in **twee afdelingsgroepen** of de verkeerde groep
- [ ] **Naam** in het systeem wijkt af van de lijst
- [ ] Rechten **direct aan een gebruiker** i.p.v. aan een groep
- [ ] **Verkeerde groep** heeft rechten op andermans afdelingsmap
- [ ] Map met **777** of `Everyone: Full Control`
- [ ] `guest ok = yes` / `map to guest = bad user`
- [ ] Afdelingsgroep **mist** juist op zijn eigen share
- [ ] Management heeft rechten op alles → meestal **toegestaan** (directie). Noem het wel, maar niet als fout.

### De afwijkingen uit de proeftoets (ter controle)

| # | Afwijking | Gevonden met |
|---|---|---|
| 1 | Salma heet in het systeem **Demirci** (lijst: Aslan) | `/etc/passwd`, Local Users |
| 2 | Ayoub heet in het systeem **Özcan** (lijst: Ahmadi) | `/etc/passwd`, Local Users |
| 3 | **Younes Bulut** zit in Sales én Administratie | `/etc/group`, Member Of |
| 4 | **Bram Burg** heeft account + in Sales, staat niet op lijst | `/etc/passwd`, `/etc/group` |
| 5 | **Gast**-account actief met rechten (o.a. Management) | `pdbedit`, smb.conf, Share Permissions |
| 6 | **Annie Stryker** heeft rechten direct als gebruiker op Administratie, Iedereen en Sales; groep Sales/Administratie heeft rechten op elkaars map | `smb.conf` → `valid users`, Share Permissions |

---

## 5. Het auditrapport invullen (de 5 onderdelen die beoordeeld worden)

> "Algemene gegevens" (opdrachtgever, auteur, datum, versie) telt **niet** mee, maar vul het wel in.
> Schrijf **formeel/zakelijk**: u-vorm, geen "ik denk", geen afkortingen als "ff", geen smileys.
> Problemen meld je bij de **directeur** (hier: Hanneke Cool).

### Algemene gegevens
```
Opdrachtgever : Hanneke Cool namens Handelsbedrijf Conley
Auteur        : Jafeth Worm, IT expert SecureIT
Datum         : 05-11-2026
Versie        : 1.0
```

### Onderdeel 1: Overzicht van de te testen onderdelen

**1.0 Gebruikte tools** (zet dit erbij, levert punten op bij "Tools kiezen"):

> Voor deze audit zijn de volgende tools gebruikt. Op de Linux-server zijn `/etc/passwd` en `pdbedit` gebruikt om alle gebruikersaccounts te vergelijken met de personeelslijst, `/etc/group` om te controleren of medewerkers in de juiste afdelingsgroep zitten, en `smb.conf` met `testparm` om te zien welke gebruikers en groepen toegang hebben tot welke gedeelde map. Met `ls -la` zijn de bestandsrechten gecontroleerd en met `smbclient` is getest of een account daadwerkelijk toegang krijgt. Op de Windows-server is Computer Management (Local Users and Groups en Shared Folders) gebruikt, aangevuld met de tabbladen Sharing en Security van iedere map. Deze ingebouwde tools zijn gekozen omdat zij direct de actuele configuratie tonen, zonder extra software op de server van de klant te installeren.

**1.1 Gebruikers en groepen**

> Hieronder een samenvatting van de gebruikers en groepen in de Windows-omgeving (Local Users and Groups) en de Linux-omgeving (`/etc/passwd` en `/etc/group`). Afwijkingen ten opzichte van de personeelslijst zijn **vetgedrukt/rood**.

| Medewerker | Afdeling (groep) | Opmerking |
|---|---|---|
| Tjeerd Nickson | Administratie | |
| Salma Aslan | Administratie | **Achternaam in het systeem is Demirci** |
| Bjorn Hentges | Magazijn | |
| Ayoub Ahmadi | Magazijn | **Achternaam in het systeem is Özcan** |
| Hanneke Cool | Management | |
| Younes Bulut | **Sales, Administratie** | **Zit ook in groep Administratie** |
| Annie Stryker | Sales | |
| **Bram Burg** | **Sales** | **Staat niet op de personeelslijst** |
| **Gast** | – | **Account actief, heeft rechten op mappen** |

(+ screenshots `/etc/passwd`, `/etc/group`, Local Users and Groups met korte zin eronder)

**1.2 Rechten op mappen**

> Deze controle is uitgevoerd in `smb.conf` (Linux) en bij de Share- en NTFS-rechten van de mappen (Windows). Rechten die afwijken van de afdeling zijn **vetgedrukt/rood**.

| Map | Toegang toegekend aan | Opmerking |
|---|---|---|
| Administratie | Administratie, Management, **Sales**, **Annie Stryker** | Groep Sales en losse gebruiker horen hier niet |
| Iedereen | Iedereen, **Annie Stryker** | Losse gebruiker, onnodig |
| Magazijn | Magazijn, Management | Correct |
| Management | Management, **Gast** | Gast heeft toegang tot directiegegevens |
| Sales | Sales, Management, **Annie Stryker** (+ **Administratie** in Linux) | Losse gebruiker; extra groep zonder akkoord directie |

### Onderdeel 2: Conclusie

> De rechten in deze serveromgeving zijn niet juist geconfigureerd. Er zijn gebruikers met een andere achternaam dan op de personeelslijst en er is een account actief van een persoon die niet op de personeelslijst voorkomt (Bram Burg). Eén medewerker zit in een groep van een afdeling waar hij niet werkt. Daarnaast is het gastaccount actief en heeft het toegang tot de map Management, waardoor vertrouwelijke directiegegevens zonder wachtwoord bereikbaar zijn. Op meerdere mappen zijn rechten direct aan een medewerker toegekend in plaats van aan een groep. Wij adviseren om de gehele server na te lopen op shares, rechten en gebruikers en de punten hieronder in volgorde van prioriteit op te lossen.

### Onderdeel 3: Kwetsbaarheden en bedreigingen (≥ 5, met impact, **gesorteerd op prioriteit**)

> Voor "Volledig" moet je **5 of meer** problemen hebben, met **goed ingeschatte impact**, en **gesorteerd** (hoogste eerst).

| Prio | Kwetsbaarheid | Bedreiging / impact | Risico |
|---|---|---|---|
| 1 | Gastaccount is actief en heeft toegang tot de map Management | Iedereen kan zonder wachtwoord vertrouwelijke directiegegevens lezen, wijzigen, verwijderen of kopiëren. Mogelijk datalek (AVG). Raakt het hele bedrijf. | **Hoog** |
| 2 | Account van Bram Burg is actief, terwijl hij niet op de personeelslijst staat | Een (oud-)medewerker kan nog steeds bij de gegevens van Sales. Onbevoegde toegang, diefstal van klantgegevens. | **Hoog** |
| 3 | Groep Sales heeft rechten op de map Administratie (en Administratie op Sales, zonder akkoord directie) | Medewerkers zien financiële/administratieve gegevens van een andere afdeling. Raakt meer dan 1 afdeling. | **Hoog** |
| 4 | Younes Bulut zit in de groep Administratie terwijl hij bij Sales werkt | Eén medewerker heeft onnodig toegang tot administratieve gegevens (strijdig met "need-to-know"). | **Middel** |
| 5 | Annie Stryker heeft rechten direct als gebruiker op Administratie, Iedereen en Sales | Onoverzichtelijk beheer; bij vertrek of functiewissel blijven rechten makkelijk staan. Annie kan bij Administratie. | **Middel** |
| 6 | Achternamen van Salma (Demirci) en Ayoub (Özcan) wijken af van de personeelslijst | Onduidelijk of het account bij de juiste persoon hoort; risico op verkeerde koppeling. | **Laag** |
| 7 | (Linux) map Management heeft rechten 777 | Iedere gebruiker op de server kan bij de directiemap. | **Hoog** (zet hem dan bovenaan bij 1) |

Hoe schat je de impact in? Gebruik **BIV** (Beschikbaarheid, Integriteit, Vertrouwelijkheid) en de **omvang**:
- Raakt het **meer dan 1 afdeling** of de **directie**, of kan **iemand van buiten** erbij (gast)? → **Hoog**
- Raakt het **1 afdeling** of **1 extra persoon** met te veel rechten? → **Middel**
- Alleen administratief/onduidelijk, geen directe extra toegang? → **Laag**

### Onderdeel 4: Verbetervoorstel / advies (volledig, onderbouwd én haalbaar)

> Per punt: **wat** je doet, **waarom** (onderbouwing) en **hoe** (haalbaar: concreet commando/klikpad, wie, wanneer).

> **1. Gastaccount uitschakelen en verwijderen van alle mappen (direct)**
> Het gastaccount geeft iedereen zonder wachtwoord toegang tot vertrouwelijke gegevens. Wij adviseren het account uit te schakelen (Windows: Guest → *Account is disabled*; Linux: `smbpasswd -d gast` en `usermod -L gast`) en het te verwijderen uit de rechten op de map Management. In `smb.conf` wordt `guest ok = no` en `map to guest = never` ingesteld. Dit kost circa 15 minuten en heeft geen gevolgen voor medewerkers.
>
> **2. Account van Bram Burg uitschakelen en na akkoord verwijderen (direct)**
> Bram Burg staat niet op de personeelslijst. Wij adviseren eerst bij mevrouw Cool te verifiëren of hij uit dienst is, het account direct uit te schakelen en daarna, na het veiligstellen van zijn bestanden, te verwijderen.
>
> **3. Rechten op mappen alleen via groepen toekennen (deze week)**
> Verwijder de losse gebruiker Annie Stryker uit de rechten van Administratie, Iedereen en Sales; zij heeft via de groep Sales en Iedereen al de juiste toegang. Verwijder de groep Sales van de map Administratie. Leg de toegang van Administratie op Sales alleen vast na schriftelijk akkoord van de directie. Hierdoor blijft het beheer overzichtelijk en verdwijnen rechten automatisch bij een functiewissel.
>
> **4. Groepslidmaatschap corrigeren (deze week)**
> Verwijder Younes Bulut uit de groep Administratie, zodat hij alleen toegang heeft tot de gegevens van Sales.
>
> **5. Namen controleren met de personeelsadministratie**
> Controleer bij Tjeerd Nickson of de accounts van Salma en Ayoub bij de juiste personen horen en pas de volledige naam aan.
>
> **6. Beleid: periodieke controle (structureel)**
> Voer elk kwartaal een controle uit van alle accounts en rechten tegen de actuele personeelslijst, en richt een vaste in- en uitdienstprocedure in, zodat accounts van vertrekkende medewerkers direct worden uitgeschakeld. Schakel auditing van bestandstoegang in, zodat bij een incident achterhaald kan worden wie welke gegevens heeft ingezien.

Tip: noem zowel **technische** (commando's) als **beleidsmatige** (procedure, controle, bewustwording) oplossingen. Dat staat letterlijk in het criterium (T2).

### Onderdeel 5: Checklist testen (zo had de voorbeelduitwerking het)

| Gebruikers en groepen | |
|---|---|
| Zijn alle medewerkers opgenomen als gebruiker? | **Ja** |
| Zijn alle voormalige medewerkers verwijderd? | **Nee** (Bram Burg, gast) |
| Zijn alle gebruikers opgenomen in de juiste groepen? | **Nee** (Younes Bulut) |
| **Rechten op mappen** | |
| Zijn rechten op mappen verleend aan de juiste groepen? | **Ja** (eigen afdeling heeft toegang) |
| Hebben de juiste groepen toegang tot alle mappen? | **Nee** (Sales op Administratie, gast op Management) |
| Zijn alle rechten verleend aan groepen, dus niet direct aan medewerkers? | **Nee** (Annie Stryker) |

> Antwoord alleen **Ja / Nee** en zet de reden tussen haakjes. "Grotendeels" is geen antwoord. Bij één fout = Nee.

---

## 6. Veelgemaakte fouten (van mezelf en volgens de onderlegger)

| Fout | Gevolg | Zo doe je het goed |
|---|---|---|
| `smb.conf` niet bekeken | Fouten in share-rechten gemist → minder dan 5 problemen | Altijd `testparm -s` of `cat /etc/samba/smb.conf` |
| Alleen Linux of alleen Windows | Helft van de afwijkingen mist | Bekijk elke server die je krijgt |
| Geen motivatie bij tools | Max "Enigszins/Grotendeels" | Tabel "tool + waarom" in onderdeel 1 |
| Problemen niet gesorteerd op prioriteit | Geen "Volledig" bij Security analyse | Kolom Prio 1, 2, 3… hoog → laag |
| Checklist "Ja" bij rechten direct aan medewerker | Fout antwoord | Eén losse gebruiker in `valid users` = **Nee** |
| Informele taal, pijltjes en halve zinnen | Communicatie lager | Hele zinnen, u-vorm |
| Advies zonder hoe/waarom | Verbetervoorstel "Enigszins" | Wat + waarom + hoe + wanneer |
| `student`/`ubuntu`-account als fout melden | Kan verwarren | Noem het eventueel als beheeraccount, controleer of het nodig is |

---

## 7. Zelf nakijken (beoordelingsformulier W1)

| Criterium | Volledig (3 pt) als... | ✔ |
|---|---|---|
| Tools kiezen | Meerdere tools gekozen, **geconfigureerd** en gebruikt, **met motivatie**. Gebruikers, groepen én rechten op gedeelde mappen bekeken. | ☐ |
| Communicatie | Alle **5 onderdelen** volledig, deskundig, formeel. Klant snapt het zonder vragen. Gemeld bij de directeur. | ☐ |
| Security analyse | **≥ 5 problemen**, impact goed ingeschat, **gesorteerd op prioriteit**. | ☐ |
| Verbetervoorstel | Helemaal ingevuld, **onderbouwd** en **haalbaar**. | ☐ |
