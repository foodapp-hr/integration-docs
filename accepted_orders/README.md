# FoodApp Partner API — Accepted Orders v2 i Fetch Menu

**Verzija dokumenta:** 1.3 (2026-11-14): §5 usklađen sa stvarnim v2 payloadom (`fulfill_at`, stavke, `tax_list`); kodovi razloga (§4.2); ID-jevi jela (§7, §9)
**Kompatibilnost:** GloriaFood Accepted Orders API v2 + Fetch Menu API v2
**Namjena:** partnerski sustavi (call centar, POS, ERP) koji zaprimaju online narudžbe restorana s FoodApp platforme
**Status:** u pripremi — specifikacija je konačna, endpointi se puštaju u rad po dogovoru s partnerom. Pristup dobiva partner po restoranu, uz ključeve koje izdaje FoodApp.

---

## 1. Ukratko

FoodApp isporučuje narudžbe restorana u vaš sustav na dva načina, isto kao GloriaFood:

| Način | Kada | Smjer |
|---|---|---|
| **Push** (preporučeno) | Centralizirani sustav s javnim HTTPS endpointom | FoodApp → vaš endpoint (webhook) |
| **Poll** | Lokalni sustav iza vatrozida, bez javnog endpointa | vaš sustav → FoodApp (`/pos/order/pop`) |

Uz to, **Fetch Menu** daje jelovnik restorana u GF v2 obliku, s ID-jevima koji se poklapaju s ID-jevima u narudžbama.

**Ako vaš sustav već prima GloriaFood Accepted Orders v2 JSON, ne morate mijenjati parser.** FoodApp šalje isti oblik poruke, iste nazive polja i iste tipove. Razlike su navedene u §9.

---

## 2. Ključevi i identifikatori

| Naziv | Tko ga izdaje | Gdje se koristi | Opseg |
|---|---|---|---|
| **Master key** | FoodApp (ili preuzet iz postojeće GF integracije) | `Authorization` header u **pushu** prema vama — po njemu provjeravate da je poruka od FoodAppa | po partneru |
| **Restaurant key** | FoodApp | polje `restaurant_key` u svakoj narudžbi; `Authorization` header za **poll** | po lokaciji restorana |
| **Restaurant token** | **Vi** (vaš ID restorana) | polje `restaurant_token` u svakoj narudžbi — vraćamo vam vaš vlastiti ID, ne trebate mapping tablicu | po lokaciji restorana |
| **Menu key** | FoodApp | `Authorization` header za **Fetch Menu** | po lokaciji restorana; **ne** daje pristup narudžbama |
| **Order API key** | FoodApp | `Authorization` header za **prihvat/odbijanje preko API-ja** (§4.2) | po lokaciji restorana; samo ako je prihvat preko API-ja uključen |

Ključevi se predaju isključivo zaštićenim kanalom (ne e-mailom u čistom tekstu). Rotacija na zahtjev, uz dogovoreni termin.

---

## 3. Push (webhook)

### 3.1 Zahtjev

```http
POST <vaš endpoint URL>
Content-Type: application/json; charset=utf-8
Authorization: <master key>
User-Agent: FoodApp-Push/2
X-FoodApp-Delivery-Id: <jedinstveni ID pokušaja isporuke>
```

- `Authorization` sadrži **sam ključ**, bez prefiksa `Bearer` (GF paritet).
- Tijelo je kolekcija (GF oblik):

```json
{
  "count": 1,
  "orders": [ { ...narudžba, vidi §5... } ]
}
```

- FoodApp šalje **jednu narudžbu po zahtjevu** (`count: 1`), ali vaš parser mora podržati niz — oblik je isti kao kod GF-a.
- Samo HTTPS (TLS 1.2+). HTTP endpointi se ne prihvaćaju.

### 3.2 Odgovor koji očekujemo

| Vaš odgovor | Značenje za FoodApp |
|---|---|
| bilo koji **2xx** unutar **15 s** | isporučeno — nema ponavljanja |
| 4xx / 5xx / timeout (> 15 s) / greška veze | nije isporučeno — ponavljamo (§3.3) |

Tijelo odgovora se ne čita. Odgovorite odmah nakon što narudžbu trajno spremite; obradu (ispis, dodjela vozača) radite asinkrono.

### 3.3 Ponavljanje (retry)

Neuspješna isporuka ponavlja se s rastućim razmakom: **1 min → 5 min → 30 min → svakih 60 min**, ukupno do **24 h** od prvog pokušaja. Nakon toga isporuka se označava neuspjelom i FoodApp operater dobiva upozorenje.

### 3.4 Duplikati — obavezna zaštita na vašoj strani

Ista narudžba u rijetkim slučajevima stigne **više puta** (npr. vaš 2xx nije stigao do nas unutar 15 s pa smo ponovili). Isto pravilo vrijedi i kod GF-a.

**Pravilo deduplikacije:** par (`id`, `status`) je jedinstven. Ako ste narudžbu s tim `id`-om i tim statusom već spremili — odgovorite 2xx i ne stvarajte novu.

Ako primate više statusa (§4), ista narudžba (isti `id`) stiže više puta s različitim `status` — to je promjena stanja, ne duplikat. Vrijedi najnoviji po `updated_at`.

### 3.5 Izvorišna IP adresa

Push dolazi s IPv4 adrese **`178.105.33.38`** (FoodApp produkcija). Ako koristite IP allowlist, dodajte je. Promjenu adrese najavljujemo najmanje 14 dana unaprijed.

---

## 4. Statusi narudžbe

Po defaultu šaljemo **samo `accepted`**. Ostale statuse uključujemo po dogovoru, po restoranu.

| `status` | Kada se šalje | Default |
|---|---|---|
| `accepted` | narudžba je prihvaćena (vidi §4.1) | ✅ |
| `pending` | narudžba je predana, čeka prihvat restorana | ❌ |
| `rejected` | restoran je odbio narudžbu | ❌ |
| `timed_out` | restoran nije odgovorio u roku, narudžba je istekla | ❌ |
| `canceled` | prihvaćena narudžba je naknadno otkazana | ❌ |

### 4.1 Tko prihvaća narudžbu

Prihvat se konfigurira po restoranu:

- **Automatski prihvat** (preporučeno kad odluku donosi vaš sustav/call centar): narudžba odmah po predaji prelazi u `accepted` s unaprijed dogovorenim vremenom isporuke i odmah vam se šalje. Kupac odmah dobiva potvrdu e-poštom. Ako je narudžba plaćena karticom, iznos se tereti odmah.
- **Prihvat u FoodApp kuhinjskoj aplikaciji**: restoran prihvaća na uređaju, upisuje vrijeme; tek tada stiže `accepted`.
- **Prihvat preko API-ja** (proširenje FoodAppa, GF ga nema): narudžba vam stiže kao `pending`, a vaš sustav je prihvaća s vremenom isporuke ili odbija pozivom na FoodApp (§4.2). Kupac dobiva vaše stvarno vrijeme ili obavijest o odbijanju. Kartica se tereti tek na prihvat, a na odbijanje se rezervacija automatski otpušta.

Kod automatskog prihvata nema povratnog kanala, isto kao kod GF-a. Ako odbijete narudžbu koja vam je stigla kao `accepted`, kupca obavještavate sami, a povrat kartičnog plaćanja radi restoran ili FoodApp operater. Prelazak s automatskog prihvata na prihvat preko API-ja je promjena postavke na našoj strani. Dogovaramo ga kad vaš sustav podrži §4.2.

### 4.2 Prihvat i odbijanje preko API-ja

Uključuje se po restoranu. Push tada obavezno uključuje status `pending`.

**Autentikacija:** `Authorization: <order API key>`. To je zaseban ključ po restoranu (§2) i ne miješa se s restaurant keyem ni master keyem.

**Prihvat**

```http
POST https://api.foodapp.hr/pos/order/{id}/accept
Authorization: <order API key>
Content-Type: application/json

{ "eta_minutes": 35 }
```

- `{id}` = `id` narudžbe iz pusha.
- `eta_minutes` (5 do 240) je za narudžbe „što prije" obavezan: za toliko minuta od prihvata narudžba mora biti isporučena (dostava) ili spremna (preuzimanje).
- Za zakazane narudžbe (`for_later: true`) šaljete prazno tijelo `{}`. Vrijeme je ono koje je kupac izabrao.

```json
{ "id": 100245, "status": "accepted", "accepted_at": "2026-11-13T14:41:09.000Z", "fulfill_at": "2026-11-13T15:16:09.000Z" }
```

**Odbijanje**

```http
POST https://api.foodapp.hr/pos/order/{id}/reject
Authorization: <order API key>
Content-Type: application/json

{ "reason": "restaurant_busy" }
```

| `reason` | Kupac vidi |
|---|---|
| `unspecified` | narudžba nije prihvaćena (bez razloga) |
| `restaurant_busy` | restoran je trenutno zauzet |
| `no_drivers` | svi dostavljači su zauzeti |
| `items_unavailable` | neke stavke nisu dostupne |
| `custom` | vaša poruka iz polja `message` (obavezno, najviše 200 znakova) |

```json
{ "id": 100245, "status": "rejected", "rejected_at": "2026-11-13T14:41:09.000Z" }
```

**Stanje narudžbe** (za usklađivanje nakon ispada ili sumnje)

```http
GET https://api.foodapp.hr/pos/order/{id}
Authorization: <order API key>
```

```json
{ "id": 100245, "status": "accepted", "accepted_at": "…", "rejected_at": null, "fulfill_at": "…", "updated_at": "…" }
```

**Pravila**

- **Rok za odluku:** ako narudžba nije prihvaćena ni odbijena unutar približno **4 minute i 45 sekundi** od predaje, ističe (`timed_out`). Nakon 2 minute restoran može dobiti upozoravajući telefonski poziv, ako je broj postavljen.
- **Ponavljanje je sigurno:** ponovljeni `accept` na već prihvaćenoj narudžbi vraća `200` s trenutnim stanjem i ništa ne mijenja (ni vrijeme). Isto vrijedi za ponovljeni `reject`. Kod mrežne greške slobodno ponovite isti zahtjev.
- **Prva odluka vrijedi:** restoran može narudžbu prihvatiti i u FoodApp kuhinjskoj aplikaciji, kao rezervu kad vaš sustav ne radi. Ako je odluka već donesena (tamo ili istekom roka), vaš poziv dobiva `409` s trenutnim statusom.
- **Potvrda pushom:** nakon vaše ili kuhinjske odluke stiže push s novim statusom (`accepted`/`rejected`/`timed_out`). Tako uvijek imate isti tok stanja bez obzira na to tko je odlučio.
- Otkazivanje nakon prihvata i promjena vremena nakon prihvata nisu podržani u ovoj verziji. Javite se telefonom.

| HTTP | `code` | Značenje |
|---|---|---|
| `200` | | uspjeh (i za ponovljeni zahtjev) |
| `400` | `VALIDATION` | neispravno tijelo (npr. nedostaje `eta_minutes`, `message` za `custom`) |
| `401` | `UNAUTHORIZED` | neispravan ili opozvan order API key |
| `403` | `ACCEPTANCE_MODE` | za ovaj restoran prihvat preko API-ja nije uključen |
| `404` | `ORDER_NOT_FOUND` | narudžba ne postoji ili ne pripada ovom restoranu |
| `409` | `ORDER_NOT_PENDING` | o narudžbi je već odlučeno; tijelo sadrži trenutni `status` |
| `429` | `RATE_LIMITED` | više od 120 zahtjeva u minuti; pričekajte `Retry-After` sekundi |
| `5xx` | | privremena greška; ponovite, pa provjerite stanje s `GET` |

Tijelo greške: `{ "code": "ORDER_NOT_PENDING", "message": "…", "status": "timed_out" }`.

---

## 5. Narudžba — polja

Oblik je GloriaFood Accepted Orders v2. Novac je `number` s najviše dvije decimale, valuta `EUR`, porez je **uključen u cijenu** (`GROSS`). Vremena su ISO 8601 u **UTC** (`2026-11-08T17:41:09.000Z`).

### 5.1 Narudžba

| Polje | Tip | Opis |
|---|---|---|
| `id` | integer | broj narudžbe na FoodAppu (isti broj vidi kupac); ključ deduplikacije |
| `api_version` | integer | uvijek `2` |
| `pos_system_id` | integer | identifikator FoodAppa kao izvora |
| `type` | string | `delivery` ili `pickup` |
| `status` | string | vidi §4 |
| `source` | string | `website` |
| `payment` | string | `CASH` (gotovina), `CARD` (kartica pri preuzimanju/dostavi), `ONLINE` (plaćeno online) |
| `used_payment_methods` | array | `[payment]` |
| `gateway_type` | string \| null | za `ONLINE`: pružatelj plaćanja; inače `null` |
| `gateway_transaction_id` | string \| null | za `ONLINE`: ID transakcije kod pružatelja; inače `null` |
| `accepted_at` | string | kada je narudžba prihvaćena |
| `fulfill_at` | string | kada narudžba mora biti isporučena (dostava) ili spremna (preuzimanje) |
| `updated_at` | string | zadnja promjena stanja |
| `for_later` | boolean | `true` = zakazana narudžba (kupac je izabrao termin) |
| `instructions` | string \| null | napomena kupca za narudžbu |
| `restaurant_id` | integer | ID restorana (§9) |
| `restaurant_name`, `restaurant_phone` | string | |
| `restaurant_street`, `restaurant_city`, `restaurant_zipcode` | string | adresa restorana |
| `restaurant_country` | string | `HR` |
| `restaurant_state` | string | prazno |
| `restaurant_latitude`, `restaurant_longitude` | string | |
| `restaurant_timezone` | string | `Europe/Zagreb` |
| `restaurant_key` | string | restaurant key (§2) |
| `restaurant_token` | string | vaš ID restorana (§2) |
| `currency` | string | `EUR` |
| `client_first_name`, `client_last_name` | string | ime i prezime kupca (zasebno) |
| `client_email`, `client_phone` | string | |
| `client_marketing_consent` | boolean | privola kupca za marketing |
| `client_address` | string \| null | puna adresa; `null` za `pickup` |
| `client_address_parts` | object \| null | `street`, `city`, opcionalno `zipcode`, `more_address` (napomena za dostavu: kat, interfon…); `null` za `pickup` |
| `latitude`, `longitude` | string \| null | lokacija dostave; `null` za `pickup` |
| `outside_delivery_area` | boolean \| null | `true` kad je restoran prihvatio dostavu izvan zona |
| `total_price` | number | ukupno za naplatu, sve uključeno |
| `sub_total_price` | number | zbroj stavki tipa `item`, bez dostave |
| `tax_type` | string | `GROSS` |
| `tax_name` | string | `PDV` |
| `tax_value` | number | ukupni porez sadržan u `total_price` |
| `tax_list` | array | razrada poreza po stopi: `{ "type": "item" \| "delivery_fee", "value", "rate" }` |
| `coupons` | array | primijenjene promocije |
| `items` | array | stavke (§5.2); **sve što utječe na cijenu je stavka** |

Ostala GF v2 polja (`persons`, `client_id`, `client_order_count`, `table_number`, `ready`, `client_language`, `billing_details`, `reference`, `delivery_zone_name` i druga) postoje radi kompatibilnosti s fiksnim vrijednostima ili `null`. Ne oslanjajte se na njih.

### 5.2 Stavka (`items[]`)

| Polje | Tip | Opis |
|---|---|---|
| `type` | string | `item` ili `delivery_fee` |
| `type_id` | integer \| null | za `item`: ID jela iz Fetch Menu (`item.id`); `null` za `delivery_fee` i za jelo koje je u međuvremenu obrisano iz jelovnika |
| `name` | string | naziv jela; za dostavu `DELIVERY_FEE` |
| `instructions` | string \| null | napomena kupca za jelo |
| `quantity` | integer | |
| `price` | number | jedinična cijena jela (s veličinom) |
| `total_item_price` | number | ukupno za stavku, uključujući opcije i količinu |
| `item_discount`, `cart_discount`, `cart_discount_rate` | number | popusti (trenutno `0`) |
| `parent_id` | integer \| null | `null` |
| `kitchen_internal_name` | string \| null | `null` |
| `options` | array | odabrane opcije (§5.3) |

Dostava, kad se naplaćuje, dolazi kao jedna stavka `type: "delivery_fee"` i jedan redak u `tax_list` s `type: "delivery_fee"`.

### 5.3 Opcija (`items[].options[]`)

| Polje | Tip | Opis |
|---|---|---|
| `type` | string | `size` (veličina/varijanta) ili `option` (modifikator) |
| `type_id` | integer \| null | ID veličine ili opcije iz Fetch Menu |
| `group_name` | string | naziv grupe; za veličinu `Veličina` |
| `name` | string | naziv opcije (npr. „Velika", „Dodatni sir") |
| `price` | number | jedinična cijena opcije, bez množenja s količinom stavke; za veličinu `0` (sadržana je u `price` stavke) |
| `quantity` | integer | |

### 5.4 Primjer (dostava, gotovina; iznosi ilustrativni, skraćeno)

```json
{
  "count": 1,
  "orders": [
    {
      "id": 100245,
      "api_version": 2,
      "type": "delivery",
      "status": "accepted",
      "source": "website",
      "payment": "CASH",
      "gateway_type": null,
      "gateway_transaction_id": null,
      "accepted_at": "2026-11-08T17:41:09.000Z",
      "fulfill_at": "2026-11-08T18:25:00.000Z",
      "updated_at": "2026-11-08T17:41:09.000Z",
      "for_later": false,
      "instructions": "Molim zvoniti dvaput",
      "restaurant_id": 1042,
      "restaurant_name": "Primjer Restoran",
      "restaurant_phone": "+38511234567",
      "restaurant_country": "HR",
      "restaurant_city": "Zagreb",
      "restaurant_zipcode": "10000",
      "restaurant_street": "Ilica 1",
      "restaurant_latitude": "45.8130",
      "restaurant_longitude": "15.9770",
      "restaurant_timezone": "Europe/Zagreb",
      "restaurant_key": "AbCdEfGhIjKlMnOpQr",
      "restaurant_token": "vas-id-restorana",
      "currency": "EUR",
      "client_first_name": "Ana",
      "client_last_name": "Horvat",
      "client_email": "ana@example.com",
      "client_phone": "+385911234567",
      "client_marketing_consent": false,
      "client_address": "Savska cesta 10, 10000 Zagreb",
      "client_address_parts": {
        "street": "Savska cesta 10",
        "city": "Zagreb",
        "zipcode": "10000",
        "more_address": "3. kat, interfon Horvat"
      },
      "latitude": "45.8040",
      "longitude": "15.9650",
      "outside_delivery_area": null,
      "total_price": 29.3,
      "sub_total_price": 25.8,
      "tax_type": "GROSS",
      "tax_name": "PDV",
      "tax_value": 3.91,
      "tax_list": [
        { "type": "item", "value": 2.65, "rate": 0.13 },
        { "type": "item", "value": 0.56, "rate": 0.25 },
        { "type": "delivery_fee", "value": 0.7, "rate": 0.25 }
      ],
      "coupons": [],
      "items": [
        {
          "type": "item",
          "type_id": 20117,
          "name": "Pizza Margherita",
          "instructions": null,
          "quantity": 2,
          "price": 11.5,
          "total_item_price": 23.0,
          "item_discount": 0,
          "cart_discount": 0,
          "cart_discount_rate": 0,
          "parent_id": null,
          "kitchen_internal_name": null,
          "options": [
            { "type": "size", "type_id": 30051, "group_name": "Veličina", "name": "Velika", "price": 0, "quantity": 1 }
          ]
        },
        {
          "type": "item",
          "type_id": 20140,
          "name": "Coca-Cola 0,5 l",
          "instructions": null,
          "quantity": 1,
          "price": 2.8,
          "total_item_price": 2.8,
          "item_discount": 0,
          "cart_discount": 0,
          "cart_discount_rate": 0,
          "parent_id": null,
          "kitchen_internal_name": null,
          "options": []
        },
        {
          "type": "delivery_fee",
          "type_id": null,
          "name": "DELIVERY_FEE",
          "instructions": null,
          "quantity": 1,
          "price": 3.5,
          "total_item_price": 3.5,
          "item_discount": 0,
          "cart_discount": 0,
          "cart_discount_rate": 0,
          "parent_id": null,
          "kitchen_internal_name": null,
          "options": []
        }
      ]
    }
  ]
}
```

---

## 6. Poll (lokalni sustav)

Za sustave bez javnog endpointa. Uključuje se po restoranu, umjesto pusha ili uz njega.

```http
POST https://api.foodapp.hr/pos/order/pop
Authorization: <restaurant key>
Glf-Api-Version: 2
Accept: application/json
```

- Odgovor: ista kolekcija kao kod pusha — `{ "count": N, "orders": [...] }`; `count: 0` kad nema novih.
- **Pop semantika:** vraćene narudžbe se smatraju preuzetima i ne vraćaju se ponovo. Nepreuzete narudžbe čekaju neograničeno.
- Interval: **60 s** preporučeno, najmanje 30 s. Češći upiti dobivaju `429`.
- Ako vam se odgovor izgubi (pad veze nakon što smo ga poslali), javite nam `id` — narudžbu vraćamo u red ručno.

| HTTP | Značenje |
|---|---|
| `200` | uspjeh (i kad je `count: 0`) |
| `401` | neispravan ili opozvan restaurant key |
| `429` | prečesto pozivanje — pričekajte `Retry-After` sekundi |
| `5xx` | privremena greška — ponovite u sljedećem intervalu |

---

## 7. Fetch Menu

```http
GET https://api.foodapp.hr/pos/menu
Authorization: <menu key>
Glf-Api-Version: 2
Accept: application/json
```

Odgovor je GF Fetch Menu v2 oblik:

| Razina | Polja |
|---|---|
| Jelovnik | `id`, `restaurant_id`, `currency`, `categories[]` |
| Kategorija | `id`, `name`, `description`, `sort`, `active`, `items[]`, `groups[]` |
| Jelo | `id`, `name`, `description`, `price`, `sort`, `active`, `is_out_of_stock`, `tags`, `sizes[]`, `groups[]`, `extras` |
| Veličina | `id`, `name`, `price` (**razlika** na cijenu jela, ne apsolutna), `default`, `groups[]` |
| Grupa opcija | `id`, `name`, `required`, `allow_quantity`, `force_min`, `force_max`, `options[]` |
| Opcija | `id`, `name`, `price`, `default`, `is_out_of_stock` |

- `item.id` = `items[].type_id` u narudžbi; `size.id` / `option.id` = `options[].type_id`.
- ID-jevi su FoodApp ID-jevi, **nisu** isti kao na GloriaFoodu. Ako mapirate po ID-u, nakon prelaska ponovo dohvatite jelovnik. Mapiranje po nazivu radi bez promjena.
- ID-jevi su **stabilni**: preimenovanje ili promjena cijene ne mijenja ID. Novo jelo dobiva novi ID; obrisano jelo nestaje iz odgovora, ali njegov ID se nikad ne dodjeljuje drugom jelu.
- Restoran sam uređuje jelovnik. Preporuka: dohvat jednom dnevno i na zahtjev; ne češće od svakih 5 minuta.

---

## 8. Testiranje i puštanje u rad

1. Pošaljite nam **testni URL** (može biti isti sustav, zasebna ruta) i potvrdite koje statuse želite (§4).
2. Šaljemo **testnu narudžbu** na testni URL. Testne narudžbe imaju header `X-FoodApp-Test: 1` i napomenu `TEST NARUDŽBA — NE PRIPREMATI`.
3. Potvrdite da je narudžba ispravno zaprimljena (restoran po `restaurant_token`, jela po `type_id` ili nazivu, iznosi, adresa).
4. Šaljemo istu testnu narudžbu ponovo — potvrdite da se **nije** stvorio duplikat (§3.4).
5. Dogovaramo termin prebacivanja; na dan prebacivanja produkcijski URL dobiva prave narudžbe.

---

## 9. Razlike u odnosu na GloriaFood v2

| Tema | GloriaFood | FoodApp |
|---|---|---|
| Oblik poruke, nazivi polja, tipovi | v2 | isto |
| `id` narudžbe | GF ID | FoodApp broj narudžbe (šesteroznamenkasti, ne preklapa se s GF ID-jevima) |
| `restaurant_id`, `restaurant_key`, `restaurant_token`, master key, menu key | GF vrijednosti | za restorane preseljene s GF-a: **iste vrijednosti** |
| ID-jevi jela/opcija (`type_id`, menu `id`) | GF ID-jevi | FoodApp ID-jevi (drugi brojevi); nakon prelaska ponovo dohvatite jelovnik ili mapirajte po nazivu |
| Polja `charge_to_room_*`, `company_account_id` | postoje | ne šalju se (hotelske/korporativne funkcije ne postoje) |
| Polje `restaurant_state` | ne postoji | postoji (dodatno, može se ignorirati) |
| Izvorišna IP adresa pusha | GF infrastruktura | `178.105.33.38` |
| Prihvat / odbijanje iz vašeg sustava | ne postoji | opcionalno, §4.2 |
| Poll endpoint | `pos.globalfoodsoft.com/pos/order/pop` | `api.foodapp.hr/pos/order/pop` |
| Fetch Menu endpoint | `pos.globalfoodsoft.com/pos/menu` | `api.foodapp.hr/pos/menu` |

---

## 10. Zaštita podataka

Narudžbe sadrže osobne podatke kupaca. Restoran je voditelj obrade; FoodApp ih isporučuje vašem sustavu po nalogu restorana. Podatke koristite samo za izvršenje narudžbe i čuvajte ih u skladu s vašim ugovorom s restoranom. Upiti o zaštiti podataka: **privatnost@foodapp.hr**.

---

## 11. Promjene API-ja

- `api_version: 2` ostaje stabilan: nova polja mogu se **dodati** bez najave; postojeća se ne mijenjaju niti uklanjaju (osim nedokumentiranih, §5.4).
- Nekompatibilna promjena = nova verzija, uz najavu najmanje **90 dana** i paralelni rad stare verzije.

---

## 12. Kontakt

Tehnička podrška za integraciju: **Ivan Milićević, FoodApp** — ivan@foodapp.hr · +385 91 444 1343

FoodApp, obrt za informatičke usluge, vl. Ivan Milićević, Županjska ulica 10, Zagreb
