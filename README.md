# Agrometeo Randwijk — nachtvorst & VD-voorspelling

Twee lichte voorspelmodellen in één offline browser-app, gebouwd op vijf jaar
weerstationdata van **FRC / Proeftuin Randwijk** (2021–2025):

1. **Nachtvorst** — voorspelt uit avondcondities (~19:00) de laagste temperatuur tot
   09:00 de volgende ochtend, plus de kans op nachtvorst.
2. **VD-piek** — voorspelt uit ochtendcondities (~08:00) de hoogste VD (vochtdeficit,
   in **g/m³**) tussen 10:00 en 18:00. Gezonde zone: 2–10 g/m³.
3. **Bladnat & schurftdruk** — schat de bladnatduur uit het weerbericht (nat bij RV ≥ 90 %
   of neerslag) en bepaalt met de Mills-tabel het infectierisico voor appelschurft. De
   bladnat-regel is gekalibreerd tegen de Davis-sensor (~81 % overeenkomst).
4. **Klopt het?** — een validatiegrafiek over de dag: het werkelijke verloop van temperatuur
   en VD naast de modelraming, zodat je ziet hoe dicht voorspeld en werkelijk bij elkaar liggen.

De app draait volledig in de browser, zonder backend. De modellen zijn lineair
(+ logistisch voor de vorstkans) en als coëfficiënten in `index.html` gebakken.

**De invoer komt automatisch uit het weersbericht** (Open-Meteo, gratis en zonder
API-sleutel). Bij het laden haalt de app de verwachting voor Randwijk op en vult de
velden voor de gekozen dag. Je kunt elke schuif nog handmatig bijstellen voor "wat-als",
een andere plaats invullen, of via het dagmenu een andere dag/nacht kiezen.

## Snel starten

Open `index.html` in een browser. Klaar. Geen build, geen server nodig. Het weersbericht
wordt automatisch geladen (internet nodig); zonder verbinding werkt de app handmatig.

## Weersbericht-koppeling (Open-Meteo)

De app roept twee open endpoints aan, rechtstreeks vanuit de browser:

- Geocoding: `https://geocoding-api.open-meteo.com/v1/search` (alleen bij een andere plaats)
- Verwachting: `https://api.open-meteo.com/v1/forecast` met `past_days=1&forecast_days=3`,
  uurlijks `temperature_2m, relative_humidity_2m, dew_point_2m, wind_speed_10m,
  precipitation, vapour_pressure_deficit` en dagelijks `temperature_2m_max, precipitation_sum`
  (tijdzone Europe/Amsterdam).

Mapping naar de modelinvoer:

| Model | Uit het weersbericht |
|-------|----------------------|
| Vorst | temp/RV/wind + **windrichting** & **regenintensiteit** om 19:00 · dagmax & neerslagsom · min-temp afgelopen nacht (gisteren 18:00 → gekozen dag 09:00) |
| VD | temp/RV/wind + **windrichting** & **regenintensiteit** om 08:00 · verwachte dagmax · VD-piek van gisteren (10:00–18:00) |

Standaardlocatie staat in `index.html` als `DEFAULT_PLACE` (Randwijk, 51.937 N, 5.719 O) —
pas die aan voor een andere vaste locatie.

> **Getoonde piek/dip = top/dal van de live uurverwachting.** Op de tabbladen Nachtvorst en
> VD-piek is de gemarkeerde waarde de laagste temperatuur, resp. de hoogste VD, uit de
> Open-Meteo-uurverwachting — de markering ligt dus exact op de curve. De getrainde modellen
> leveren de **vorstkans**, voeden de **validatietab** en de **what-if/ochtend-stand** (handmatig
> bijstellen van de schuiven of "alleen ochtend" gebruikt het lineaire model).

## Validatietab "Klopt het?"

Toont per dag een grafiek (00:00–23:00) met het **werkelijke verloop** van temperatuur en VD
(doorgetrokken = al gemeten, gestippeld = nog verwacht), met daarin als gestippelde lijnen de
**modelvoorspelling** van de nachtvorst-Tmin (van de afgelopen nacht) en de VD-piek. Eronder
twee kaarten met voorspeld vs werkelijk en de afwijking, kleurgecodeerd:

- groen = binnen de getoetste nauwkeurigheid (≤ 2 °C voor vorst, ≤ 1 g/m³ voor VD)
- oranje = matige afwijking · rood = grote afwijking

De "werkelijke" waarden komen uit de gerealiseerde Open-Meteo-analyse voor verstreken uren —
dat is niet rechtstreeks het Davis-station, maar wel de best beschikbare realisatie in de browser.
Validatie is compleet zodra de nacht (na 09:00) respectievelijk de middag (na 18:00) voorbij is.

### Deployen op Render (static site)

1. Push deze map naar een GitHub-repo.
2. Render → **New** → **Static Site** → koppel de repo.
3. Build command: *(leeg laten)* · Publish directory: `.`

Elke static-host (GitHub Pages, Netlify, Vercel) werkt net zo goed.

## Mapstructuur

```
randwijk-agrometeo/
├── index.html                 ← de app (vorst + VD, zelfstandig)
├── README.md
├── models/
│   ├── frost_model.json        ← lineair (Tmin) + logistisch (vorstkans)
│   ├── vd_model_tmax.json      ← VD-piek mét verwachte dagmax (aanbevolen)
│   └── vd_model_morning.json   ← VD-piek alleen ochtend
├── scripts/
│   ├── harmonize.py            ← 5 tabbladen → 1 nette tijdreeks
│   ├── train_frost.py          ← traint + valideert het vorstmodel
│   ├── train_vd.py             ← traint + valideert het VD-model
│   └── requirements.txt
└── data/
    ├── Randwijk_weer_2021-2025_schoon.csv   ← geharmoniseerd, 154.390 rijen
    ├── Randwijk_nachten_kenmerken.csv       ← nacht-dataset (1.579 nachten)
    └── Randwijk_dagen_VD_kenmerken.csv      ← dag-dataset (VD-piek)
```

## Hoe de modellen rekenen

Beide gebruiken gestandaardiseerde kenmerken:

```
z_i = (x_i − mean_i) / scale_i
y   = intercept + Σ coef_i · z_i
```

Voor de vorstkans gaat `y` (de logit) door een sigmoïde: `kans = 1 / (1 + e^−y)`.
Dauwpunt en VD worden in de app berekend met de Magnus-formule:

```
Td  = 243.12 · γ / (17.62 − γ),  γ = ln(RV/100) + 17.62·T/(243.12+T)
svp = 0.6108 · exp(17.27·T / (T+237.3))      (kPa)
VPD = svp(T) · (1 − RV/100)                   (kPa)
VD  = 2.16679 · (VPD·1000) / (T+273.15)       (g/m³, vochtdeficit)
```

De JSON-bestanden in `models/` bevatten `feats`, `mean`, `scale`, coëfficiënten en
intercept, zodat je de modellen ook elders kunt hergebruiken (bijv. in het TorenVD-dashboard).

## Bladnat & schurftdruk

Het weerbericht levert geen bladnatheid; die wordt geschat: **nat als RV ≥ 90 % of het regent**.
Die regel is gekalibreerd tegen de Davis-bladnatsensor uit de Randwijk-data (sensor "nat" ≈
leafwet ≥ 6) en komt daar ~81 % mee overeen (F1 ≈ 0,79). De app telt de **langste aaneengesloten
natperiode** (loopt door over middernacht) en de gemiddelde temperatuur daarbij. Via de (herziene)
**Mills-tabel** voor appelschurft (Venturia inaequalis) volgt dan het infectierisico: een
temperatuur-afhankelijke drempel in natte uren (bijv. ~9 u bij 16–24 °C, ~22 u bij 6 °C), met
classificatie geen / licht / matig / zwaar naarmate de natperiode die drempel overschrijdt.

> Dit is een **indicatieve** benadering — een signaal, geen vervanging van een officiële
> schurftwaarschuwing (bijv. RIMpro of de regionale waarschuwingsdienst).

## Prestaties (getoetst op 2025, volledig out-of-sample)

| Model | Maat | Resultaat |
|-------|------|-----------|
| Nachtvorst — minimumtemperatuur | MAE | 1,86 °C |
| Nachtvorst — vorstkans | ROC-AUC | 0,98 |
| VD-piek — alleen ochtend | MAE / R² | 1,9 g/m³ / 0,62 |
| VD-piek — mét verwachte dagmax | MAE / R² | 1,18 g/m³ / 0,86 |

Sterkste voorspellers: voor vorst het **avond-dauwpunt** (uitstralingsvorst), voor VD de
**verwachte dagmaximum-temperatuur** samen met het ochtend-dauwpunt. Windrichting (als sin/cos
gecodeerd) en regenintensiteit zijn als kenmerken toegevoegd; hun bijdrage is in deze data klein.

## Opnieuw trainen

```bash
cd scripts
pip install -r requirements.txt
python harmonize.py /pad/naar/Weerdata_FRCRandwijk_2021_tm_2025.xlsx
python train_frost.py
python train_vd.py
```

De trainscripts splitsen op tijd (train t/m 2024, test 2025) en printen de metrieken.
Modelcoëfficiënten exporteren ze naar `models/`; neem die JSON over in `index.html`
als je het model bijwerkt.

## Beperkingen

- Eén station, één hoogte. De vorstvoorspelling geldt voor **uitstralingsvorst**, niet
  voor aanvoer van koude lucht of veranderende bewolking gedurende de nacht.
- De brondata bevat geen straling/bewolking; het seizoenskenmerk vangt dat deels op.
- VD-richtbanden (te vochtig / gunstig / oplopend / stress) zijn algemene teeltindicaties —
  stem af op gewas en groeifase.
- Geen vervanging voor een officiële waarschuwing. In de bloeiperiode: aan de veilige kant blijven.

## Databron

Davis-weerstation, FRC / Proeftuin Randwijk. De vijf jaartabbladen verschilden in
kolomopbouw (39 / 20 / 33 / 12 / 20 kolommen) en tijdnotatie (24-uurs vs 12-uurs met
AM/PM); `harmonize.py` brengt alles terug tot één consistente tijdreeks.
