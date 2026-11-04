# 04 · Denk-eraan checklist (wat je NIET letterlijk gevraagd wordt, maar wel punten kost)

> De opdracht zegt vaak alleen "controleer de security" of "los de zwakheden op". De punten zitten in de details hieronder.
> Print/lees dit vlak voor je inlevert en vink af.

## Tijdens het zoeken (W1 én W2)

- [ ] Heb ik **beide servers** bekeken (Linux én Windows), als ik er twee heb gekregen?
- [ ] Heb ik **`smb.conf`** gelezen, ook de regels met `#` (commentaar)?
- [ ] Heb ik **`pdbedit -L -v`** gedaan? Een Linux-account is niet hetzelfde als een Samba-account.
- [ ] Heb ik bij Windows **Share Permissions én tabblad Security** bekeken, voor **alle 5** mappen?
- [ ] Heb ik **Guest/gast** gecontroleerd: actief? rechten?
- [ ] Heb ik **elke naam** op de server vergeleken met de personeelslijst (ook de spelling van de achternaam)?
- [ ] Heb ik gekeken of iemand in **twee afdelingsgroepen** zit?
- [ ] Heb ik gekeken of er **losse gebruikers** in de rechten staan (zonder `@` in Linux, persoonsicoon in Windows)?
- [ ] Heb ik gekeken of een **afdeling op een andere afdelingsmap** staat, of juist **ontbreekt** op de eigen map?
- [ ] Heb ik **`ls -la`** gedaan (777? eigenaar root:root?)
- [ ] Heb ik de **logs** bekeken: heeft gast of een oud-medewerker ooit ingelogd? (Event Viewer 4624 / `last`)
- [ ] Heb ik van **alles een screenshot** met een zin eronder?

## Auditrapport (W1)

- [ ] **Tools + motivatie** (waarom deze tool) in onderdeel 1
- [ ] Tabel gebruikers/groepen + tabel rechten op mappen
- [ ] **Conclusie** in hele zinnen
- [ ] **Minstens 5** kwetsbaarheden, met **impact** en **gesorteerd op prioriteit** (Hoog → Laag)
- [ ] Verbetervoorstel per punt: **wat, waarom, hoe, wanneer** + technisch én beleid
- [ ] Checklist: **Ja/Nee** (geen "grotendeels"); "rechten direct aan medewerkers?" = **Nee** als er één losse gebruiker staat
- [ ] Gericht aan de **directeur** (Hanneke Cool), formele taal, u-vorm
- [ ] Management overal bij = **conform beleid**, niet als fout tellen

## Oplossen (W2)

- [ ] **Backup** van `smb.conf` gemaakt vóór wijzigen
- [ ] **Vóór-screenshot** van elk probleem
- [ ] Alle **3** kwetsbaarheden opgelost: gast actief / gast rechten / Management te ruim
- [ ] `testparm -s` + **`systemctl restart smbd`** na elke smb.conf-wijziging
- [ ] **Elke fix getest** (gast geweigerd, Annie geweigerd) → **ná-screenshot**
- [ ] Gecontroleerd dat **Hanneke/Management er nog wél in kan** (niet te streng gemaakt)
- [ ] Gecontroleerd dat **afdelingen nog bij hun eigen map** kunnen
- [ ] Bij Windows: **Effective Access** screenshot

## Onderzoeksverslag (W2)

- [ ] Aanleiding, uitgevoerd onderzoek, conclusie
- [ ] Per kwetsbaarheid: **analyse** + **categorie** (Rechten verkeerd / Configuratiefout) + **impact** (1 gebruiker / 1 afdeling / > 1 afdeling) + **wie informeren**
- [ ] Aanpassingen met **screenshots**
- [ ] **Communicatiematrix** erin + zin "daarom is de directie geïnformeerd"
- [ ] Gelogde login van gast genoemd → **mogelijk datalek**

## E-mail (W2) — 6 punten!

- [ ] **Van**: eigen zakelijke/schooladres (geen privé)
- [ ] **Aan**: directie (Hanneke Cool)
- [ ] **CC**: AVG-verantwoordelijke (kijk in de opdracht: Bjorn Hentges of Salma?)
- [ ] Niet iedereen in CC; meerdere externe ontvangers = BCC
- [ ] **Zin waarin staat waar de mail heen gaat en waarom** (communicatiematrix)
- [ ] **Bijlage = onderzoeksverslag** + genoemd in de tekst
- [ ] Zakelijk: "Geachte mevrouw Cool" … "Met vriendelijke groet"
- [ ] Bevindingen, maatregelen, risico's, adviezen
- [ ] **AVG**: mogelijk datalek, melden bij **Autoriteit Persoonsgegevens binnen 72 uur**, datalekregister
- [ ] **Geen** wachtwoorden of persoonsgegevens in de mailtekst

## Logboek (W2)

- [ ] **Datum** ingevuld
- [ ] Elke regel een **tijd**
- [ ] Omschrijving kwetsbaarheid + **actie** (wat heb ik gedaan)
- [ ] **Screenshot** bij elke verbetering
- [ ] Ook "alleen onderzocht, geen aanpassing" regels (logs bekijken)

## Inleveren

- [ ] Alle bestanden met juiste naam (bv. `Auditrapport_Jafeth_Worm.docx`)
- [ ] Algemene gegevens (opdrachtgever, auteur, datum, versie) ingevuld
- [ ] Spelling gecontroleerd (F7 in Word)
