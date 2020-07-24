# Login gennem KOMBIT Adgangsstyring
Denne vejledning beskriver, hvordan man som tjenesteudbyder opsætter sin applikation, så den tilbyder login gennem KOMBIT Adgangsstyring. Login går gennem KIT's infrastruktur, og man får således mulighed for at logge ind gennem Adgangsstyring, uden at man skal oprettes som tjenesteudbyder i KOMBITs infrastruktur. 

KOMBIT Adgangsstyring er en fælleskommunal løsning, der fungerer som bindeled mellem kommunale brugerdatabaser (f.eks. AD) og brugervendte systemer, så det brugervendte system kun skal håndtere login gennem Adgangsstyring.

Vejledningen beskriver hvilke skridt man skal udføre for at gennemføre opkoblingen. Der medfølger også et demo-setup, der demonstrerer login.

## Forudsætninger
For at køre setuppet har man brug for at have [Docker Community Edition](https://docs.docker.com/install/) installeret. 

### TODO: få styr på nedenstående
Man skal desuden have en konto i KIT's docker registry, kitdocker.kvalitetsit.dk. Man kan skrive til njo@kvalitetsit.dk eller mads@kvalitetsit.dk, hvis man ikke har fået eller ikke kan huske sit brugernavn og password.

## Det udleverede setup
Setuppet består af en række containere, beskrevet i filen docker-compose.yml, samt en række filer, som vist her:

```
.
├── client
│   ├── appa
│   │   └── config
│   │       ├── appa.cert
│   │       └── appa.pem
│   ├── docker-compose.yml
├── config
│   └── metadata_broker_azure.xml
├── docker-compose.yml
├── venliglogin
│   ├── cert
│   │   ├── server.crt
│   │   └── server.pem
│   ├── venliglogin-auth
│   │   ├── saml20-sp-remote.php
│   └── venliglogin-metadata
│       ├── saml20-idp-remote.php
│       └── saml20-sp-remote.php
└── vlss
    └── config
        ├── vlss.cert
        └── vlss.pem

```

Overordnet er filerne opdelt som følger:
- config: Metadata for KeyCloak
- venliglogin: Nøglepar for VenligLogin-idp samt metadata-filer.
- vlss: Nøglepar for VenligLogin web-applikation

Man starter setuppet ved at køre:

```
docker login kitdocker.kvalitetsit.dk (kun nødvendigt første gang)
docker-compose up
```

Når containerne er startet op, kan man afprøve setuppet med testapplikationen i _client_-folderen.

```
cd client
docker-compose up
```

Når containerne er oppe tilgår man _http://localhost:8082/appa/_ i en browser. Når man logger ind første gang gennem VenligLogin, bliver man videresendt til KeyCloak:

![keycloak](images/keycloak_login.png)

Til testformål kan man logge ind med brugernavn/password _test/test_. Når man er logget succesfuldt ind, sendes man tilbage til VenligLogin, og får lov til at aktivere VenligLogin.

![venliglogin](images/aktiver_venliglogin.png)

Følg vejledningen på skærmen for at aktivere VenligLogin. Til sidst bliver man sendt tilbage til http://localhost:8082/appa, nu logget ind:

![succes](images/succes.png)

Hvis man får vist ovenstående skærmbillede, så har man succesfuldt fået setuppet til at køre.

## Opsætning
Setuppet består som tidligere nævnt af en række containere, samt en række filer. Man bør generere nye nøglepar i stedet for de medfølgende, som er selvudstedt af KvalitetsIT, og kun må anvendes til udviklingsformål. Ud over dette skal man ændre en række miljøvariabler, som vil blive beskrevet i det følgende. Det beskrives hvilke miljøvariabler man har brug for at ændre, men der gives _ikke_ en beskrivelse af hver enkelt miljøvariabel i setuppet.

### citizen-venliglogin
VenligLogin-IdP'en. Man har brug for at rette:

- IDP_ADMIN_PASSWORD
-- Admin-password til VenligLogin.
- IDP_TECHNICAL_EMAIL
-- E-mail på administrator for VenligLogin.
- IDP_SECRET_SALT
-- Bruges til at generere hashværdier. SimpleSAML foreslår at generere værdien med kommandoen _tr -c -d '0123456789abcdefghijklmnopqrstuvwxyz' </dev/urandom | dd bs=32 count=1 2>/dev/null;echo_
- IDP_HOSTNAME
-- Sættes til det domæne, som VenligLogin skal køre på.
- IDP_PROTOCOL
-- Sættes til https
- VENLIGLOGIN_WEBGUI_URL
-- Sættes til den URL, hvor ui'et for VenligLogin ligger.

### venliglogin-webgui-saml-sp
Sidevogn for VenligLogin's brugergrænseflade. Man har brug for at rette: 

- IDP_LOGIN_URL
-- Er i setuppet sat til testudgaven af KIT's KeyCloak-installation, men skal senere rettes til at pege på produktionsmiljøet.
- SP_ENTITY_ID
-- Rettes til et ønsket entityID.
- SAML_LB_SCHEME
-- Rettes til https.
- SAML_LB_SERVERNAME
-- Rettes til det domæne, som VenligLogin skal køre på.
- SAML_LB_SERVERPORT
-- Rettes til den port, som VenligLogin skal køre på.

## Udveksling af metadata
Der skal udveksles et antal metadata-filer som en del af installationen, hvilket vil blive beskrevet i det følgende. Der skal dels  _genereres_ et antal metadata-filer, som sendes til KvalitetsIT, og _installeres_ et antal metadata-filer.

### Generering af metadata-filer
Der skal genereres to metadata-filer, som sendes til KvalitetsIT. De genereres ved at tilgå følgende adresser:

 * http://localhost:8092/venliglogin/module.php/saml/sp/metadata.php/default-sp
 * http://localhost:8083/vlss/saml/metadata

hvor domæne og protokol er erstattet med det domæne, som VenligLogin skal køre på.

### Installation af metadata-filer
Den vedlagte metadata-fil for KeyCloak, _config/metadata_broker_azure.xml_, skal installeres i containeren _venliglogin-webgui-saml-sp_. Dette gøres meget simpelt ved at volume-mounte filen på stien _/app/config/_ (er gjort i docker-compose setuppet).

Containeren _citizen-venliglogin_ udstiller metadata i php-format, som skal genereres ud fra metadata i xml-format. Denne konvertering udføres ved brug af SimpleSAML's metadata-værktøj. Når setuppet kører (med docker-compose up), tilgår man værktøjet på http://localhost:8092/venliglogin-auth/admin/metadata-converter.php.

![metadata_parser](images/metadata_parser.png)

De filer der skal tilpasses er:

```
venliglogin-auth
└── saml20-sp-remote.php
venliglogin-metadata
├── saml20-idp-remote.php
└── saml20-sp-remote.php
```

For at generere filerne gøres som beskrivet i det følgende.

#### venliglogin-auth/saml20-sp-remote.php
Åbn metadata-converteren, og indsæt indholdet af metadata-filen på http://localhost:8092/venliglogin/module.php/saml/sp/metadata.php/default-sp, hvor _http://localhost:8092_ erstattes med den adresse, hvor VenligLogin skal køre. Erstat metadata-entry'en i saml20-sp-remote.php med outputtet fra værktøjet.

#### venliglogin-metadata/saml20-idp-remote.php
Åbn metadata-converteren, og indsæt indholdet af KeyCloak metadata-filen. Åbn den udleverede saml20-idp-remote.php-fil, og erstat den øverste metadata-entry (linje 3), med outputtet fra værktøjet. Der skal tilføjes en selectid-attribut i konfigurationen, som følger (linje to i eksemplet):

```
$metadata['https://keycloak.kvalitetsit.dk/auth/realms/broker'] = array (
  'selectid' => 'default',
  'entityid' => 'https://keycloak.kvalitetsit.dk/auth/realms/broker',
```

Hent metadata for VenligLogin på http://localhost:8092/venliglogin-auth/saml2/idp/metadata.php, hvor _http://localhost:8092_ erstattes med den adresse, hvor VenligLogin skal køre. Erstat den nederste metadata-entry i saml20-idp-remote.php med outputtet fra værktøjet. Der skal igen tilføjes en selectid-attribut i konfigurationen, som følger (linje to i eksemplet):

```
$metadata['http://localhost:8092/venliglogin-auth/saml2/idp/metadata.php'] = array (
  'selectid' => 'twofactor',
  'entityid' => 'http://localhost:8092/venliglogin-auth/saml2/idp/metadata.php',
```

#### venliglogin-metadata/saml20-sp-remote.php
Åbn metadata-converteren, og indsæt indholdet af metadata-filen for MobilizeMe. Erstat metadata-entry'en i saml20-sp-remote.php med outputtet fra værktøjet. 

## Test af setuppet
Når metadata er udvekslet kan setuppet testes som beskrevet tidligere.
