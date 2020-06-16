# Immutable-webapp
En implementasjon av stukturen fra https://immutablewebapps.org/

[Slides](https://docs.google.com/presentation/d/1gcnwG0NzTiAlQ9NrjWCTa6c0yCiKYEkowBLn9BSKbjA/present)

## Forberedelser

- Opprett en fork av dette repoet på din egen bruker og klon det ned på egen maskin
- Sjekk at `node` og `npm` er installert
- `brew install awscli`
- `brew install terraform`
- `git --version` er større en 2.9 (om du har lavere versjon, drop githook som er nevnt senere)
- Opprett en AWS-konto på https://aws.amazon.com/.
    - Bruk Basic-versjonen
    - Legg inn betalingskort <- følg med på kostnadene og husk å slette infrastrukturen som koster penger når dagen er ferdig
    - Sjekk at du kommer inn på [S3](https://s3.console.aws.amazon.com/s3/home) uten å bli bedt om mer signup  
- Opprett en ny bruker i [IAM](https://console.aws.amazon.com/iam/home?#/users).
    - Add user: username `terraform` og access type `Programmatic access`
    - Permissions: `Attach existing policies directly` og velg policyen med policy name `AdministratorAccess`
    - Tags: name = `system` og value=`terraform`
    - Etter Create,husk å last ned access-key og secret.
- Kjør kommandoen `aws configure` med ACCESS_KEY_ID og SECRET_ACCESS_KEY som du fikk fra brukeren over. Bruk region `eu-north-1`
    - Kommandoen `aws iam get-user` kan brukes som en ping og sjekk av alt ok!
    - Når vi senere skal bruke terraform til å sette opp vår infrastruktur, er det credentials konfigurert gjennom aws-cliet over som terraform også bruker som credentials

Om du allerede nå ser at du vil lage noe under et eget domene, anbefaler jeg å gå inn på AWS Route 53 og opprettet et billig et med en gang. Selv om det sikkert går mye fortere, advarere Amazon om at det kan ta opp til 3 dager.

## Bli kjent

### Om appen

### Lokal oppstart

* Kjør opp appen med `npm install && npm run start`
* Generer en index.html med `node src-index/main.js`
* Gjør deg kjent med hvor de forskjellige inputene og env-variablene i appen kommer fra

## Min første immutable webapp

Felles mål her er en immutable webapp med to S3-buckets og et CDN foran som hoster index.html og kildekode.

Nyttige lenker:
* Om du ikke er veldig kjent i aws-konsollen fra før, anbefaler jeg å sjekke ut de forskjellige servicene
underveise
    - https://console.aws.amazon.com/s3
    - https://console.aws.amazon.com/cloudfront
    - https://console.aws.amazon.com/route53
* [Terraform-docs](https://www.terraform.io/docs/providers/aws/r/s3_bucket.html)
* [AWS-cli-docs](https://docs.aws.amazon.com/cli/latest/reference/s3/cp.html)


### Testmiljø med buckets

Opprett to [buckets](https://www.terraform.io/docs/providers/aws/r/s3_bucket.html) med terraform som skal bli der vi server asset og host. Start i `terraform/test/main.tf`. Husk at S3-bucketnavn må være unike innenfor en region!

Anbefalt terraform-output for begge buckets:
* bucket_domain_name - denne lenken kan du bruke til å aksessere filene du har lastet opp
* id - navnet på bucketen du har opprettet

Når begge bucket er oppprettet uten mer oppsett, og du kan gå inn i konsollen på web og manuelt laste opp en tilfeldig fil. Den vil ikke tilgjengelig på internett via `bucket_domain_name/filnavn`, ettersom default-policyen er at bucket er private. Vi kan konfigurere public tilgang ved å bruke acl-parameteret på en bucket eller en bucket policy. Sistnevnte er anbefalt av AWS  ettersom bucketacl er et eldre og skjørere konsept.

Opprett bucketpolicies for begge bøttene ved å bruke [`aws_s3_bucket_policy`](https://www.terraform.io/docs/providers/aws/r/s3_bucket_policy.html). I policy-atributtet kan du bruke en [templatefile](https://www.terraform.io/docs/configuration/functions/templatefile.html) med fila `policy/public_bucket.json.tpl`. Denne trenger en variabel `bucket_arn`. Bruk atributtet fra bucketen for å sende inn rett arn.

Se [policy.md](terraform/test/policy/policy.md) for en forklaring på innholdet i policyen.


### npm run upload-assets

Gjør endring i `upload-assets.js` og sett navn inn rett navn på bucket. Som version kan du beholde 1 forelpig. Kjør scriptet med `npm run upload-assets` og sjekk at du får den bygde `main.js` lastet opp i bøtta og public tilgjengelig på nett.

### npm run deploy-test

Gjør endring i `deploy-env.js` og sett navn inn rett navn på bucket og rett url til assets-bucket. Som version kan du beholde 1 *eller* sette samme versjon som du gjorde i steget over. Kjør scriptet med `npm run deploy-test` og sjekk at du får den bygde `index.html` lastet opp i bøtta og public tilgjengelig på nett.

Denne fila skal du nå kunne åpne fra bucketen og se appen :rocket:

Dersom du kjører `npm run deploy-test` med samme versjonsnummer en gang til, vil du se at `Build deploy at` endrer seg, mens fargen, heading og `Build created at` er den samme.

###

Ny versjon! Prøv å gjør en endring i koden og deploy en ny versjon! Hvilket tall du velger spiller ingen rolle, men husk å oppdatere versjonen både i `upload-assets.js` og `deploy-env.js`

### CDN

AWS CloudFront er Amazon sin CDN-provider, se [terraform-docs](https://www.terraform.io/docs/providers/aws/r/cloudfront_distribution.html).
Om du gjør dette for første gang anbefaler jeg at du starter med et cloudfront-domene og heller endrer til eget domene i neste steg.

For å mappe terraform-input til rett verdier, anbefaler jeg å se i aws-konsollen på CloudFront og velge "Create a distribution".
En gotcha som er fin å vite om, dersom du [ikke setter verdier i ttl-atributtene](https://github.com/terraform-providers/terraform-provider-aws/issues/1994) til terraform vil dette gjøre at CloudFront velger å bruker cachecontrol-headers fra origin, tilsvarende `Use Origin Cache Headers` fra AWS-console'en.

Figuren bakerst i slidesettet gir en slags oversikt av hvordan CloudFront passer inn som server for både host og assets - men dette var også den vanskeligste delen av oppgaven å beskrive! Så vær så snill å stikk innom Tine eller andre om det ikke gir mening.

Test ut endringer i `App.jsx` og deploy ny versjon av assets og index for å sjekke caching og endringer.
- OBS: Nå kan du bruke `domain_name` outputen fra cloudfront som erstatning for `my-url` i `src-index/main.js`

<details><summary>Tips</summary>
<p>

- du trenger en `origin` pr. s3 bucket
- `enabled`, `restrictions`, `viewer_certificate` kan være default
- `default_root_object` er `index.html`
- `default_cache_behavior` og `ordered_cache_behavior` kan ha like configparameter, men default må peke på host-bucket og ordered_cache_behavior på assets. Path `assets/*` matcher url-strukturen fra index.html

</p>
</details>

Løsningsforslag i repoet frem til hit ligger under https://github.com/kleivane/immutable-webapp/tree/master/terraform/test-1 .

## Videre

Cirka frem til punktet "Lag et eget domene" kan du finne et løsningsforslag i repoet https://github.com/kleivane/immutable-webapp/ under mappene `terraform/test`, `terraform/prod` og `terraform/common`.

* Lag et prodmiljø
* Trekk ut bygging av index.html til en lambda
    * Lambdaen trenger kildekode i egen bucket
    * Provisjoner lambda med terraform pr miljø og send inn versjon av kildekoden som skal brukes


# Notater

## Lage Starterpack

* Klone repoet git clone <ssh> starterpack
* Slett .git-mappa
* Slett stuff under terraform (behold test/main og test/output og test/policy)
* Lag et nytt repo på github
* Slett notatene her
* Kjør git init, add, commit, push til nytt repo

# Tilbakemeldinger 19.mai
* kom kort 🙈
* funker bra med små grupper og skjermdeling
* hvordan angripe CloudFront-delen....? mye config/doc/ukjent
* teste ut import av en ressurss fra AWS
