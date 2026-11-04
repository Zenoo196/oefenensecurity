# 03 · Kant-en-klare teksten (overnemen en aanpassen)

> Geschreven voor de casus van de proeftoets (Conley, Hanneke Cool). Vervang wat tussen `<...>` staat.
> **Controleer altijd** of wat je opschrijft ook echt zo op jouw server stond. Schrijf alleen op wat je zelf hebt gezien (met screenshot).
> Gebruik je alleen Linux of alleen Windows? Haal de andere kant uit de tekst.

1. [Auditrapport (werkproces 1)](#1-auditrapport-werkproces-1)
2. [Onderzoeksverslag (werkproces 2)](#2-onderzoeksverslag-werkproces-2)
3. [E-mail (werkproces 2)](#3-e-mail-werkproces-2)
4. [Logboek (werkproces 2)](#4-logboek-werkproces-2)
5. [Losse zinnen die altijd goed vallen](#5-losse-zinnen-die-altijd-goed-vallen)

---

## 1. Auditrapport (werkproces 1)

```
Opdrachtgever : Hanneke Cool namens Handelsbedrijf Conley
Auteur        : <Jafeth Worm>, IT expert SecureIT
Datum         : <05-11-2026>
Versie        : 1.0
```

### 1. Overzicht van de te testen onderdelen

In dit auditrapport wordt beschreven welke onderdelen zijn getest en welke resultaten dit heeft opgeleverd. Op verzoek van mevrouw Cool is gecontroleerd of alle gebruikers correct zijn ingesteld en geautoriseerd. Daarvoor is de personeelslijst uit het interview, opgesteld door Tjeerd Nickson, vergeleken met de gebruikers, groepen en rechten op fileserver SRV001.

**Gebruikte tools en motivatie**

| Tool | Omgeving | Waarom gekozen |
|---|---|---|
| `/etc/passwd` en `pdbedit -L -v` | Linux | Tonen alle gebruikersaccounts en of het Samba-account actief is. Hiermee is vergeleken of iedere account bij een medewerker op de personeelslijst hoort. |
| `/etc/group` en `id` | Linux | Tonen de leden van iedere afdelingsgroep. Hiermee is gecontroleerd of medewerkers in de juiste afdeling zitten. |
| `/etc/samba/smb.conf` en `testparm -s` | Linux | Tonen per gedeelde map welke groepen en gebruikers toegang hebben (`valid users`) en of gasttoegang aan staat. |
| `ls -la` | Linux | Toont eigenaar, groep en rechten van de mappen op de schijf. |
| `smbclient` | Linux | Praktijktest: controleren of een account daadwerkelijk toegang krijgt tot een map. |
| Computer Management → Local Users and Groups | Windows | Toont alle gebruikers, of een account is uitgeschakeld en de leden van iedere groep. |
| Eigenschappen map → Sharing → Advanced Sharing → Permissions, en tabblad Security | Windows | Tonen de share- en NTFS-rechten per gedeelde map. |
| PowerShell (`Get-LocalUser`, `Get-SmbShareAccess`) | Windows | Geeft in één overzicht alle gebruikers en share-rechten, zodat niets over het hoofd wordt gezien. |

Deze ingebouwde tools zijn gekozen omdat ze de actuele configuratie direct en betrouwbaar tonen, zonder dat er extra software op de server van de klant geïnstalleerd hoeft te worden.

#### 1.1 Gebruikers en groepen

Hieronder staat een samenvatting van de Windows-omgeving (Local Users and Groups) en de Linux-omgeving (`/etc/passwd` en `/etc/group`). Afwijkingen ten opzichte van de personeelslijst zijn **vetgedrukt**.

| Medewerker | Afdeling | Opmerking |
|---|---|---|
| Tjeerd Nickson | Administratie | – |
| Salma Aslan | Administratie | **Achternaam in het systeem is Demirci** |
| Bjorn Hentges | Magazijn | – |
| Ayoub Ahmadi | Magazijn | **Achternaam in het systeem is Özcan** |
| Hanneke Cool | Management | – |
| Younes Bulut | **Sales, Administratie** | **Is ook lid van de groep Administratie** |
| Annie Stryker | Sales | – |
| **Bram Burg** | **Sales** | **Staat niet op de personeelslijst** |
| **Gast** | – | **Account is actief en heeft rechten op mappen** |

*<screenshot /etc/passwd>* Overzicht van de gebruikersaccounts op de Linux-server. Naast de zeven medewerkers bestaan ook de accounts bramburg en gast.
*<screenshot /etc/group>* Overzicht van de groepen. younesbulut is lid van administratie en sales; bramburg is lid van sales en iedereen.
*<screenshot Local Users and Groups>* Overzicht van de gebruikers op de Windows-server. Het account Guest is actief.

#### 1.2 Rechten op mappen

Deze controle is uitgevoerd in het Samba-configuratiebestand (`smb.conf`) en in de share- en NTFS-rechten van de mappen in Windows. Rechten die afwijken van het beleid zijn **vetgedrukt**.

| Map | Toegang toegekend aan | Opmerking |
|---|---|---|
| Administratie | Administratie, Management, **Sales**, **Annie Stryker** | Groep Sales en losse gebruiker horen hier niet |
| Iedereen | Iedereen, **Annie Stryker** | Rechten direct aan een gebruiker |
| Magazijn | Magazijn, Management | Conform beleid |
| Management | Management, **Gast** | Gast heeft toegang tot directiegegevens; in Linux rechten 777 |
| Sales | Sales, Management, **Annie Stryker**, **Administratie** (Linux) | Losse gebruiker; groep Administratie toegevoegd "op verzoek van Salma" zonder akkoord directie |

Management heeft toegang tot alle afdelingsmappen. Dit is conform het beleid van de directie.

*<screenshots smb.conf, ls -la /home/fileserver, Share Permissions per map>*

### 2. Conclusie

De rechten in deze serveromgeving zijn niet juist geconfigureerd. Er zijn gebruikers met een andere achternaam dan op de personeelslijst en er is een actief account van een persoon die niet op de personeelslijst voorkomt (Bram Burg). Eén medewerker is lid van een afdelingsgroep waar hij niet werkt. Het gastaccount is actief en heeft toegang tot de map Management, waardoor vertrouwelijke directiegegevens zonder wachtwoord bereikbaar zijn. Daarnaast zijn op meerdere mappen rechten direct aan een medewerker toegekend in plaats van aan een groep en hebben afdelingen toegang tot elkaars mappen. Wij adviseren de gehele server na te lopen op shares, rechten en gebruikers en de onderstaande punten in volgorde van prioriteit op te lossen.

### 3. Kwetsbaarheden en bedreigingen (gesorteerd op prioriteit)

| Prio | Kwetsbaarheid | Bedreiging en impact | Risico |
|---|---|---|---|
| 1 | Het gastaccount is actief en heeft toegang tot de map Management (Linux: map Management heeft rechten 777) | Iedereen kan zonder wachtwoord vertrouwelijke directiegegevens lezen, wijzigen, verwijderen of kopiëren. Dit kan leiden tot een datalek dat onder de AVG valt. Het raakt het hele bedrijf. | Hoog |
| 2 | Het account van Bram Burg is actief, terwijl hij niet op de personeelslijst staat | Een persoon die niet (meer) voor Conley werkt, heeft toegang tot de gegevens van Sales en de map Iedereen. Klant- en verkoopgegevens kunnen worden meegenomen. | Hoog |
| 3 | De groep Sales heeft toegang tot de map Administratie, en de groep Administratie tot de map Sales | Medewerkers kunnen financiële en administratieve gegevens van een andere afdeling inzien. Dit raakt meer dan één afdeling. | Hoog |
| 4 | Younes Bulut is lid van de groep Administratie, terwijl hij bij Sales werkt | Eén medewerker heeft onnodig toegang tot administratieve gegevens. Dit is in strijd met het principe dat medewerkers alleen toegang hebben tot wat zij nodig hebben. | Middel |
| 5 | Annie Stryker heeft rechten direct als gebruiker op de mappen Administratie, Iedereen en Sales | Annie heeft toegang tot de map Administratie. Rechten op personen maken het beheer onoverzichtelijk; bij een functiewissel of vertrek blijven deze rechten eenvoudig bestaan. | Middel |
| 6 | De achternamen van Salma (Demirci) en Ayoub (Özcan) wijken af van de personeelslijst | Het is niet zeker dat deze accounts bij de juiste personen horen. Bij een verkeerde koppeling kan iemand toegang hebben tot gegevens die niet voor hem of haar bedoeld zijn. | Laag |

### 4. Verbetervoorstel

**1. Gastaccount uitschakelen en gasttoegang verwijderen (direct)**
Het gastaccount geeft iedereen zonder wachtwoord toegang tot vertrouwelijke gegevens. Wij adviseren het account uit te schakelen (Windows: Guest → Account is disabled; Linux: `smbpasswd -d gast` en `usermod -L gast`), het te verwijderen uit de rechten van de map Management en in Samba `guest ok = no` en `map to guest = never` in te stellen. De map Management krijgt de rechten 770 met groep management. Dit kost ongeveer 15 minuten en heeft geen gevolgen voor de medewerkers.

**2. Account van Bram Burg uitschakelen (direct)**
Wij adviseren bij mevrouw Cool te controleren of Bram Burg uit dienst is, zijn account direct uit te schakelen en het na het veiligstellen van zijn bestanden te verwijderen.

**3. Afdelingsrechten herstellen (deze week)**
Verwijder de groep Sales van de map Administratie. De toegang van de groep Administratie tot de map Sales is toegevoegd op verzoek van Salma; wij adviseren dit alleen te behouden na schriftelijk akkoord van de directie. Zo hebben medewerkers alleen toegang tot de gegevens van hun eigen afdeling.

**4. Groepslidmaatschap corrigeren (deze week)**
Verwijder Younes Bulut uit de groep Administratie (Linux: `gpasswd -d younesbulut administratie`; Windows: Groups → Administratie → Remove).

**5. Rechten alleen via groepen toekennen (deze week)**
Verwijder Annie Stryker als losse gebruiker uit de rechten van Administratie, Iedereen en Sales. Via de groepen Sales en Iedereen heeft zij al de juiste toegang. Rechten op groepen zijn overzichtelijk en verdwijnen automatisch bij een functiewissel.

**6. Namen verifiëren**
Laat Tjeerd Nickson controleren of de accounts van Salma en Ayoub bij de juiste personen horen en pas de volledige naam aan. Een gebruikersnaam wijzigen kan gevolgen hebben voor de rechten; maak daarom bij voorkeur een nieuw account aan en zet de gegevens over.

**7. Beleid (structureel)**
- Voer ieder kwartaal een controle uit van alle accounts en rechten tegen de actuele personeelslijst.
- Leg een in- en uitdienstprocedure vast, zodat een account op de laatste werkdag wordt uitgeschakeld.
- Laat wijzigingen in rechten alleen doorvoeren na akkoord van de directie en registreer deze.
- Schakel logging (auditing) van bestandstoegang in, zodat bij een incident te achterhalen is wie welke gegevens heeft benaderd.

### 5. Checklist testen

| Gebruikers en groepen | |
|---|---|
| Zijn alle medewerkers opgenomen als gebruiker? | Ja |
| Zijn alle voormalige medewerkers verwijderd? | Nee (Bram Burg, gast) |
| Zijn alle gebruikers opgenomen in de juiste groepen? | Nee (Younes Bulut ook in Administratie) |
| **Rechten op mappen** | |
| Zijn rechten op mappen verleend aan de juiste groepen? | Ja (iedere afdeling heeft toegang tot de eigen map) |
| Hebben de juiste groepen toegang tot alle mappen? | Nee (Sales op Administratie, gast op Management) |
| Zijn alle rechten verleend aan groepen, dus niet direct aan medewerkers? | Nee (Annie Stryker) |

---

## 2. Onderzoeksverslag (werkproces 2)

```
Opdrachtgever : Hanneke Cool, Handelsbedrijf Conley
Auteur        : <Jafeth Worm>, IT expert SecureIT
Datum         : <05-11-2026>
Versie        : 1.0
```

### Overzicht van de te onderzoeken onderdelen
In dit onderzoeksverslag wordt beschreven welke onderdelen zijn onderzocht en welke resultaten dit heeft opgeleverd.

### 1.1 Aanleiding onderzoek
Handelsbedrijf Conley heeft SecureIT gevraagd fileserver SRV001 te onderzoeken naar aanleiding van twee gemelde zwakheden: de gasttoegang staat open op alle mappen en het eigenaarschap van de mappen is verkeerd geconfigureerd. De directrice, mevrouw Cool, wil zeker weten dat alleen geautoriseerde medewerkers toegang hebben tot de bedrijfsgegevens en of er mogelijk gegevens zijn uitgelekt.

### 1.2 Uitgevoerd onderzoek
In de Linux-omgeving:
- Gebruikers: `/etc/passwd` en `pdbedit -L -v` (is het gastaccount actief?)
- Groepen: `/etc/group`
- Rechten: `/etc/samba/smb.conf` (`guest ok`, `valid users`, `map to guest`) en `ls -la /home/fileserver` (eigenaar en rechten)
- Logbestanden: `last`, `/var/log/auth.log` en `/var/log/samba/`

In de Windows-omgeving:
- Gebruikers en groepen: Computer Management → Local Users and Groups
- Rechten: eigenschappen van de mappen (Share Permissions en tabblad Security)
- Logbestanden: Event Viewer → Windows Logs → Security (Event ID 4624)

### 1.3 Analyse en categorisering van de kwetsbaarheden

| # | Kwetsbaarheid | Analyse | Categorie (communicatiematrix) | Impact | Informeren |
|---|---|---|---|---|---|
| 1 | Het gastaccount is actief | Iedereen kan zonder wachtwoord inloggen op de fileserver. Volgens de Security-log heeft het gastaccount op 8 mei 2023 om 08:21 succesvol ingelogd. | Configuratiefout fileserver, groter dan 1 afdeling | Hoog (vertrouwelijkheid) | De directie |
| 2 | Het gastaccount heeft rechten op mappen | Via het gastaccount zijn alle mappen te benaderen, waaronder Management. Gegevens kunnen zijn gelezen, gewijzigd of gekopieerd. Omdat Conley een webshop voor particulieren heeft, kunnen persoonsgegevens betrokken zijn: mogelijk datalek (AVG). | Rechten verkeerd ingesteld, groter dan 1 afdeling | Hoog (vertrouwelijkheid en integriteit) | De directie + AVG-verantwoordelijke |
| 3 | De rechten en het eigenaarschap van de map Management zijn te ruim | De map had rechten 777 met eigenaar root:root. Iedere gebruiker op de server kon de directiegegevens lezen en wijzigen. | Rechten verkeerd ingesteld, groter dan 1 afdeling | Hoog (vertrouwelijkheid) | De directie |

### Conclusie
De rechten op fileserver SRV001 zijn niet juist geconfigureerd. Het gastaccount was actief en had toegang tot alle mappen, waaronder de map Management met vertrouwelijke directiegegevens. Volgens de Security-log heeft het gastaccount op 8 mei 2023 om 08:21 succesvol ingelogd. Bestandstoegang werd niet gelogd, waardoor niet vast te stellen is welke gegevens zijn ingezien of gekopieerd. Er moet daarom rekening worden gehouden met een mogelijk datalek. Daarnaast waren de rechten en het eigenaarschap van de map Management te ruim ingesteld. Alle drie de kwetsbaarheden raken meer dan één afdeling en zijn volgens de communicatiematrix gemeld aan de directie. De kwetsbaarheden zijn opgelost en getest; zie hieronder.

### Aanpassing

**Kwetsbaarheid 1: het gastaccount is actief**
- Linux: het account is uitgeschakeld met `smbpasswd -d gast` en `usermod -L gast`. Test: `smbclient //localhost/iedereen -U gast` geeft `NT_STATUS_ACCOUNT_DISABLED`. *<screenshot vóór + ná>*
- Windows: bij Guest is *Account is disabled* aangevinkt. Test: `net user Guest` toont *Account active No*. *<screenshot vóór + ná>*

**Kwetsbaarheid 2: het gastaccount heeft rechten op mappen**
- Linux: in `smb.conf` is bij alle shares `guest ok = no` ingesteld, gast is verwijderd uit `valid users` van Management en in `[global]` is `map to guest = never` ingesteld. Samba is herstart. Test: `smbclient -L //localhost -N` wordt geweigerd. *<screenshot>*
- Windows: Guest is verwijderd uit de Share Permissions en het tabblad Security van de map Management. Test: Effective Access voor Guest geeft geen rechten. *<screenshot>*

**Kwetsbaarheid 3: de rechten op de map Management zijn te ruim**
- Linux: eigenaar aangepast met `chown -R root:management` en rechten van 777 naar 2770 met `chmod 2770`. Test: Annie Stryker krijgt *Permission denied*; Hanneke Cool heeft nog wel toegang. *<screenshot>*
- Windows: alleen de groep Management (Modify), Administrators en SYSTEM hebben nog rechten; eigenaar is Administrators. *<screenshot>*

**Overige aanpassingen** *(alleen opnemen als je ze echt hebt gedaan)*
- Het account van Bram Burg is uitgeschakeld.
- Younes Bulut is verwijderd uit de groep Administratie.
- Annie Stryker is als losse gebruiker verwijderd uit de rechten van Administratie, Iedereen en Sales.
- De volledige naam van Salma en Ayoub is aangepast naar de naam op de personeelslijst. Gebruikersnamen zijn niet aangepast, omdat daarvoor een nieuw account moet worden aangemaakt en de gegevens moeten worden overgezet.

### Communicatiematrix

| Probleem | Impact | Met wie contact opnemen |
|---|---|---|
| Rechten verkeerd ingesteld | Gegevens van 1 gebruiker | De betreffende gebruiker |
| Rechten verkeerd ingesteld | Gegevens van 1 afdeling | Alle medewerkers binnen de afdeling + de directie |
| Rechten verkeerd ingesteld | Groter dan 1 afdeling | **De directie** ← van toepassing |
| Configuratiefout fileserver | Gegevens van 1 gebruiker | De betreffende gebruiker |
| Configuratiefout fileserver | Gegevens van 1 afdeling | Alle medewerkers binnen de afdeling + de directie |
| Configuratiefout fileserver | Groter dan 1 afdeling | **De directie** ← van toepassing |

Alle kwetsbaarheden raken meer dan één afdeling. Daarom is de directie, mevrouw Cool, per e-mail geïnformeerd. Omdat er sprake is van een mogelijk datalek, is de e-mail in kopie verstuurd aan <Bjorn Hentges / Salma Aslan>, verantwoordelijk voor de naleving van de AVG.

---

## 3. E-mail (werkproces 2)

```
Van:        <jouw schooladres of j.worm@secureit.nl>
Aan:        <adres van Hanneke Cool uit de opdracht>
CC:         <adres AVG-verantwoordelijke uit de opdracht>
Onderwerp:  Resultaten security-onderzoek fileserver SRV001 – mogelijk datalek
Bijlage:    Onderzoeksverslag SRV001 v1.0
```

Geachte mevrouw Cool,

Op uw verzoek heb ik fileserver SRV001 onderzocht naar aanleiding van de gemelde zwakheden. In deze e-mail vat ik mijn bevindingen, de risico's en mijn adviezen samen. Het volledige onderzoeksverslag vindt u in de bijlage.

**Bevindingen**
- Het gastaccount was actief, waardoor iedereen zonder wachtwoord kon inloggen op de fileserver.
- Het gastaccount had toegang tot de mappen, waaronder de map Management.
- De rechten en het eigenaarschap van de map Management waren te ruim ingesteld, waardoor alle gebruikers toegang hadden.
- Uit de logbestanden blijkt dat het gastaccount op 8 mei 2023 om 08:21 succesvol heeft ingelogd. Omdat bestandstoegang niet werd gelogd, is niet vast te stellen welke gegevens zijn ingezien of gekopieerd.

**Uitgevoerde maatregelen**
Het gastaccount is uitgeschakeld, de gasttoegang is van alle mappen verwijderd en de map Management is alleen nog toegankelijk voor de groep Management. Alle aanpassingen zijn getest en werken naar behoren.

**Risico's**
Via het gastaccount konden onbevoegden vertrouwelijke bedrijfs- en klantgegevens lezen, wijzigen of kopiëren. Omdat Conley een webshop voor particuliere klanten heeft, kunnen hierbij persoonsgegevens betrokken zijn. Er is daarom sprake van een mogelijk datalek in de zin van de AVG.

**Adviezen**
- Meld het mogelijke datalek binnen 72 uur bij de Autoriteit Persoonsgegevens en registreer het in het interne datalekregister.
- Ken rechten op mappen alleen toe aan groepen en niet aan individuele medewerkers.
- Schakel het account van Bram Burg uit; hij staat niet op de personeelslijst.
- Schakel logging van bestandstoegang in, zodat bij een volgend incident te achterhalen is wie welke gegevens heeft benaderd.
- Laat gebruikers, groepen en rechten ieder kwartaal controleren tegen de actuele personeelslijst.

Omdat de kwetsbaarheden meer dan één afdeling raken, is deze e-mail volgens de communicatiematrix verzonden aan u als directie, met een kopie aan <de heer Hentges / mevrouw Aslan> als verantwoordelijke voor de AVG. Andere medewerkers zijn niet in kopie gezet, omdat de inhoud vertrouwelijk is.

Mocht u vragen hebben over het onderzoek of de adviezen, dan hoor ik dat graag.

Met vriendelijke groet,

<Jafeth Worm>
IT expert – SecureIT

---

## 4. Logboek (werkproces 2)

**Datum: <05-11-2026>**

| Tijd | Omschrijving kwetsbaarheid | Actie uitgevoerd | Screenshots |
|---|---|---|---|
| 09:00 | Start onderzoek naar de gemelde zwakheden | Backup gemaakt van smb.conf; controlescript uitgevoerd (situatie vóór) | audit vóór |
| 09:05 | Het gastaccount is actief | Onderzocht met `pdbedit -L -v gast` en `net user Guest`: account is actief | vóór |
| 09:10 | Het gastaccount is actief | Gast uitgeschakeld (`smbpasswd -d gast`, `usermod -L gast`; Windows: Account is disabled). Getest: inloggen als gast wordt geweigerd | ná + test |
| 09:20 | Het gastaccount heeft rechten op mappen | `guest ok = no` bij alle shares, gast uit `valid users` Management, `map to guest = never`; Samba herstart. Windows: Guest verwijderd uit Share Permissions en Security van Management | vóór + ná |
| 09:30 | Het gastaccount heeft rechten op mappen | Getest met `smbclient -L //localhost -N` en Effective Access: toegang geweigerd | test |
| 09:40 | Rechten/eigenaarschap map Management te ruim (777, root:root) | `chown -R root:management`, `chmod 2770`. Windows: alleen Management, Administrators en SYSTEM | vóór + ná |
| 09:45 | Rechten map Management te ruim | Getest: Annie Stryker geweigerd, Hanneke Cool heeft toegang | test |
| 09:55 | Mogelijk misbruik door het gastaccount | Logbestanden onderzocht: gast heeft op 08-05-2023 om 08:21 ingelogd. Alleen onderzocht, geen aanpassing | Event Viewer |
| 10:05 | Overige afwijkingen | Bram Burg uitgeschakeld, Younes Bulut uit groep Administratie | ná |
| 10:15 | – | Controlescript opnieuw uitgevoerd (situatie ná) | audit ná |
| 10:20 | – | Onderzoeksverslag geschreven | – |
| 10:50 | – | E-mail verstuurd aan directie, CC AVG-verantwoordelijke, bijlage onderzoeksverslag | mail |

Voorbeelduitwerking van school ter vergelijking (zo kort mag het ook):

| Tijd | Omschrijving kwetsbaarheid | Actie uitgevoerd |
|---|---|---|
| 14:00 | Guest is geactiveerd | Guest gedeactiveerd |
| 14:10 | Gebruiker die niet meer werkt voor bedrijf, is nog actief | Gebruiker Bram Burg verwijderd |
| 14:20 | Gebruiker (Annie Stryker) is toegevoegd aan een afdelingsshare wat niet hoort | Gebruiker verwijderd en groep toegevoegd |

---

## 5. Losse zinnen die altijd goed vallen

**Impact / risico**
- "Dit vormt een hoog risico op een datalek, omdat vertrouwelijke gegevens zonder authenticatie bereikbaar zijn."
- "Hierdoor wordt de vertrouwelijkheid en integriteit van de bedrijfsgegevens aangetast."
- "Deze kwetsbaarheid raakt meer dan één afdeling en is daarom gemeld aan de directie."
- "Medewerkers hebben toegang tot gegevens die zij voor hun werk niet nodig hebben (need-to-know-principe)."

**Advies**
- "Wij adviseren rechten uitsluitend aan groepen toe te kennen en niet aan individuele medewerkers."
- "Wij adviseren een vaste uitdienstprocedure, waarbij het account op de laatste werkdag wordt uitgeschakeld."
- "Wij adviseren ieder kwartaal een controle uit te voeren van accounts en rechten ten opzichte van de personeelslijst."
- "Wij adviseren logging van bestandstoegang in te schakelen, zodat incidenten achteraf onderzocht kunnen worden."
- "Wijzigingen in rechten worden alleen doorgevoerd na schriftelijk akkoord van de directie."

**AVG**
- "Omdat Conley een webshop voor particuliere klanten heeft, kunnen persoonsgegevens betrokken zijn. Er is daarom sprake van een mogelijk datalek in de zin van de AVG."
- "Een datalek moet binnen 72 uur worden gemeld bij de Autoriteit Persoonsgegevens en worden vastgelegd in het datalekregister."
- "Andere medewerkers zijn niet in kopie gezet, omdat de inhoud vertrouwelijk is (dataminimalisatie)."
