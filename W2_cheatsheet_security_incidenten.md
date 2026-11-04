# Cheatsheet W2: Reageert op security incidenten

> P2-K2 Controleert de security, opdracht 2. Max 15 punten: 8 pt = 5,8 / 10 pt = 7,0 / 12 pt = 8,2 / 15 pt = 10.
> Wat je maakt (3 dingen):
> 1. **Onderzoeksverslag** (bijlage 2.1) met communicatiematrix
> 2. **E-mail** aan de klant (bijlage 2.2), volgens de AVG, met de juiste bijlage
> 3. **Logboek** (bijlage 2.4) met datum, tijd, acties én **screenshots**
>
> Je krijgt een lijstje **zwakheden** (bijlage 2.3). In de proeftoets:
> ```
> Gast toegang staat open op alle mappen
> Eigenaarschap van de mappen is verkeerd geconfigureerd
> ```
> Volgens de nakijkonderlegger zijn dat **3 kwetsbaarheden** die je moet oplossen én testen:
> 1. Het **gastaccount is actief**
> 2. Het **gastaccount heeft rechten op mappen**
> 3. De rechten op de map **Management** zijn te ruim

---

## 0. Stappenplan op het examen

| Stap | Wat | Lever op |
|---|---|---|
| 1 | Logboek openen en **meteen** datum invullen. Bij elke actie: tijd + wat + screenshot. | Logboek |
| 2 | Zwakheden lezen. Elke zwakheid onderzoeken: bestaat hij echt? Hoe groot? (screenshot **vóór**) | Analyse |
| 3 | Elke kwetsbaarheid **categoriseren** met de communicatiematrix (type + impact + wie informeren) | Verslag |
| 4 | Oplossen (configureren) | Verslag + logboek |
| 5 | **Testen** dat het werkt (screenshot **ná**: "access denied") | Verslag + logboek |
| 6 | Logs bekijken: is er al misbruik geweest? (bv. gast heeft ingelogd) | Verslag + mail |
| 7 | Onderzoeksverslag afmaken | Bijlage 2.1 |
| 8 | E-mail schrijven, **onderzoeksverslag als bijlage**, juiste Aan/CC/Van | Bijlage 2.2 |

**3 x oplossen + 3 x testen = punten voor "Technische oplossingen" (Volledig = 3 verbeteringen, correct geconfigureerd én getest).**

---

## 1. Linux + Samba: oplossen en testen

> Altijd eerst een backup van de config:
> ```bash
> sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
> ```

### 1.1 Onderzoeken (vóór screenshot)

```bash
sudo pdbedit -L -v gast                  # Account Flags zonder [D] = actief
sudo passwd -S gast                      # P = actief, L = geblokkeerd
grep -n -E 'guest|valid users|map to guest' /etc/samba/smb.conf
ls -la /home/fileserver                  # management 777 = te ruim, eigenaar/groep fout?
smbclient -L //localhost -N              # lukt dit zonder wachtwoord = gasttoegang open
smbclient //localhost/management -U gast # komt gast erin?
last gast ; sudo grep -i gast /var/log/auth.log ; ls /var/log/samba/
```

### 1.2 Kwetsbaarheid 1: gastaccount is actief → uitschakelen

```bash
sudo smbpasswd -d gast                   # Samba-account uitschakelen (disable)
sudo usermod -L gast                     # Linux-wachtwoord blokkeren (lock)
sudo usermod -s /usr/sbin/nologin gast   # kan niet meer inloggen op shell
sudo usermod -e 1 gast                   # account laten verlopen (optioneel)
# OF helemaal verwijderen:
sudo pdbedit -x gast                     # uit Samba
sudo userdel gast                        # uit Linux (-r = ook homemap weg)
```
**Testen:**
```bash
sudo pdbedit -L -v gast | grep Flags     # moet [DU ] tonen (D = disabled)
sudo passwd -S gast                      # moet L tonen
smbclient //localhost/iedereen -U gast   # moet NT_STATUS_ACCOUNT_DISABLED / LOGON_FAILURE geven
```

### 1.3 Kwetsbaarheid 2: gast heeft rechten op mappen → weghalen in smb.conf

```bash
sudo nano /etc/samba/smb.conf
```
In `[global]`:
```ini
map to guest = never          ; was: bad user
restrict anonymous = 2        ; geen anonieme lijst van shares
```
In **elke** share:
```ini
guest ok = no                 ; was: yes
; public = no                 ; (public is hetzelfde als guest ok)
valid users = @management     ; "gast" weghalen uit valid users
```
Opslaan (`Ctrl+O`, `Enter`, `Ctrl+X`), dan:
```bash
sudo testparm -s                         # config controleren op fouten
sudo systemctl restart smbd nmbd         # Samba herstarten (anders werkt het niet!)
```
**Testen:**
```bash
smbclient -L //localhost -N              # moet NT_STATUS_ACCESS_DENIED geven
smbclient //localhost/management -N      # moet geweigerd worden
smbclient //localhost/management -U hannekecool   # directie moet er WEL nog in kunnen
```

### 1.4 Kwetsbaarheid 3: rechten/eigenaarschap van Management te ruim → 770 + juiste groep

```bash
ls -ld /home/fileserver/management       # vóór: drwxrwxrwx root root
sudo chown root:management /home/fileserver/management   # groep = management
sudo chmod 770 /home/fileserver/management               # alleen eigenaar + groep
# alle afdelingsmappen in één keer (als die ook fout staan):
for a in administratie magazijn management sales; do
  sudo chown -R root:$a /home/fileserver/$a
  sudo chmod 2770 /home/fileserver/$a      # 2 = setgid: nieuwe bestanden krijgen de groep
done
```
In smb.conf voor de share (extra zekerheid):
```ini
[management]
   path = /home/fileserver/management
   valid users = @management
   guest ok = no
   writable = yes
   create mask = 0660
   directory mask = 0770
```
**Testen:**
```bash
ls -ld /home/fileserver/management                       # na: drwxrwx--- root management
sudo -u annie ls /home/fileserver/management             # Permission denied = goed
smbclient //localhost/management -U anniestryker         # NT_STATUS_ACCESS_DENIED = goed
smbclient //localhost/management -U hannekecool          # lukt = goed
```

### 1.5 Extra (als het in de zwakheden staat of tijd over is)

```bash
sudo gpasswd -d younesbulut administratie      # gebruiker uit groep
sudo usermod -aG sales younesbulut             # gebruiker aan groep toevoegen (-a NIET vergeten!)
sudo usermod -c "Salma Aslan" salmademirci     # volledige naam aanpassen
sudo usermod -L bramburg ; sudo smbpasswd -d bramburg   # oud-medewerker uitschakelen
sudo userdel -r bramburg ; sudo pdbedit -x bramburg     # of verwijderen
# losse gebruiker uit share: in smb.conf "anniestryker" weghalen bij valid users
```

---

## 2. Windows Server: oplossen en testen

### 2.1 Kwetsbaarheid 1: Guest is actief → uitschakelen

GUI: `lusrmgr.msc` (of Computer Management → Local Users and Groups) → **Users** → dubbelklik **Guest** → ☑ **Account is disabled** → OK.

```cmd
net user Guest /active:no
net user Guest                    :: controle: "Account active   No"
```
```powershell
Disable-LocalUser -Name "Guest"
Get-LocalUser Guest | Select Name, Enabled     # Enabled = False
```
Domein: `dsa.msc` → Users → Guest → rechtermuis → **Disable Account**.

### 2.2 Kwetsbaarheid 2: Guest/Everyone heeft rechten op mappen → weghalen

GUI (per map!): `C:\Fileserver\Management` → **Properties** → **Sharing** → **Advanced Sharing** → **Permissions** → selecteer *Guest* (of *Everyone*) → **Remove** → OK.
Daarna ook tabblad **Security** → **Edit** → Guest/Everyone **Remove**.

```powershell
Get-SmbShareAccess -Name "Management"                                    # vóór
Revoke-SmbShareAccess -Name "Management" -AccountName "Guest" -Force
Revoke-SmbShareAccess -Name "Management" -AccountName "Everyone" -Force  # als die erop staat
Get-SmbShareAccess -Name "Management"                                    # na
icacls C:\Fileserver\Management /remove Guest                            # NTFS
```

### 2.3 Kwetsbaarheid 3: rechten Management te ruim → alleen Management-groep

```powershell
Grant-SmbShareAccess  -Name "Management" -AccountName "Management" -AccessRight Full -Force
Revoke-SmbShareAccess -Name "Management" -AccountName "Everyone" -Force
icacls C:\Fileserver\Management /inheritance:r /grant "Management:(OI)(CI)M" /grant "Administrators:(OI)(CI)F"
```
Rechten aan een andere afdelingsshare toevoegen/verwijderen:
```powershell
Grant-SmbShareAccess  -Name "Sales" -AccountName "Sales" -AccessRight Change -Force
Revoke-SmbShareAccess -Name "Sales" -AccountName "Annie Stryker" -Force
```

### 2.4 Overige fixes

```cmd
net localgroup Administratie younesbulut /delete    :: uit groep
net user bramburg /active:no                        :: uitschakelen
net user bramburg /delete                           :: verwijderen
```
```powershell
Set-LocalUser -Name "salma" -FullName "Salma Aslan"           # volledige naam aanpassen
Remove-LocalGroupMember -Group "Administratie" -Member "younesbulut"
Remove-LocalUser -Name "bramburg"
```
Let op: een **gebruikersnaam** zelf kun je niet netjes "hernoemen" zonder gevolgen (profiel, rechten). Pas de **Full name** aan, of maak een nieuw account en zet de data over. Zet dat zo in je verslag.

### 2.5 Testen in Windows (screenshot = bewijs)

- **Effective Access**: map → Properties → Security → **Advanced** → tabblad **Effective Access** → *Select a user* → Guest → **View effective access** → alles ✘.
- Inloggen als andere gebruiker en `\\SRV001\Management` openen → *"Windows cannot access… You do not have permission"*.
- CMD: `net use \\SRV001\Management /user:Guest ""` → *System error 1331: account disabled*.
- `net user Guest` → *Account active No*.

### 2.6 Is er al misbruik geweest? (Event Viewer)

`eventvwr.msc` → **Windows Logs** → **Security** → **Filter Current Log** → Event ID **4624**, zoek op Account Name *Guest*.
In de proeftoets: Guest is op **8 mei 2023 om 08:21** succesvol ingelogd. Bestandstoegang werd **niet** gelogd → je kunt niet vaststellen wat er gekopieerd is → **mogelijk datalek**. Zet dit in verslag én mail.

Auditing aanzetten (advies): `gpedit.msc` → Computer Configuration → Windows Settings → Security Settings → Advanced Audit Policy → Object Access → **Audit File Share** / **Audit File System** = Success, Failure.

---

## 3. Kwetsbaarheden analyseren en categoriseren (criterium "Security incidenten")

> Volledig = **alle 3** kwetsbaarheden overgenomen, **geanalyseerd** én **juist gecategoriseerd**.
> Categoriseren = per kwetsbaarheid: **soort probleem** + **impact (omvang)** + **wie informeren** (uit de communicatiematrix).

### De communicatiematrix (staat in het sjabloon, neem hem over en gebruik hem!)

| Probleem | Impact | Met wie contact opnemen |
|---|---|---|
| Rechten verkeerd ingesteld | Gegevens van 1 gebruiker | De betreffende gebruiker |
| Rechten verkeerd ingesteld | Gegevens van 1 afdeling | Alle medewerkers binnen de afdeling + de directie |
| Rechten verkeerd ingesteld | Groter dan 1 afdeling | De directie |
| Configuratiefout fileserver | Gegevens van 1 gebruiker | De betreffende gebruiker |
| Configuratiefout fileserver | Gegevens van 1 afdeling | Alle medewerkers binnen de afdeling + de directie |
| Configuratiefout fileserver | Groter dan 1 afdeling | De directie |

### Voorbeeld: analyse + categorie (zet dit als tabel in je verslag)

| # | Kwetsbaarheid | Analyse (wat is er aan de hand, hoe groot) | Categorie (matrix) | Impact (BIV) | Informeren |
|---|---|---|---|---|---|
| 1 | Gastaccount is actief | Iedereen kan zonder wachtwoord inloggen op de fileserver. Uit de Security-log blijkt dat Guest op 08-05-2023 heeft ingelogd. | **Configuratiefout fileserver**, groter dan 1 afdeling | Vertrouwelijkheid: hoog | **Directie** |
| 2 | Gastaccount heeft rechten op mappen | Via gast zijn alle mappen (o.a. Management) te lezen, wijzigen en kopiëren. Mogelijk datalek met persoonsgegevens (webshop-klanten). | **Rechten verkeerd ingesteld**, groter dan 1 afdeling | V + I: hoog | **Directie** (+ AVG-verantwoordelijke) |
| 3 | Rechten/eigenaarschap map Management te ruim (777 / Everyone) | Alle medewerkers kunnen bij vertrouwelijke directiegegevens. | **Rechten verkeerd ingesteld**, groter dan 1 afdeling | V: hoog | **Directie** |

Woorden die examinatoren willen zien: *datalek*, *vertrouwelijkheid*, *integriteit*, *omvang*, *ongeautoriseerde toegang*, *AVG*, *meldplicht*.

---

## 4. Onderzoeksverslag (bijlage 2.1): voorbeeld om in te vullen

```
Opdrachtgever : Hanneke Cool, Handelsbedrijf Conley
Auteur        : Jafeth Worm, IT expert SecureIT
Datum         : 05-11-2026
Versie        : 1.0
```

**Overzicht van de te onderzoeken onderdelen**
> In dit onderzoeksverslag wordt beschreven welke onderdelen zijn onderzocht en welke resultaten dit heeft opgeleverd.

**1.1 Aanleiding onderzoek**
> Handelsbedrijf Conley heeft SecureIT gevraagd de fileserver SRV001 te onderzoeken naar aanleiding van gemelde zwakheden: gasttoegang staat open op alle mappen en het eigenaarschap van de mappen is verkeerd geconfigureerd. De directrice, mevrouw Cool, wil zeker weten dat alleen geautoriseerde medewerkers bij de bedrijfsgegevens kunnen en dat er geen gegevens zijn uitgelekt.

**1.2 Uitgevoerd onderzoek**
> In de Linux-omgeving zijn gecontroleerd:
> - Gebruikers: `/etc/passwd` en `pdbedit -L -v` (is het gastaccount actief?)
> - Groepen: `/etc/group`
> - Rechten: `/etc/samba/smb.conf` (`guest ok`, `valid users`) en `ls -la /home/fileserver` (eigenaar en rechten)
> - Logbestanden: `/var/log/samba/` en `last`
>
> In de Windows-omgeving zijn gecontroleerd:
> - Gebruikers en groepen: Computer Management → Local Users and Groups
> - Rechten: eigenschappen van de mappen (Share Permissions en Security)
> - Logbestanden: Event Viewer → Security (Event ID 4624)

**1.3 Analyse en categorisering**
> *(tabel uit paragraaf 3 hierboven)*

**Conclusie**
> De rechten op de fileserver zijn niet juist geconfigureerd. Het gastaccount was actief en had toegang tot alle mappen, waaronder de map Management met vertrouwelijke directiegegevens. Uit de Security-log blijkt dat het gastaccount op 8 mei 2023 om 08:21 succesvol heeft ingelogd. Omdat bestandstoegang niet werd gelogd, is niet vast te stellen welke gegevens zijn ingezien of gekopieerd. Daarom moet rekening worden gehouden met een mogelijk datalek. Daarnaast waren de rechten en het eigenaarschap van de map Management te ruim ingesteld. Alle drie de kwetsbaarheden raken meer dan één afdeling en zijn volgens de communicatiematrix gemeld aan de directie.

**Aanpassing** (met screenshot vóór én na bij elk punt)
> **Kwetsbaarheid 1: gastaccount actief**
> Het gastaccount is uitgeschakeld met `smbpasswd -d gast` en `usermod -L gast` (Windows: *Account is disabled*). Getest met `smbclient //localhost/iedereen -U gast`: de toegang wordt geweigerd. *(screenshot)*
>
> **Kwetsbaarheid 2: gast heeft rechten op mappen**
> In `smb.conf` is bij alle shares `guest ok = no` gezet, `gast` is verwijderd uit `valid users` en in `[global]` is `map to guest = never` ingesteld. Samba is herstart. (Windows: Guest verwijderd uit de Share Permissions van de map Management.) Getest met `smbclient -L //localhost -N`: toegang geweigerd. *(screenshot)*
>
> **Kwetsbaarheid 3: rechten map Management te ruim**
> De eigenaar is aangepast naar `root:management` en de rechten van 777 naar 770. Getest door als Annie Stryker de map te openen: *Permission denied*. Mevrouw Cool heeft nog wel toegang. *(screenshot)*

**Communicatiematrix**
> *(matrix uit paragraaf 3 overnemen) +* Op basis van deze matrix is de directie (mevrouw Cool) geïnformeerd, omdat alle kwetsbaarheden meer dan één afdeling raken.

---

## 5. E-mail (bijlage 2.2) volgens de AVG

### Waar letten ze op? (AVG + Informeren = samen 6 punten!)

| Punt | Goed |
|---|---|
| **Van** | Jouw **eigen zakelijke** adres (school/SecureIT), niet privé-Gmail |
| **Aan** | De **directie** = opdrachtgever (mevrouw Cool), want impact > 1 afdeling (matrix) |
| **CC** | De **AVG-verantwoordelijke** die in de opdracht staat. Proeftoets: onderlegger noemt **Bjorn Hentges**; het interview noemt **Salma** (afdeling Administratie). **Lees in de examenopdracht wie het is!** Geen andere medewerkers in CC (dataminimalisatie). Meerdere externe ontvangers = **BCC**. |
| **Benoem waar de mail heen gaat** | Schrijf in de mail én in het verslag: *"Deze e-mail is verzonden aan mevrouw Cool (directie) met een kopie aan … (AVG-verantwoordelijke), conform de communicatiematrix."* |
| **Bijlage** | Het **onderzoeksverslag** (bijlage 2.1). Niet het logboek met wachtwoorden of de config. Noem de bijlage in de tekst. |
| **Geen gevoelige data in de mail** | Geen wachtwoorden, geen klantgegevens, geen persoonsgegevens kopiëren in de tekst |
| **Toon** | Zakelijk: "Geachte mevrouw Cool", "Met vriendelijke groet", u-vorm |
| **AVG-meldplicht** | Mogelijk datalek met persoonsgegevens (webshop met particuliere klanten) → binnen **72 uur** melden bij de **Autoriteit Persoonsgegevens**; ook datalek **registreren** |

### Voorbeeldmail

```
Van:        jafeth.worm@student.graafschapcollege.nl   (of: j.worm@secureit.nl)
Aan:        hanneke.cool@conley.nl
CC:         bjorn.hentges@conley.nl   (AVG-verantwoordelijke volgens opdracht)
Onderwerp:  Resultaten security-onderzoek fileserver SRV001 – mogelijk datalek
Bijlage:    Onderzoeksverslag_SRV001_v1.0.pdf
```

> Geachte mevrouw Cool,
>
> Op uw verzoek heb ik de fileserver SRV001 onderzocht naar aanleiding van de gemelde zwakheden. In deze e-mail vat ik mijn bevindingen, de risico's en mijn adviezen samen. Het volledige onderzoeksverslag vindt u in de bijlage.
>
> **Bevindingen**
> - Het gastaccount was actief, waardoor iedereen zonder wachtwoord kon inloggen op de fileserver.
> - Het gastaccount had toegang tot alle mappen, waaronder de map Management.
> - De rechten en het eigenaarschap van de map Management waren te ruim ingesteld, waardoor alle medewerkers toegang hadden.
> - Uit de logbestanden blijkt dat het gastaccount op 8 mei 2023 om 08:21 succesvol heeft ingelogd. Omdat bestandstoegang niet werd gelogd, is niet vast te stellen welke gegevens zijn ingezien of gekopieerd.
>
> **Uitgevoerde maatregelen**
> Het gastaccount is uitgeschakeld, de gasttoegang is van alle mappen verwijderd en de map Management is alleen nog toegankelijk voor de groep Management. Alle aanpassingen zijn getest en werken naar behoren.
>
> **Risico's**
> Via het gastaccount konden onbevoegden vertrouwelijke bedrijfs- en klantgegevens lezen, wijzigen of kopiëren. Omdat Conley een webshop voor particulieren heeft, kunnen hierbij persoonsgegevens betrokken zijn. Er is daarom sprake van een mogelijk datalek in de zin van de AVG.
>
> **Adviezen**
> - Meld het mogelijke datalek binnen 72 uur bij de Autoriteit Persoonsgegevens en registreer het in het interne datalekregister.
> - Ken rechten op mappen alleen toe aan groepen en niet aan individuele medewerkers.
> - Schakel auditing van bestandstoegang in, zodat bij een volgend incident te achterhalen is wie welke gegevens heeft benaderd.
> - Laat gebruikers, groepen en rechten periodiek controleren tegen de actuele personeelslijst.
>
> Omdat de kwetsbaarheden meer dan één afdeling raken, is deze e-mail volgens de communicatiematrix verzonden aan u als directie, met een kopie aan de heer Hentges als verantwoordelijke voor de AVG.
>
> Mocht u vragen hebben, dan hoor ik dat graag.
>
> Met vriendelijke groet,
>
> Jafeth Worm
> IT expert – SecureIT

---

## 6. Logboek (bijlage 2.4)

> Volledig = **datum, tijd, werkzaamheden, acties** ingevuld **én screenshots** van de verbeteringen.
> Kort maar duidelijk: wat was het probleem, wat heb je gedaan. Vul het **tijdens** het werk in, niet achteraf.

**Datum: 05-11-2026**

| Tijd | Omschrijving kwetsbaarheid | Actie uitgevoerd | Screenshots |
|---|---|---|---|
| 09:05 | Start onderzoek, backup gemaakt van smb.conf | `cp smb.conf smb.conf.bak` | – |
| 09:10 | Gastaccount is actief (pdbedit-flags zonder D / Guest "Account active Yes") | Onderzocht, screenshot vóór | vóór |
| 09:15 | Gastaccount is actief | Gast uitgeschakeld (`smbpasswd -d gast`, `usermod -L gast` / *Account is disabled*). Getest: inloggen als gast geweigerd | vóór + ná |
| 09:25 | Gast heeft rechten op alle mappen (`guest ok = yes`, gast in `valid users`) | `guest ok = no`, gast uit `valid users`, `map to guest = never`, `testparm`, Samba herstart. Getest met `smbclient -N`: geweigerd | vóór + ná |
| 09:40 | Map Management heeft rechten 777 / eigenaar root:root | `chown root:management`, `chmod 770`. Getest: Annie *Permission denied*, Hanneke heeft toegang | vóór + ná |
| 09:50 | Mogelijk misbruik door gast | Logbestanden gecontroleerd: gast ingelogd 08-05-2023 08:21. Alleen onderzocht, geen aanpassing | log |
| 10:00 | – | Onderzoeksverslag geschreven | – |
| 10:30 | – | E-mail verstuurd aan directie, CC AVG-verantwoordelijke, bijlage onderzoeksverslag | – |

**Screenshot-tips:** laat het **commando én de uitkomst** zien. Maak er één **vóór** en één **ná** de fix. Bij Windows: het venster met het vinkje/de rechten **en** de testmelding "access denied".

---

## 7. Zelf nakijken (beoordelingsformulier W2)

| Taak | Criterium | Volledig (3 pt) als... | ✔ |
|---|---|---|---|
| T1 | Security incidenten | Alle **3** kwetsbaarheden overgenomen, **geanalyseerd** en **juist gecategoriseerd** (matrix) | ☐ |
| T2 | Technische oplossingen | **3** verbeteringen doorgevoerd, correct geconfigureerd **en getest** (screenshot test) | ☐ |
| T2 | AVG | **Aan/CC** goed, **Van**-adres goed, en **benoemd waar** de mail heen gaat | ☐ |
| T3 | Logboek | Datum, tijd, werkzaamheden, acties **+ screenshots** | ☐ |
| T4 | Informeren | **Zakelijke** mail, **juiste bijlage** (onderzoeksverslag), volgens AVG verstuurd | ☐ |

## 8. Veelgemaakte fouten

| Fout | Zo doe je het goed |
|---|---|
| Samba niet herstart na wijziging smb.conf | `sudo systemctl restart smbd` en `testparm -s` |
| Fix gedaan maar **niet getest** | Altijd proberen in te loggen als gast / andere gebruiker → screenshot van de weigering |
| Alleen Share-rechten in Windows aangepast | Ook tabblad **Security** (NTFS) controleren |
| `usermod -G` zonder `-a` | Gooit de gebruiker uit al zijn andere groepen! Gebruik `usermod -aG` |
| Directie zelf de map Management afgepakt | Na de fix testen dat Management (Hanneke) er **wel** nog in kan |
| Mail zonder bijlage of met verkeerde bijlage | Onderzoeksverslag bijvoegen en noemen in de tekst |
| Iedereen in CC | Alleen directie + AVG-verantwoordelijke (dataminimalisatie) |
| Niet benoemd waar de mail heen gaat | Zin over communicatiematrix in mail en verslag |
| Logboek achteraf invullen zonder tijden | Tijdens het werk elke actie met tijd noteren |
