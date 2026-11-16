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

Sąrašai kuriami iš importo failų `Z17_Uzsakymai.xlsx` ir `Z17_RMC_eilutes.xlsx` (failai ne repozitorijoje).

1. Įkelkite failus į SharePoint svetainės dokumentų biblioteką.
2. Svetainėje: **Naujas → Sąrašas → Iš „Excel“** → pasirinkite failą.
3. `Z17_Uzsakymai.xlsx` → sąrašas **`Z17_Uzsakymai`**. Visi stulpeliai – *Viena teksto eilutė*.
4. `Z17_RMC_eilutes.xlsx` → sąrašas **`Z17_RMC_eilutes`**. Tipai:
   - *Data ir laikas (tik data)*: Registravimo data, Užsakymo terminas, Atkrovimo data, Faktinis atkrovimas, TD pradėta, TD baigta
   - *Skaičius*: Kiekis, Sąmatinė vertė, Norma (val.), Faktas (val.)
   - *Kelios teksto eilutės*: Užsakymo turinys, Technologo pastabos
   - kiti – *Viena teksto eilutė* (RMC Nr. ir Užs. Nr. – būtinai tekstas)
5. `Z17_RMC_eilutes` → Sąrašo parametrai → **Indeksuoti stulpeliai** → indeksas stulpeliui *Užs. Nr.*
6. Komentarams – tuščias sąrašas **`Z17_Komentarai`** (Naujas → Sąrašas → Tuščias sąrašas) su stulpeliais:
   - `Title` – paliekamas (į jį rašomas Užs. Nr.)
   - **Eilutės ID** – *Skaičius* (tuščias = komentaras visam užsakymui)
   - **Komentaras** – *Kelios teksto eilutės*

   Autorius ir laikas imami iš SharePoint (Sukūrė / Sukurta). Jei sąrašo nėra, programa veikia, tik komentarai išjungti.

Stulpelių pavadinimų nekeiskite – programa jų ieško pagal šiuos pavadinimus (žr. `ORDER_FIELDS` / `LINE_FIELDS` / `COMMENT_FIELDS` faile `index.html`). Jei kurio nors neranda, viršuje parodo įspėjimą.

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
| `Z17_RMC_eilutes` | RMC eilutė | RMC Nr. (Title), Užs. Nr., datos, turinys, kiekis, vertė, PO Nr., vykdo, technologas, norma, faktas, brėžinys, medžiaga, pastaba, atkrovimo data, faktinis atkrovimas, technologo pastabos, QA (Ar reikia TD, TD rengėjas, TD pradėta, TD baigta, TD pastabos), Kitas užsakovas / Kita objekto vieta |
| `Z17_Komentarai` | komentaras | Title (Užs. Nr.), Eilutės ID, Komentaras + SharePoint autorius ir laikas |

Eilutė su užsakymu susieta per tekstinį stulpelį *Užs. Nr.* „Kitas užsakovas“ / „Kita objekto vieta“ pildomi tik tada, kai eilutės reikšmė skiriasi nuo užsakymo.

TD būsena skaičiuojama: *Ar reikia TD* = Reikia → Nepradėta / Rengiama (yra „TD pradėta“) / Baigta (yra „TD baigta“); Tikslinti; Nereikia.

## Rodiniai

- **Registras** – užsakymai ir RMC eilutės, filtrai, rikiavimas, komentarai
- **Suvestinė** – vertė, normos ir fakto valandos pagal vykdytojus, technologus, užsakovus, projektus; vėluojantys užsakymai pagal vertę; didžiausi laiko kaštai (paiso filtrų)
- **Dokumentacija** – QA: eilutės, kurioms reikia TD, pagal būseną ir rengėją

## Kas toliau (idėjos)

- Operacijų stulpeliai (plazma, suvirinimas, …) ir eiga
- Užsakymo aplanko kūrimas (esamas Power Automate srautas)
- Eksportas į Excel
