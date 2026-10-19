# Stezka

Webová appka pro sledování denních kroků pro uzavřenou skupinu uživatelů. Každý má vlastní cíl (výchozí 4 000 000 kroků za 365 dní od registrace), zadává kroky i zpětně, vidí srovnání s ostatními a hlasuje o odměnách za splněné milníky.

Appka je jeden statický soubor (`index.html`) bez vlastního serveru — přihlašování a data zajišťuje Firebase (Authentication + Firestore). Dá se hostovat kdekoli (GitHub Pages, Netlify, Firebase Hosting…) nebo i jen otevřít lokálně v prohlížeči.

## Funkce

Boxy na dashboardu v aktuálním pořadí:

1. **Postup k cíli** — grafická stezka s milníky 25 %, 50 %, 75 % a CÍL, stav (na dobré cestě / pozadu / splněno), potřebný denní průměr, změna vlastního cíle a nastavení e-mailu pro obnovu hesla.
2. **Přidat kroky** — zadání kroků k libovolnému datu (i zpětně), historie záznamů s možností mazání. Pole Datum se předvyplňuje na první nezadaný den po posledním záznamu, nejvýš na dnešek (po uložení se posune na další den).
3. **Tento týden – Top 5** — pořadí podle absolutního počtu kroků od pondělí do neděle. U jména je datum posledního záznamu ("dnes", "včera" nebo např. "9. 7.").
4. **Týdenní vyhodnocení** — stupně vítězů za minulý týden (1.–3. místo) a tabulka 🥇🥈🥉 s počtem umístění za dřívější týdny. Minulý týden se do tabulky nepočítá, je vidět jen na stupních vítězů.
5. **Jak si stojíte vůči ostatním** — žebříček podle nachozených kroků, u procenta plnění je pod ním "z X tis." (cíl dané uživatelky v tisících).
6. **Tento měsíc – plnění cíle Top 5** — pořadí podle plnění poměrné části ročního cíle (cíl / 365 × počet dní v měsíci), také s "z X tis.".
7. **Historie** — kdo byl nejlepší v plnění cíle v jednotlivých měsících (jen měsíce, za které jsou data).
8. **Holky, čím se odměníme?** — návrhy odměn pro milníky 25/50/75/100 % (název a jméno navrhovatelky v závorce) a hlasování: jeden hlas na kategorii, lze změnit, návrhy s nejvíce hlasy mají 🏆.
9. **Nebezpečná zóna** — zrušení účtu a smazání všech svých dat (vyžaduje heslo).

Dále: registrace chráněná přístupovým kódem a seznamem schválených jmen, samoobslužná obnova zapomenutého hesla e-mailem, responzivní vzhled pro mobil.

## Jak appka funguje pod kapotou

Uživatel se v appce přihlašuje pouze jménem a heslem. Firebase Authentication ale interně vyžaduje e-mail, takže appka:

1. Při registraci uloží skutečný e-mail (zadaný jen kvůli obnově hesla) přímo jako přihlašovací e-mail účtu ve Firebase.
2. Zároveň si do kolekce `emailLookup` uloží pár *uživatelské jméno → e-mail*, aby si při přihlášení (nebo obnově hesla) mohla e-mail k zadanému jménu dohledat, aniž by uživatel musel e-mail znovu zadávat.

### Pravidla výpočtů

- **Týden** je pondělí až neděle. Všechny týdenní výpočty používají stejné pomocné funkce; práce s daty je řešená přes `setDate`, aby nevadil přechod letního a zimního času.
- **Týdenní vyhodnocení:** pořadí je podle součtu kroků za týden bez ohledu na cíl. Počítají se jen uzavřené týdny a jen lidé s nenulovými kroky. Pokud někdo doplní kroky zpětně, pořadí se při dalším načtení přepočítá.
- **Plnění cíle za měsíc:** kroky v měsíci / (cíl / 365 × počet dní v měsíci). Při shodě v historii se zobrazí obě jména.
- Žebříčky se počítají v prohlížeči z dat všech uživatelů, která se načtou při každém otevření appky.

## Datový model (Firestore)

| Kolekce | ID dokumentu | Obsah |
|---|---|---|
| `allowlist` | uživatelské jméno (lowercase) | prázdný dokument — jeho pouhá existence povoluje registraci pod tímto jménem |
| `emailLookup` | uživatelské jméno (lowercase) | `{ email, uid }` — pro dohledání e-mailu při přihlášení/resetu hesla |
| `users` | uid | `{ username, goal, registeredAt }` |
| `steps` | uid | mapa `{ "YYYY-MM-DD": počet_kroků, ... }` |
| `rewardProposals` | auto ID | `{ category ("25"/"50"/"75"/"100"), text, proposedBy, uid, createdAt }` |
| `rewardVotes` | `{uid}_{category}` | `{ category, proposalId, uid, username, updatedAt }` — jeden dokument na uživatele a kategorii, takže má každý jen jeden aktivní hlas na kategorii |

## Firestore Security Rules

Do Firestore → záložka **Rules** patří přesně tohle (nahraďte jimi celý obsah):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /allowlist/{username} {
      allow read: if request.auth != null;
      allow write: if false;
    }
    match /emailLookup/{username} {
      allow read: if true;
      allow write: if request.auth != null && request.resource.data.uid == request.auth.uid;
    }
    match /users/{uid} {
      allow read: if request.auth != null;
      allow create: if request.auth != null && request.auth.uid == uid
                    && exists(/databases/$(database)/documents/allowlist/$(request.resource.data.username));
      allow update, delete: if request.auth != null && request.auth.uid == uid;
    }
    match /steps/{uid} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == uid;
    }
    match /rewardProposals/{proposalId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null && request.resource.data.uid == request.auth.uid;
      allow update, delete: if false;
    }
    match /rewardVotes/{voteId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.resource.data.uid == request.auth.uid
                   && voteId == request.auth.uid + '_' + request.resource.data.category;
    }
  }
}
```

**Pozor na odsazení závorek:** bloky `rewardProposals` a `rewardVotes` musí být na stejné úrovni jako `steps`, ne vnořené do něj, jinak zápisy selhávají.

**Princip:** každý přihlášený uživatel může číst vše (potřeba kvůli žebříčkům a hlasování), ale zapisovat smí jen do svých vlastních dokumentů. Registrace nového účtu navíc vyžaduje, aby dané uživatelské jméno existovalo v `allowlist`.

## Administrace

### Přidání nového schváleného uživatele
Firestore → `allowlist` → **Add document** → ID dokumentu = uživatelské jméno malými písmeny (obsah může zůstat prázdný). Teprve potom se může daná osoba zaregistrovat.

### Přístupový kód pro registraci
V `index.html` je konstanta `INVITE_CODE`:
```js
const INVITE_CODE = "...";
```
Kód se mění přímo v souboru. Je vidět ve zdrojovém kódu stránky (repozitář je veřejný), takže funguje jen jako clona proti botům, ne jako skutečné zabezpečení. Skutečné omezení "jen tito lidé" hlídá `allowlist`. Samotný kód proto do dokumentace nepíšeme.

### Smazání testovacího/nechtěného účtu
Nejjednodušší cestou přes appku samotnou (přihlásit se → dole "Zrušit účet a smazat data"), nebo ručně: smazat uživatele v **Authentication → Users** a odpovídající dokumenty v `users`, `steps` a `emailLookup` ve Firestore.

## Nasazení / aktualizace

Repozitář je nastavený na **GitHub Pages**:
1. Upravte `index.html` (pracovní verze se jmenuje `stezka-firebase.html`, pro nahrání se kopíruje jako `index.html`, obsah je stejný).
2. V repozitáři na GitHubu: otevřít `index.html` → ikona tužky (Edit) → vložit nový obsah → **Commit changes**, nebo **Add file → Upload files**.
3. Stránka se aktualizuje automaticky přibližně do minuty na adrese `https://<uživatel>.github.io/<repozitář>/`.

Doména `<uživatel>.github.io` musí být ve Firebase v **Authentication → Settings → Authorized domains**, jinak nefunguje přihlašování. Firebase konfigurace (`firebaseConfig`) je přímo v souboru, při přesunu na jiný hosting se v něm nic nemění (jen je potřeba přidat novou doménu mezi povolené).

## Poznámky pro další úpravy

- **Responzivní CSS musí zůstat na konci `<style>`.** Pravidla v `@media` jsou záměrně úplně dole, jinak je přebijí základní pravidla stejných prvků a mobilní zobrazení se rozbije.
- **Po každé úpravě skriptu ověřit syntaxi**, například extrakcí obsahu `<script type="module">` do souboru a spuštěním `node --check`. Chybějící řádek (např. začátek funkce) způsobí prázdnou stránku a v konzoli hlášku "Illegal return statement".
- Vzhled: lesní zelená, písková a jantarová; písma Fraunces (nadpisy), IBM Plex Sans (text) a IBM Plex Mono (čísla). Jazyk rozhraní je čeština.

## Známá omezení

- **Veřejný repozitář** = konfigurace Firebase i přístupový kód jsou viditelné ve zdrojovém kódu. Konfigurace není citlivá (identifikuje jen projekt, chrání ji Firestore Rules), přístupový kód je jen měkká clona proti botům.
- **`emailLookup` je čitelná bez přihlášení** (nutné kvůli přihlášení/obnově hesla jen jménem) — kdokoli, kdo zná přesné uživatelské jméno, si tak může dohledat přiřazený e-mail. Pro uzavřenou důvěryhodnou skupinu je to přijatelný kompromis.
- **E-maily pro obnovu hesla zatím nedorážejí.** Nedošly ani při odeslání přímo z Firebase konzole, ani na osobní adresu u Seznam.cz, ani do složky Spam. Příčina není vyřešená. Možný další krok je nastavit vlastní SMTP v Authentication → Templates → SMTP settings. Hlášení o Dynamic Links v konzoli se appky netýká. Do vyřešení lze heslo resetovat ručně ve Firebase konzoli (Authentication → Users).
- Návrhy odměn zatím nejde po odeslání smazat ani upravit.
- Žebříčky při každém načtení čtou data všech uživatelů z Firestore — pro desítky uživatelů bez problému v rámci bezplatných limitů, při větším růstu by šlo optimalizovat průběžným počítáním součtů.

## Historie hlavních změn

- Přihlašování jménem a heslem, cloudové úložiště (Firebase)
- Vlastní cíl pro každého uživatele
- Zrušení účtu a smazání dat
- Srovnání podle týdne, měsíce a historie po měsících
- Ochrana registrace: přístupový kód + allowlist
- Samoobslužná obnova zapomenutého hesla
- Mobilní optimalizace
- Návrhy a hlasování o odměnách za milníky
- Datum posledního záznamu u jmen v boxu Tento týden, cíl v tisících v žebříčcích
- Předvyplňování data na první nezadaný den
- Týdenní vyhodnocení (stupně vítězů a tabulka umístění)
