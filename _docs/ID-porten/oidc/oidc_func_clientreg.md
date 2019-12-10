---
title: Registrering av OIDC klienter
description: Registrering av OIDC klienter
summary: "ID-porten støtter flere typer klienter. Klienter må forhåndsregisteres, og korrekt registering av klient er viktig at sikkerheten skal være ivaretatt."
permalink: oidc_func_clientreg.html
sidebar: oidc
product: ID-porten
---


## Bakgrunn

ID-porten støtter flere typer klienter, og det er kundens ansvar å sørge for at det faktiske bruksmønsteret er i samsvar med registreringen. Korrekt registrering er spesielt viktig for klienter som konsumerer APIer tilbudt av andre, og ID-porten og API-tilbyders bruksavtaler regulerer dette ansvarsforholdet.



## ID-portens integrasjoner {#integrasjoner}

ID-porten håndterer 5 ulike typer av integrasjoner:

* ID-porten
* Kontaktregisteret
* Maskinporten
* API-klient innlogget bruker
* (eFormidling integrasjonspunkt)

Det er viktig å være klar over at disse integrasjonstypene rent teknisk alle er standard Oauth2 klienter, men med ulike egenskaper.  Se detaljer lenger ned.

Vi har 3 måter du kan få registrert din integrasjon:

- Selvbetjening, ved å logge inn på [selvbetjening på Samarbeidsportalen](https://selvbetjening-samarbeid.difi.no/#/).
- Selvbetjening, ved å bruke vårt [selvbetjenings-API](oidc_api_admin.html)
- Manuelt, ved å sende epost til idporten@difi.no  (kun for ID-porten og Kontaktregisteret)

## Oauth2-egenskaper

Dette avsnittet detaljerer noen viktige Oauth2 egenskaper som kategoriserer våre klienter.  Se gjerne [Oauth2 Dynamic Client Registration for mer informasjon (RFC7591)](https://tools.ietf.org/html/rfc7591#section-2).

I utgangspunktet kan du som kunde velge Oauth2-egenskaper fritt etter egen risikovurdering. De som konsumerer API tilbudt av andre må være oppmerksom på at API-tilbyder kan stille krav til spesifikke egenskaper.   Validering av slike krav skjer som hovedregel kun run-time ved tokenutstedelse og ikke ved klient-registrering.




### Klient-autentisering

Alt etter bruksområde, så tilbyr vi forskjellige metoder for autentisering av din klient.  Dette blir styrt av attributtet `token_endpoint_auth_method`:


|Metode|token_endpoint_auth_method|Beskrivelse|
|-|-|-|
| Statisk hemmelighet | client_secret_basic client_secret_post | En statisk hemmelighet (*client_secret*) som Difi genererer og blir utvekslet manuelt, eller tilgjengeliggjort via selvbetjening.  Maks tillatt levetid er satt til 360 dager. Det er kundens ansvar å få rotert hemmeligheten før utløp for å sikre kontinuerlig tjenesteleveranse. |
| Virksomhetssertifikat   | private_key_jwt | Klienten bruker et gyldig virksomhetssertifikat fra Buypass eller Commfides. Organisasjonsnummeret i sertifikatet må stemme med klient-registreringa. Kunden kan valgfritt velge å "låse" klienten til bare et spesifikt virksomhetssertifikat. |
| Asymmetrisk nøkkel  | private_key_jwt | Den offentlige nøkkelen fra et egen-generert asymmetrisk nøkkelpar blir registrert på klient, og klienten bruker privatnøkkelen til å autentisere seg.  For å få lov til å registere slike klienter, må kunden etablere en [egen  selvbetjeningsapplikasjon](oidc_api_admin.html) (som selv må bruke virksomhetssertifikat) |
| Ingen   | none  | Klienten er en såkalt *public*-klient som ikke kan beskytte en hemmelighet på en tilfredstillende måte.  Gjelder single-page-applikasjon og i noen tilfeller mobil-apper  |

Difi anbefaler bruk av virksomhetssertifikat til klientautentisering,  da prosedyren for utstedelsen av slike er grundig regulert i lovverk og gir derfor både Difi og API-tilbydere en god og sikker identifisering av klienten.   

Difi forutsetter at API-tilbyder og API-konsument håndterer sertifikat og nøkler på en måte som sikrer at ikke uvedkommende kan misbruke disse.

**Merk at eksisterende secret slettes dersom man endrer metoden til noe annet enn client_secret_***

### Grant-typer

Et grant representerer brukerens samtykke til å hente et access token (som i sin tur brukes til hente den beskytta ressurs tilhørende brukeren, se [Oauth2, kap 1.3](https://tools.ietf.org/html/rfc6749#section-1.3) samt [`grant_types` i DCR kap. 2](https://tools.ietf.org/html/rfc7591#section-2) )

ID-porten støtter følgende grants:

|Grant-type|Beskrivelse|
|-|-|
|authorization_code         | Autorisasjonskode-flyten, som beskrevet i [RFC 6749 kap 4.1](https://tools.ietf.org/html/rfc6749#section-4.1)  |
|refresh_token      | Klienten bruker eit refresh-token for å hente nytt access-token. Bruker blir (normalt) ikke involvert.  |
|urn:ietf:params:oauth:grant-type:jwt-bearer|En signert JWT ihht [RFC7523](https://tools.ietf.org/html/rfc7523#section-2.1). Kan enten bruke virksomhetssertifikat i `x5c` eller `kid` til forhåndsregistrert asymmetrisk nøkkel.|  
|jwt_bearer_token   | kortform av urn:ietf:params:oauth:grant-type:jwt-bearer    |

Maskinporten-klienter skal alltid bruke `jwt_bearer_token`.

Vi støtter ikke implicit, password eller client-credentials grant.

### Klient-typer

Valg av klient-type er en sikkerhetsvurdering kunden skal utføre.  Vi kategoriserer klienter ved hvordan de autentiserer og identifiserer seg opp mot ID-porten. Dette er i sin tur avhengig av kjøretidsmiljøet til klienten. Vi legger til grunn  på definisjonene fra  [Oauth2 kap 2.1](https://tools.ietf.org/html/rfc6749#section-2.1).


|Klient-type|Oauth2-begrep|tilatt klientautentisering|Beskrivelse|
|-|-|-|-|
| Standard-klient   | Web app   | private_key_jwt client_secret_basic client_secret_post | Typisk en server-side nett-tjeneste som er plassert i et sikkert driftsmiljø.  De aller fleste av ID-portens kunder skal bruke denne klient-typen.  Det er sterkt anbefalt, men ikke påkrevd, å bruke PKCE, samt state- og nonce-parametrene for standardklienter. <p/>Maskinporten-klienter faller alltid i 'standardklient'-kategorien, men her tillates ikke statiske hemmeligheter.  |
| Single-page applikasjon (SPA)   | Brower-based app  | none |Typisk en javascript-klient som fullt og helt lever i brukerens browser.  En slik klient kan ikke beskytte en klient-hemmelighet/virksomhetssertfikat, og blir derfor en *public* klient, den har ingen klientautentisering <p/>Vi følger [de siste anbefalingene fra IETF](https://tools.ietf.org/html/draft-ietf-oauth-browser-based-apps-00), som krever at slike klienter skal bruke autorisasjonskodeflyten, og at både PKCE og state-parameter er påkrevd.  |
| Mobil-app  | Native app | none   | Tilsvarende som for SPAer så kan ikke en mobil-app beskytte en hemmelighet når den blir distribuert gjennom App Store, og blir derfor også en public klient.

Merk at klient-type ikke blir lagret som del av klient-registreringen, men utledet basert på `token_endpoint_auth_method` og `grant_types`.






## Oversikt over kombinasjonar

Tabellen under oppsummerer sammenhengen mellom de ulike egenskapene:


| Integrasjon | Klient-type |  tillatte `token_endpoint_auth_method` | tillatte `grant_types` | scope | Kan legge til scopes? |
|-|-|-|-|-|-|
|ID-porten| web |  client_secret_basic client_secret_post private_key_jwt      | authorization_code refresh_token  |openid profile | nei |
||  browser |  none     | authorization_code   |openid profile | nei |
||  native |   none     | authorization_code   |openid profile | nei |
|API-klient innlogget bruker  |samme som for idporten ||| | ja |
|Maskinporten| web |private_key_jwt  | jwt_bearer_token | |ja|
|Kontaktregisteret| web | private_key_jwt  | jwt_bearer_token |global/kontaktinformasjon.read global/spraak.read global/sikkerdigitalpost.read global/sertifikat.read global/varslingsstatus.read |nei|






Ved bruk av selvbetjenings-API, må kunden passe på å sende konfigurasjoner som er kompatible med tabellen over, ellers risikerer man å ende opp med en ubrukelig klient.

Ved bruk av selvbetjening på Samarbeidsportalen er tilgjengelige valg avgrenset av valgt integrasjonstype.


## Andre metadata

Vi forsøker å følge spec'en i så stor grad som mulig.  Se gjerne [Oauth2 Dynamic Client Registration for mer informasjon (RFC7591)](https://tools.ietf.org/html/rfc7591#section-2).

### Basis-sett

Følgende metadata er felles for alle typer klienter:

|attributt|Påkrevd?|beskrivelse|
|-|-|-|
| client_id | Ja |Unik identifikator for klienten. Blir tildelt av Difi. |
| client_orgno | Ja |Klientens organisasjonsnummer.  Juridisk konsument dersom leverandør-styrt integrasjon. Utleveres som "consumer_orgno" i tokens |
| supplier_orgno   | Nei  | Leverandørens organisansjonummer, dersom integrasjonen er kontrollert av leverandør  |
| scopes | Ja |Liste over scopes som klienten kan forespørre. For innlogging må alltid *openid* være tilstede. |


### Metadata for innloggings-klienter:

Klienter som skal innvolvere brukeren (altså brukerens browser) må ha følgende satt:

|attributt|Påkrevd?|beskrivelse|
|-|-|-|
| display_name | Ja |Klientens organisasjonsnavn som benyttes ved visning på web |
| redirect_uris | Ja| Liste over gyldige url'er som provideren kan redirecte tilbake til etter vellykket autorisasjonsforespørsel. |
| post_logout_redirect_uris | Ja |Liste over url'er som provideren redirecter til etter fullført egen-initiert utlogging. |
| frontchannel_logout_uri | Nei|  URL som provideren sender request til ved utlogging trigget av annen klient i samme sesjon |
| frontchannel_logout_session_required | Nei |Flagg som bestemmer om parameterne for issuer og sesjons-id skal sendes med frontchannel_logout_uri |
| logo | Nei |Logo som vises i innloggingsbildete utveksles p.t. manuelt |


### Metadata for klienter som konsumerer APIer

For klienter (både innlogging og maskin) som mottar *access_token* til API-sikring kan man registrere følgende:

| attributt | beskrivelse |
|-|-|
| authorization_lifetime | Levetid for registrert autorisasjon. Default: 7200 sekunder (2 timer).|
| access_token_lifetime | Levetid for utstedt access_token. Default: 120 sekunder. |
| refresh_token_lifetime |Levetid for utstedt refresh_token. Default: 600 sekunder (10 minutter). |

Merk at de fleste av egenskapene til access_token blir bestemt av API-tilbyder, og ikke som en del av klient-registreringen.  En klient kan for eksempel ikke få token som har lengre levetid enn det API-tilbyder har satt som maks-grense.
