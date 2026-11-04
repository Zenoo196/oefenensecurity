# 01 · Linux + Samba compleet (vinden, oplossen, testen)

> Werk als root: `sudo -i` (of zet `sudo` voor elk commando).
> Paden in de proeftoets: fileserver `/home/fileserver`, config `/etc/samba/smb.conf`.
> Weet je de map niet? `testparm -s | grep path` of `grep path /etc/samba/smb.conf`.

**Inhoud**
1. [Oriënteren: waar ben ik?](#1-oriënteren-waar-ben-ik)
2. [Gebruikers vinden](#2-gebruikers-vinden)
3. [Groepen vinden](#3-groepen-vinden)
4. [Rechten op mappen (bestandssysteem)](#4-rechten-op-mappen-bestandssysteem)
5. [Samba shares (smb.conf)](#5-samba-shares-smbconf)
6. [Oplossen: de 3 kwetsbaarheden uit W2](#6-oplossen-de-3-kwetsbaarheden-uit-w2)
7. [Oplossen: overige afwijkingen uit W1](#7-oplossen-overige-afwijkingen-uit-w1)
8. [Testen + logs](#8-testen--logs)
9. [Controlescript: alles in één keer](#9-controlescript-alles-in-één-keer)
10. [Noodgevallen / foutmeldingen](#10-noodgevallen--foutmeldingen)

---

## 1. Oriënteren: waar ben ik?

```bash
hostname                       # naam server (SRV001?)
whoami                         # wie ben ik
ip a                           # IP-adres (handig voor smbclient vanaf andere pc)
cat /etc/os-release            # welke Linux
systemctl status smbd          # draait Samba? (q = afsluiten)
date                           # tijd voor in je logboek
```

---

## 2. Gebruikers vinden

```bash
cat /etc/passwd                                     # ALLES (ook systeemaccounts)
awk -F: '$3>=1000 && $3<65534 {print $1}' /etc/passwd     # alleen echte gebruikers
awk -F: '$3>=1000 && $3<65534 {print $1" | uid="$3" | naam="$5" | shell="$7}' /etc/passwd
tail -n 15 /etc/passwd                              # nieuwste accounts staan onderaan
getent passwd gast                                  # bestaat gast? (geen output = nee)
id bramburg                                         # uid + alle groepen van iemand
```

Wat zie je in de proeftoets:
```
tjeerdnickson:x:1002:1002::/home/tjeerdnickson:/bin/sh
salmademirci:x:1003:1003::/home/salmademirci:/bin/sh     <- lijst zegt Aslan
bjornhentges:x:1004:1004::/home/bjornhentges:/bin/sh
ayoubozcan:x:1005:1005::/home/ayoubozcan:/bin/sh         <- lijst zegt Ahmadi
hannekecool:x:1006:1006::/home/hannekecool:/bin/sh
younesbulut:x:1007:1007::/home/younesbulut:/bin/sh
anniestryker:x:1008:1008::/home/anniestryker:/bin/sh
bramburg:x:1009:1009::/home/bramburg:/bin/sh             <- niet op de lijst
```
Het naamverschil zit hier in de **gebruikersnaam zelf** (het naamveld is leeg).

### Is een account actief?
```bash
passwd -S gast            # gast P ... = actief (Password set) | L = locked | NP = geen wachtwoord(!)
passwd -S -a              # status van ALLE accounts
chage -l gast             # verloopt account?
lastlog                   # laatste login van iedereen ("**Never logged in**")
last                      # loginhistorie
```

### Samba-gebruikers (dit is wat telt voor de shares!)
```bash
pdbedit -L                       # lijst  (naam:uid:volledige naam)
pdbedit -L -v                    # alles; let op "Account Flags"
pdbedit -L -v gast               # één gebruiker
pdbedit -L -v | grep -E 'Unix username|Account Flags'    # compact overzicht
```
`Account Flags: [U          ]` = actief · `[DU         ]` = **D**isabled · `[NU ...]` = geen wachtwoord nodig (!).

### Gastinstellingen in Samba
```bash
testparm -s 2>/dev/null | grep -i -E 'guest|map to'
#  map to guest = Bad User   -> foute gebruikersnaam wordt automatisch gast  (FOUT)
#  guest account = gast      -> account dat gebruikt wordt als gast
#  guest ok = Yes            -> share open voor gast                         (FOUT)
```

---

## 3. Groepen vinden

```bash
cat /etc/group
grep -E '^(administratie|magazijn|management|sales|iedereen):' /etc/group
getent group sales                  # leden van één groep
groups younesbulut                  # groepen van één persoon
for u in $(awk -F: '$3>=1000 && $3<65534 {print $1}' /etc/passwd); do echo "$u: $(id -nG $u)"; done   # iedereen + groepen
```
Let op: de **primaire** groep van een gebruiker staat niet in het 4e veld van `/etc/group` maar in `/etc/passwd` (4e veld = gid). `id` laat beide zien.

---

## 4. Rechten op mappen (bestandssysteem)

```bash
ls -la /home/fileserver              # rechten + eigenaar + groep per map
ls -ld /home/fileserver/*            # alleen de mappen zelf
stat -c '%A %a %U:%G %n' /home/fileserver/*      # rechten als letters + getal + eigenaar:groep
getfacl -R /home/fileserver 2>/dev/null | less   # extra ACL's (als ls een + toont: drwxrwx---+)
find /home/fileserver -perm -o+w     # alles wat door IEDEREEN beschrijfbaar is (fout!)
find /home/fileserver -perm -o+r -maxdepth 1     # alles wat iedereen kan lezen
```

| Wat zie je | Betekenis | Goed? |
|---|---|---|
| `drwxrwxrwx root root management` | 777, iedereen alles, groep niet management | ❌ te ruim + eigenaarschap fout |
| `drwxrwx--- root sales sales` | 770, alleen groep sales | ✔ |
| `drwxrws--- root sales sales` | 2770 (s = setgid) | ✔✔ |
| `drwxr-xr-x root root` | iedereen lezen | ❌ |

Rekenen: r=4 w=2 x=1 → rwx=7, rw-=6, r-x=5, ---=0. Volgorde: eigenaar / groep / anderen.

---

## 5. Samba shares (smb.conf)

```bash
cat /etc/samba/smb.conf
grep -v -E '^\s*(#|;|$)' /etc/samba/smb.conf        # zonder commentaar
grep -n -E '^\[|path|valid users|invalid users|write list|read list|guest|public|force|#' /etc/samba/smb.conf
testparm -s 2>/dev/null                              # nette uitvoer zoals Samba het ziet
smbclient -L //localhost -N                          # shares opvragen zonder wachtwoord (= test gast)
smbstatus                                            # wie is nu verbonden
```
**Commentaar met `#` ook lezen!** In de proeftoets: `#administratie toegevoegd per 3 januari 2023 op verzoek van Salma`.

Checklijst per share:

| Instelling | Moet zijn | Fout als |
|---|---|---|
| `valid users` | `@afdeling, @management` | losse naam (`anniestryker`), `gast`, verkeerde `@groep` |
| `guest ok` / `public` | `no` | `yes` |
| `read only` / `writable` | `no` / `yes` voor eigen afdeling | – |
| `force user` / `force group` | niet nodig | `force user = root` (alles als root!) |
| `[global] map to guest` | `never` | `bad user` / `bad password` |
| `browseable` | mag `yes` | – |

---

## 6. Oplossen: de 3 kwetsbaarheden uit W2

### Altijd eerst
```bash
cp /etc/samba/smb.conf /etc/samba/smb.conf.bak.$(date +%F)
date                              # tijd voor logboek
```

### 6.1 Kwetsbaarheid 1 · Gastaccount is actief
```bash
smbpasswd -d gast                 # Samba: uitschakelen   (Disabled user gast.)
usermod -L gast                   # Linux: wachtwoord op slot
usermod -s /usr/sbin/nologin gast # geen shell
usermod -e 1 gast                 # account verlopen (extra)
```
Helemaal verwijderen (alleen als de opdracht dat zegt, uitschakelen is veiliger):
```bash
pdbedit -x gast                   # uit Samba
userdel gast                      # uit Linux (userdel -r = ook homemap weg)
```
Test:
```bash
pdbedit -L -v gast | grep 'Account Flags'    # [DU         ]
passwd -S gast                               # gast L ...
smbclient //localhost/iedereen -U gast       # NT_STATUS_ACCOUNT_DISABLED of LOGON_FAILURE
```

### 6.2 Kwetsbaarheid 2 · Gast heeft rechten op mappen
```bash
nano /etc/samba/smb.conf
```
Pas aan (zoeken in nano: `Ctrl+W`, opslaan `Ctrl+O` + Enter, sluiten `Ctrl+X`):
```ini
[global]
   map to guest = never
   restrict anonymous = 2

[management]
   comment = Management
   path = /home/fileserver/management
   valid users = @management
   browseable = yes
   writable = yes
   read only = no
   guest ok = no
```
En bij **elke** share: `guest ok = no` en `gast` weg uit `valid users`.

Snel met sed (alle `guest ok = yes` → `no` in één keer):
```bash
sed -i 's/guest ok = yes/guest ok = no/I; s/public = yes/public = no/I' /etc/samba/smb.conf
sed -i 's/map to guest = bad user/map to guest = never/I' /etc/samba/smb.conf
sed -i '/valid users/ s/,\s*gast//g' /etc/samba/smb.conf         # ", gast" weghalen uit valid users
grep -n -E 'guest|valid users|map to' /etc/samba/smb.conf        # controleren
```
Toepassen:
```bash
testparm -s                       # "Loaded services file OK." = goed
systemctl restart smbd nmbd
systemctl status smbd --no-pager
```
Test:
```bash
smbclient -L //localhost -N                  # NT_STATUS_ACCESS_DENIED / LOGON_FAILURE
smbclient //localhost/management -N          # geweigerd
smbclient //localhost/management -U gast     # geweigerd
```

### 6.3 Kwetsbaarheid 3 · Rechten/eigenaarschap Management te ruim
```bash
ls -ld /home/fileserver/management                     # vóór: drwxrwxrwx root root
chown -R root:management /home/fileserver/management   # eigenaar root, groep management
chmod 2770 /home/fileserver/management                 # alleen groep, nieuwe bestanden krijgen groep
chmod -R g+rw,o-rwx /home/fileserver/management        # bestaande bestanden: anderen eruit
ls -ld /home/fileserver/management                     # na: drwxrws--- root management
```
Alle afdelingsmappen in één keer goedzetten ("eigenaarschap verkeerd geconfigureerd"):
```bash
for a in administratie magazijn management sales iedereen; do
  chown -R root:$a /home/fileserver/$a
  chmod 2770 /home/fileserver/$a
  chmod -R o-rwx /home/fileserver/$a
done
ls -la /home/fileserver
```
In smb.conf zorgen dat nieuwe bestanden ook goed staan:
```ini
   create mask = 0660
   directory mask = 2770
```
Test:
```bash
sudo -u anniestryker ls /home/fileserver/management    # Permission denied
sudo -u hannekecool  ls /home/fileserver/management    # werkt
smbclient //localhost/management -U anniestryker       # NT_STATUS_ACCESS_DENIED
smbclient //localhost/management -U hannekecool -c ls  # werkt (Hanneke moet erbij blijven!)
```

---

## 7. Oplossen: overige afwijkingen uit W1

```bash
# Bram Burg (niet op personeelslijst): eerst uitschakelen, na akkoord verwijderen
smbpasswd -d bramburg ; usermod -L -s /usr/sbin/nologin bramburg
gpasswd -d bramburg sales ; gpasswd -d bramburg iedereen
# verwijderen:  pdbedit -x bramburg ; userdel -r bramburg   (maak eerst backup: tar czf /root/bramburg.tgz /home/bramburg)

# Younes Bulut hoort alleen bij Sales
gpasswd -d younesbulut administratie

# Iemand aan een groep toevoegen (-a NIET vergeten, anders verliest hij andere groepen)
usermod -aG sales younesbulut         # of: gpasswd -a younesbulut sales

# Annie Stryker direct op shares -> weghalen uit valid users (zij zit al in @sales en @iedereen)
sed -i '/valid users/ s/,\s*anniestryker//g' /etc/samba/smb.conf

# @administratie op share sales (toegevoegd "op verzoek van Salma", geen akkoord directie)
sed -i '/^\[sales\]/,/^\[/ { /valid users/ s/,\s*@administratie//g }' /etc/samba/smb.conf
#   (of met nano handmatig weghalen; noem in het rapport dat de directie moet beslissen)

# Namen: volledige naam invullen
usermod -c "Salma Aslan" salmademirci
usermod -c "Ayoub Ahmadi" ayoubozcan
# Gebruikersnaam hernoemen kan (usermod -l), maar raakt homemap en Samba-account:
#   usermod -l salmaaslan -d /home/salmaaslan -m salmademirci ; pdbedit -x salmademirci ; smbpasswd -a salmaaslan
#   -> liever ADVIES geven: eerst verifiëren bij directie/personeelsadministratie

testparm -s && systemctl restart smbd
```

Nieuwe gebruiker goed aanmaken (voor het geval dat):
```bash
useradd -m -s /bin/bash -c "Voornaam Achternaam" -G sales,iedereen gebruikersnaam
passwd gebruikersnaam
smbpasswd -a gebruikersnaam          # Samba-wachtwoord
```

---

## 8. Testen + logs

```bash
# toegang testen
smbclient -L //localhost -U hannekecool                       # shares zien met wachtwoord
smbclient //localhost/sales -U younesbulut -c 'ls'            # mag Younes in sales? (ja)
smbclient //localhost/administratie -U younesbulut -c 'ls'    # na fix: NT_STATUS_ACCESS_DENIED
sudo -u younesbulut ls /home/fileserver/administratie         # bestandsniveau

# logs: heeft gast (of Bram) ooit ingelogd?
last gast ; last bramburg
lastlog -u gast
grep -i -E 'gast|bramburg' /var/log/auth.log*
ls -la /var/log/samba/                 # per pc een log.<naam/ip>
grep -ri gast /var/log/samba/ 2>/dev/null | tail
journalctl -u smbd --no-pager | tail -n 50
```
Logging uitbreiden (advies + kun je doen):
```ini
[global]
   log level = 1 auth_audit:3
   vfs objects = full_audit
   full_audit:success = connect open mkdir rmdir unlink rename
   full_audit:failure = connect
   full_audit:prefix = %u|%I|%S
```

---

## 9. Controlescript: alles in één keer

Typ dit (of plak het) en maak screenshots van de uitvoer. Het past niets aan, het laat alleen zien.

```bash
nano /root/audit.sh
```
```bash
#!/bin/bash
echo "===== $(hostname) - $(date) ====="
echo; echo "== 1. GEBRUIKERS (uid>=1000) =="
awk -F: '$3>=1000 && $3<65534 {print $1" | naam="$5" | shell="$7}' /etc/passwd
echo; echo "== 2. STATUS ACCOUNTS (P=actief L=op slot) =="
for u in $(awk -F: '$3>=1000 && $3<65534 {print $1}' /etc/passwd); do passwd -S $u; done
echo; echo "== 3. SAMBA-GEBRUIKERS (D = disabled) =="
pdbedit -L -v 2>/dev/null | grep -E 'Unix username|Account Flags'
echo; echo "== 4. GROEPEN =="
for u in $(awk -F: '$3>=1000 && $3<65534 {print $1}' /etc/passwd); do echo "$u: $(id -nG $u)"; done
echo; grep -E '^(administratie|magazijn|management|sales|iedereen):' /etc/group
echo; echo "== 5. RECHTEN FILESERVER =="
ls -la /home/fileserver
echo; echo "== 6. SAMBA SHARES =="
grep -n -E '^\[|path|valid users|guest|public|map to guest|#' /etc/samba/smb.conf
echo; echo "== 7. TEST ANONIEM =="
smbclient -L //localhost -N 2>&1 | head -n 15
```
```bash
chmod +x /root/audit.sh
/root/audit.sh | less              # bladeren met spatie, q = stoppen
/root/audit.sh > /root/audit_voor.txt   # opslaan (doe het later nog eens: audit_na.txt)
diff /root/audit_voor.txt /root/audit_na.txt    # verschil vóór/na = bewijs voor je logboek
```

---

## 10. Noodgevallen / foutmeldingen

| Probleem | Oplossing |
|---|---|
| `Permission denied` bij commando | `sudo` ervoor of `sudo -i` |
| `smbclient: command not found` | `apt install smbclient` (als internet werkt) of test via Windows-pc: `\\IP\management` |
| Samba start niet na wijziging | `testparm -s` geeft de foute regel; of backup terug: `cp /etc/samba/smb.conf.bak.* /etc/samba/smb.conf` |
| Wijziging werkt niet | `systemctl restart smbd nmbd` vergeten |
| `NT_STATUS_LOGON_FAILURE` bij een echte gebruiker | Samba-wachtwoord ≠ Linux-wachtwoord. Opnieuw zetten: `smbpasswd username` |
| `NT_STATUS_ACCESS_DENIED` | Toegang geweigerd = bij een test van gast **goed** |
| `NT_STATUS_BAD_NETWORK_NAME` | Sharenaam fout gespeld (kijk `smbclient -L //localhost -U hannekecool`) |
| nano: opslaan | `Ctrl+O`, Enter, `Ctrl+X` |
| Toetsenbord `|` of `/` vinden | `|` = Shift + \ (naast Enter of linksonder) |
