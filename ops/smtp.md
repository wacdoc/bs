# Izgradite vlastiti SMTP server za slanje pošte

## preambula

SMTP može direktno kupiti usluge od dobavljača u oblaku, kao što su:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Ali cloud push email](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Takođe možete izgraditi sopstveni server za poštu - neograničeno slanje, niska ukupna cena.

U nastavku ćemo pokazati korak po korak kako da izgradimo vlastiti mail server.

## Izbor servera

Samohostovani SMTP server zahteva javnu IP adresu sa otvorenim portovima 25, 456 i 587.

Uobičajeni javni oblaci su blokirali ove portove prema zadanim postavkama i možda ih je moguće otvoriti izdavanjem radnog naloga, ali je to ipak vrlo problematično.

Preporučujem kupovinu od hosta koji ima otvorene ove portove i koji podržava postavljanje obrnutih imena domena.

Evo, preporučujem [Contabo](https://contabo.com) .

Contabo je hosting provajder sa sedištem u Minhenu, Nemačka, osnovan 2003. godine sa veoma konkurentnim cenama.

Ako odaberete euro kao valutu kupovine, cijena će biti jeftinija (server sa 8 GB memorije i 4 CPU-a košta oko 529 juana godišnje, a početna instalacija je besplatna godinu dana).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Prilikom narudžbe, napomenite `prefer AMD` i server sa AMD CPU-om će imati bolje performanse.

U nastavku ću uzeti Contabo-ov VPS kao primjer da pokažem kako da napravite svoj vlastiti server za poštu.

## Ubuntu sistemska konfiguracija

Operativni sistem ovdje je Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Ako server na ssh-u prikaže `Welcome to TinyCore 13!` (kao što je prikazano na slici ispod), to znači da sistem još nije instaliran. Isključite ssh i pričekajte nekoliko minuta da se ponovo prijavite.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Kada se pojavi `Welcome to Ubuntu 22.04.1 LTS` , inicijalizacija je završena i možete nastaviti sa sljedećim koracima.

### [Opcionalno] Inicijalizirajte razvojno okruženje

Ovaj korak nije obavezan.

Radi praktičnosti, stavio sam instalaciju i konfiguraciju sistema ubuntu softvera na [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Pokrenite sljedeću naredbu za instalaciju jednim klikom.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Kineski korisnici, koristite sljedeću naredbu umjesto toga, a jezik, vremenska zona itd. će se automatski postaviti.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo omogućava IPV6

Omogućite IPV6 tako da SMTP može slati i e-poštu s IPV6 adresama.

uredi `/etc/sysctl.conf`

Izmijenite ili dodajte sljedeće redove

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Nastavite sa [vodičem za kontabo: Dodavanje IPv6 veze na vaš server](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Uredite `/etc/netplan/01-netcfg.yaml` , dodajte nekoliko redova kao što je prikazano na slici ispod (Contabo VPS default konfiguracijski fajl već ima ove redove, samo ih dekomentirajte).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Zatim `netplan apply` da bi modificirana konfiguracija stupila na snagu.

Nakon što je konfiguracija uspješna, možete koristiti `curl 6.ipw.cn` da vidite ipv6 adresu vaše vanjske mreže.

## Klonirajte ops spremišta konfiguracije

```
git clone https://github.com/wactax/ops.soft.git
```

## Generirajte besplatni SSL certifikat za ime vaše domene

Slanje pošte zahtijeva SSL certifikat za šifriranje i potpisivanje.

Koristimo [acme.sh](https://github.com/acmesh-official/acme.sh) za generiranje certifikata.

acme.sh je automatizirani alat za potpisivanje certifikata otvorenog koda,

Unesite konfiguracijsko skladište ops.soft, pokrenite `./ssl.sh` i `conf` folder će biti kreiran u **gornjem direktoriju** .

Pronađite svog DNS provajdera na [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , uredite `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Zatim pokrenite `./ssl.sh 123.com` da generišete `123.com` i `*.123.com` sertifikate za ime vašeg domena.

Prvo pokretanje će automatski instalirati [acme.sh](https://github.com/acmesh-official/acme.sh) i dodati zakazani zadatak za automatsku obnovu. Možete vidjeti `crontab -l` , postoji takva linija kako slijedi.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Putanja za generirani certifikat je nešto poput `/mnt/www/.acme.sh/123.com_ecc。`

Obnova certifikata će pozvati `conf/reload/123.com.sh` skriptu, uredite ovu skriptu, možete dodati komande kao što je `nginx -s reload` da osvježite keš certifikata povezanih aplikacija.

## Izgradite SMTP server sa chasquidom

[chasquid](https://github.com/albertito/chasquid) je SMTP server otvorenog koda napisan na Go jeziku.

Kao zamjena za drevne mail serverske programe kao što su Postfix i Sendmail, chasquid je jednostavniji i lakši za korištenje, a lakši je i za sekundarni razvoj.

Pokreni `./chasquid/init.sh 123.com` će se automatski instalirati jednim klikom (zamijenite 123.com imenom vašeg domena za slanje).

## Konfigurirajte DKIM potpis e-pošte

DKIM se koristi za slanje potpisa e-pošte kako bi se spriječilo da se pisma tretiraju kao neželjena pošta.

Nakon što se naredba uspješno pokrene, od vas će biti zatraženo da postavite DKIM zapis (kao što je prikazano ispod).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Samo dodajte TXT zapis u svoj DNS (kao što je prikazano ispod).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Pogledajte status usluge i zapise

 `systemctl status chasquid` Pogledajte status usluge.

Stanje normalnog rada je kao što je prikazano na donjoj slici

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` ili `journalctl -xeu chasquid` može vidjeti dnevnik grešaka.

## Obrnuta konfiguracija naziva domene

Obrnuto ime domene omogućava da se IP adresa razriješi na odgovarajuće ime domene.

Postavljanje obrnutog imena domene može spriječiti da se e-pošta identifikuje kao neželjena pošta.

Kada se mail primi, server primatelj će izvršiti obrnutu analizu imena domena na IP adresi servera koji šalje kako bi potvrdio da li server koji šalje ima valjano obrnuto ime domene.

Ako server koji šalje nema obrnuti naziv domene ili ako obrnuti naziv domene ne odgovara IP adresi servera koji šalje, server primatelj može prepoznati e-poštu kao neželjenu poštu ili je odbiti.

Posjetite [https://my.contabo.com/rdns](https://my.contabo.com/rdns) i konfigurirajte kao što je prikazano ispod

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Nakon postavljanja obrnutog imena domene, ne zaboravite konfigurirati prosljeđivanje naziva domene ipv4 i ipv6 na server.

## Uredite ime hosta chasquid.conf

Modificirajte `conf/chasquid/chasquid.conf` na vrijednost obrnutog imena domene.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Zatim pokrenite `systemctl restart chasquid` da ponovo pokrenete uslugu.

## Backup conf u git spremište

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Na primjer, pravim sigurnosnu kopiju conf foldera u svom github procesu na sljedeći način

Prvo napravite privatno skladište

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Unesite conf imenik i pošaljite u skladište

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Dodaj pošiljaoca

trči

```
chasquid-util user-add i@wac.tax
```

Može dodati pošiljaoca

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Provjerite je li lozinka ispravno postavljena

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Nakon dodavanja korisnika, `chasquid/domains/wac.tax/users` će se ažurirati, ne zaboravite ga poslati u skladište.

## DNS dodaje SPF zapis

SPF (Sender Policy Framework) je tehnologija za verifikaciju e-pošte koja se koristi za sprečavanje prevare putem e-pošte.

On provjerava identitet pošiljatelja pošte provjeravajući da li IP adresa pošiljaoca odgovara DNS zapisima imena domene za koju tvrdi da je, sprječavajući prevarante da šalju lažne e-poruke.

Dodavanje SPF zapisa može spriječiti da se e-pošta identifikuje kao neželjena što je više moguće.

Ako vaš server imena domena ne podržava SPF tip, samo dodajte zapis tipa TXT.

Na primjer, SPF za `wac.tax` je sljedeći

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF za `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Imajte na umu da sam ovdje `include:_spf.google.com` , to je zato što ću kasnije konfigurirati `i@wac.tax` kao adresu za slanje u Google poštanskom sandučetu.

## DNS konfiguracija DMARC

DMARC je skraćenica od (Domain-based Message Authentication, Reporting & Conformance).

Koristi se za hvatanje SPF odbijanja (možda uzrokovanih greškama u konfiguraciji ili se neko drugi pretvara da ste vi da šalje neželjenu poštu).

Dodaj TXT zapis `_dmarc` ,

Sadržaj je sljedeći

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Značenje svakog parametra je sljedeće

### p (Politika)

Označava kako postupati s e-poštom koja nije uspjela SPF (Sender Policy Framework) ili DKIM (DomainKeys Identified Mail) verifikaciju. Parametar p može se postaviti na jednu od tri vrijednosti:

* ništa: Ne preduzima se nikakva radnja, samo se rezultat verifikacije vraća pošiljaocu putem mehanizma za prijavu e-pošte.
* Karantena: Stavite poštu koja nije prošla verifikaciju u folder neželjene pošte, ali neće direktno odbiti poštu.
* odbiti: Direktno odbaciti e-poštu koja nije uspjela provjeravati.

### fo (Opcije kvara)

Određuje količinu informacija koje vraća mehanizam za izvještavanje. Može se postaviti na jednu od sljedećih vrijednosti:

* 0: Prijavi rezultate validacije za sve poruke
* 1: Prijavite samo poruke koje nisu uspjele provjeru
* d: Prijavite samo greške u verifikaciji imena domena
* s: prijavi samo greške SPF verifikacije
* l: Prijavite samo greške u DKIM verifikaciji

### rua & ruf

* rua (URI za izvještavanje za zbirne izvještaje): E-mail adresa za primanje zbirnih izvještaja
* ruf (URI za izvještavanje za forenzičke izvještaje): adresa e-pošte za primanje detaljnih izvještaja

## Dodajte MX zapise za prosljeđivanje e-pošte na Google Mail

Budući da nisam mogao pronaći besplatno korporativno poštansko sanduče koje podržava univerzalne adrese (Catch-All, može primati bilo koju e-poštu poslanu na ime ovog domena, bez ograničenja na prefikse), koristio sam chasquid da proslijedim sve e-poruke u svoj Gmail poštanski sandučić.

**Ako imate svoje plaćeno poslovno sanduče, nemojte mijenjati MX i preskočite ovaj korak.**

Uredite `conf/chasquid/domains/wac.tax/aliases` , postavite sanduče za prosljeđivanje

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` označava sve e-poruke, `i` je prefiks adrese e-pošte korisnika koji šalje poruku kreiran iznad. Da bi proslijedio poštu, svaki korisnik mora dodati liniju.

Zatim dodajte MX zapis (pokazujem direktno na adresu obrnutog imena domena ovdje, kao što je prikazano u prvom redu na slici ispod).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Nakon što je konfiguracija završena, možete koristiti druge adrese e-pošte za slanje e-pošte na `i@wac.tax` i `any123@wac.tax` da vidite možete li primati e-poštu u Gmailu.

Ako nije, provjerite chasquid dnevnik ( `grep chasquid /var/log/syslog` ).

## Pošaljite e-mail na i@wac.tax sa Google Mail-om

Nakon što je Google Mail primio mail, naravno, nadao sam se da ću odgovoriti sa `i@wac.tax` umjesto i.wac.tax@gmail.com.

Posjetite [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) i kliknite na "Dodaj drugu adresu e-pošte".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Zatim unesite verifikacioni kod primljen putem e-pošte na koju je proslijeđen.

Konačno, može se postaviti kao zadana adresa pošiljaoca (zajedno sa opcijom da se odgovori istom adresom).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Na ovaj način smo završili uspostavljanje SMTP mail servera i istovremeno koristimo Google Mail za slanje i primanje e-pošte.

## Pošaljite probni email da provjerite da li je konfiguracija uspješna

Unesite `ops/chasquid`

Pokrenite `direnv allow` instalaciju zavisnosti (direnv je instaliran u prethodnom procesu inicijalizacije s jednim ključem i zakačivanje je dodano u ljusku)

onda trči

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Značenje parametara je sljedeće

* korisnik: SMTP korisničko ime
* pass: SMTP lozinka
* za: primaoca

Možete poslati probni email.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Preporučuje se korištenje Gmaila za primanje testnih e-poruka kako biste provjerili jesu li konfiguracije uspješne.

### TLS standardna enkripcija

Kao što je prikazano na slici ispod, postoji ova mala brava, što znači da je SSL certifikat uspješno omogućen.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Zatim kliknite na "Prikaži originalnu e-poštu"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Kao što je prikazano na donjoj slici, Gmail originalna stranica pošte prikazuje DKIM, što znači da je DKIM konfiguracija uspješna.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Provjerite Received u zaglavlju originalne e-pošte i možete vidjeti da je adresa pošiljatelja IPV6, što znači da je IPV6 također uspješno konfigurisan.
