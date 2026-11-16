# Ž-17 RMC užsakymų registras

Vieno puslapio programa (`index.html`) Ž-17 registrui: užsakymai sugrupuoti pagal Iremo užsakymo nr., naujo užsakymo ir RMC eilučių registravimas. Prisijungimas – Microsoft (MSAL), duomenys – du SharePoint sąrašai.

Tai **pagrindas**: viena prieiga visiems, be rolių ir patvirtinimų. Funkcijos bus pridedamos po truputį.

## Kaip veikia

- `index.html` – visa programa (HTML + CSS + JS, be build žingsnio). MSAL biblioteka kraunama iš jsDelivr.
- Prisijungimas: MSAL `loginRedirect`, tokenas SharePoint'ui su scope `https://achemosgrupelt.sharepoint.com/AllSites.Write` (deleguotas, kaip ir `leidimai`).
- Duomenys: SharePoint REST (`/_api/web/lists/getbytitle(...)`). Stulpeliai randami pagal rodomą pavadinimą, todėl tinka sąrašai, sukurti importuojant iš Excel (vidiniai vardai `field_1`, `field_2`…).
- Kol `SP_CONFIG` neužpildytas, programa veikia **demo režimu** su išgalvotais duomenimis.

> Repozitorija vieša – čia laikomas tik kodas. Jokių užsakymų duomenų (Excel failų, eksportų) į repo nekelti.

## Paleidimas

### 1. SharePoint sąrašai

Sąrašai kuriami iš `Z17_SharePoint_importas.xlsx` (failas ne repozitorijoje; jame – lentelės `Uzsakymai` ir `RMC_eilutes`).

1. Įkelkite failą į SharePoint svetainės dokumentų biblioteką.
2. Svetainėje: **Naujas → Sąrašas → Iš „Excel“** → pasirinkite failą.
3. Lentelė `Uzsakymai` → sąrašo pavadinimas **`Z17_Uzsakymai`**. Visi stulpeliai – *Viena teksto eilutė*.
4. Lentelė `RMC_eilutes` → sąrašo pavadinimas **`Z17_RMC_eilutes`**. Tipai:
   - *Data*: Registravimo data, Užsakymo terminas, Atkrovimo data
   - *Skaičius*: Kiekis, Sąmatinė vertė, Norma (val.), Faktas (val.)
   - *Kelios teksto eilutės*: Užsakymo turinys
   - kiti – *Viena teksto eilutė*
5. `Z17_RMC_eilutes` → Sąrašo parametrai → **Indeksuoti stulpeliai** → pridėti indeksą stulpeliui *Užs. Nr.* (eilučių bus > 5000).

Stulpelių pavadinimų nekeiskite – programa jų ieško pagal šiuos pavadinimus (žr. `ORDER_FIELDS` / `LINE_FIELDS` faile `index.html`). Jei kurio nors neranda, viršuje parodo įspėjimą.

### 1a. Užsakymų aplankų nuorodos (neprivaloma)

Atidarius užsakymą programa per Power Automate „folder lookup“ srautą suranda užsakymo aplanką (`IremasUsakymai2 › … › Iremas uzsakymai/<metai>/…`) ir parodo mygtuką **Atidaryti aplanką**. Rasta nuoroda įsimenama naršyklėje, tad srautas kviečiamas vieną kartą kiekvienam užsakymui (mygtukas ↻ – ieškoti iš naujo).

Srauto URL turi `sig=` raktą (veikia kaip slaptažodis), todėl **jo nėra kode** (repo viešas). Jis laikomas SharePoint sąraše:

1. Toje pačioje svetainėje sukurkite tuščią sąrašą **`Z17_Nustatymai`**.
2. Pridėkite stulpelį **Reikšmė** – *Kelios teksto eilutės* (paprastas tekstas; URL ilgesnis nei 255 simbolių).
3. Įrašas: Title = `FolderLookupUrl`, Reikšmė = srauto HTTP trigger URL.
4. Teises į sąrašą rekomenduojama riboti (skaityti – tik programos naudotojams, keisti – IT).

Jei sąrašo ar įrašo nėra, automatinė paieška neveikia.

**Nuorodos išsaugojimas.** Sąraše `Z17_Uzsakymai` sukurkite stulpelį **Aplankas** (*Kelios teksto eilutės*). Rastas aplankas įrašomas į užsakymą, todėl kitą kartą (ir kitiems naudotojams) paieška nebekartojama. Jei paieška aplanko neranda, nuorodą galima įklijuoti ranka („Įvesti nuorodą“). Be šio stulpelio nuorodos įsimenamos tik naršyklėje.

**Diagnostika.** Užvedus pelę ant „Aplankas nerastas“ / klaidos pranešimo matomas srauto atsakymas; jis taip pat rodomas „Įvesti nuorodą“ lange ir naršyklės konsolėje (F12).

### 2. Entra ID programos registracija

1. Entra ID → App registrations → **New registration** (pvz. „Iremas Ž-17“), *Single tenant*.
2. **Authentication → Add a platform → Single-page application**, Redirect URI – tikslus programos adresas, pvz.:
   - `https://iremas-automation.github.io/z17/`
   - (vėliau, jei bus savas domenas) `https://z17.iremas.lt/`
3. **API permissions → Add → SharePoint → Delegated → `AllSites.Write`**. Admin consent nereikalingas, bet jei grupės IT politika to reikalauja – paprašyti grupės IT.
4. Nukopijuokite *Application (client) ID*.

### 3. Konfigūracija

`index.html` viršuje:

```js
const SP_CONFIG = {
  tenantId: 'f300c84f-87f0-4d9e-8284-1e36dbe06fbe',
  clientId: '<Application (client) ID>',
  siteUrl:  'https://achemosgrupelt.sharepoint.com/sites/<svetainė>',
  ordersList: 'Z17_Uzsakymai',
  linesList:  'Z17_RMC_eilutes'
};
```

Client ID ir Tenant ID nėra slapti. Prieigą riboja SharePoint teisės: programa gali daryti tik tai, ką prisijungęs vartotojas gali daryti svetainėje.

### 4. GitHub Pages

Settings → Pages → *Deploy from a branch* → `main` / `/ (root)`. Adresas: `https://iremas-automation.github.io/z17/`.

## Duomenų modelis

| Sąrašas | Vienas įrašas | Pagrindiniai stulpeliai |
|---|---|---|
| `Z17_Uzsakymai` | Iremo užsakymas | Užs. Nr. (Title), Projektas, Užsakovas, Objekto vieta |
| `Z17_RMC_eilutes` | RMC eilutė | RMC Nr. (Title), Užs. Nr., datos, turinys, kiekis, vertė, PO Nr., dokumentacija, vykdo, technologas, norma, faktas, brėžinys, medžiaga, pastaba, atkrovimo data, Kitas užsakovas / Kita objekto vieta |

Eilutė su užsakymu susieta per tekstinį stulpelį *Užs. Nr.* „Kitas užsakovas“ / „Kita objekto vieta“ pildomi tik tada, kai eilutės reikšmė skiriasi nuo užsakymo.

## Kas toliau (idėjos)

- Filtrai pagal brėžinį, medžiagą, vykdytoją, datas
- Operacijų stulpeliai (plazma, suvirinimas, …) ir eiga
- Užsakymo aplanko kūrimas (esamas Power Automate srautas)
- Eksportas į Excel
