# Handover planbord

Laatst bijgewerkt: 6 augustus 2026. Live versie: **v3.98b** (beta).

## Waar we staan

Alles is gepusht naar `main` en staat live op https://dieteren-magazijn.github.io/taakverdeling/beta/

| Versie | Wat |
|---|---|
| v3.90b | Rol per toestel: PLAN-PC of database. Alleen een plan-pc mag de Excel koppelen en publiceren. Plus: geen schrijfactie meer na een mislukte databaselezing. |
| v3.91b | Iemand buiten zijn uren blijft zichtbaar in het bewerkscherm (gedimd, "weg 15:00"). TV-bord ongewijzigd. Nieuwe badge met de herkomst van de planning. |
| v3.92b | Uren-aftrek "-1.5" uit de Excel bleef niet behouden en sloeg om de paar seconden om. Opgelost. |
| v3.93b | Terugkomdatum verschijnt nu ook bij tijdskrediet (TK). Die stond enkel bij verlof en ziekte. |
| v3.94b | Balkje op de chip bij een halve dag: de gevulde helft toont of iemand voor- of namiddag werkt. Plus terugkomdatum bij TR en SYN (niet bij HT). `obClassify` bewaart nu de code in `_code`. |
| v3.95b | Halve dag omgezet naar een horizontale strook onderaan de chip (`.ob-hd`, `::after`) in plaats van het verticale staafje. Links geel is voormiddag, rechts geel is namiddag. |
| v3.97b | De aanduiding bleef op één toestel hangen. `syncWrite` bouwt het document op uit een vaste lijst velden en `obQual` stond daar niet bij, dus het werd nooit weggeschreven. Plus hertekenen bij ontvangst. Let op: een sleutel toevoegen vraagt vier ingrepen, niet drie (dirty-vlag, `collectLocal`, `syncRead` en het `doc` in `syncWrite`). |
| v3.96b | Brandweer en EHBO per persoon. Sleutel `vp_ob_qual` (genormaliseerde naam naar "B", "E" of "BE"), alleen de uitzonderingen. Symbool bij de naam, twee tegels met wie nu binnen zijn uren is, aanduiden via de knop Brandweer / EHBO achter Beheer. |
| v3.98b | Inbound-namen optioneel mee in de outbound-pool. Knop "Inbound" naast de avond-knop, standaard uit. Zie de aparte paragraaf hieronder. |

## Inbound-namen in de outbound-pool (v3.98b)

Standaard staat dit uit. De knop "📥 Inbound" zit in de outbound-knoppenbalk naast
"🌙 Avond" en is dus enkel zichtbaar met Beheer aan.

Hoe het werkt:

- Zet je hem aan, dan neemt hij de blokken over die op dat moment op het
  inbound-scherm aangevinkt staan (standaard enkel UITPAK). Die lijst wordt
  bewaard in `vp_ob_inb` en gaat mee in de sync.
- De sleutel bewaart bewust de **lijst blokken**, niet enkel "aan" of "uit".
  Reden: de blokfilter `vp_blocksel` is per toestel. Zou de schakelaar enkel
  "aan" zijn, dan stelde elk toestel een ander roster samen en gooide het
  toestel met minder blokken die namen uit de afdelingen. Nu stellen alle
  toestellen exact hetzelfde roster samen.
- Verander je de blokfilter terwijl de koppeling aan staat, dan volgt de lijst
  automatisch mee.
- Zet je hem weer uit, dan verdwijnen die mensen uit de pool maar blijft hun
  plaats in `vp_ob_place` bewaard. Zet je hem opnieuw aan, dan staan ze terug
  waar ze stonden.

Wat je op het scherm ziet:

- Aparte sectie "📥 Inbound" onderaan de pool, met een teller.
- Het teken **IN** in het paars op de chip (via `OB_STATUS.inbound`).
- Een geel waarschuwingsteken bij wie vandaag al op een inbound-taak staat, met
  de taaknaam in de tooltip. Dat blokkeert niets, het is enkel een signaal.

Twee dingen om te weten:

1. `obWindowFor` geeft inbound-mensen bewust **geen** automatisch urenvenster.
   Ze staan niet in de ploegkolommen van het blad, dus er is geen betrouwbaar
   standaarduur. Zonder venster vallen ze nooit ten onrechte als "buiten uren"
   van het bord. Stel je hun uren met de klok in, dan geldt dat venster wel.
2. Afwezigen uit de inbound-blokken komen niet mee. Die blijven op het
   inbound-scherm staan, waar ze thuishoren.

Nog open: de blokfilter `vp_blocksel` zelf synchroniseert nog altijd niet. Dat
is nu onschadelijk omdat de outbound-koppeling zijn eigen lijst bijhoudt, maar
het blijft een verschil tussen toestellen.

## Eerst controleren morgen

1. Iemand met `-1.5` in de namiddagkolom: eindigt die permanent op 19:30, met het label en de oranje markering, ook terwijl iemand anders zit te plannen? Dat was de bug van v3.92b.
2. Plan-pc: staat de badge in de bovenbalk groen op PLAN-PC, en toont `📂 Excel <datum>` de juiste bestandsdatum?
3. Gewoon toestel: staat er `📋 Planning <datum> · opgehaald <tijd>` in plaats van de amber melding?
4. Sectorkaarten: klopt de teller (aantal dat nu binnen zijn uren is, met een klein +n voor wie weg of nog niet begonnen is)?

## Als de planning niet lijkt bij te werken

De plan-pc leest de Excel elke 30 seconden opnieuw in, maar dat valt stil door:

- Een verborgen of geminimaliseerd browservenster (de browser bevriest de timers).
- Vervallen bestandstoestemming na een herstart van de browser. Zichtbaar aan het Plan-lampje: amber betekent klikken en toestemming geven.
- Een gat in de wachter: is de Excel in die sessie nog nooit gelezen, dan slaat hij altijd over. Eenmalig op 🔄 klikken lost dat op.
- OneDrive die op die pc niet synchroniseert.

## Openstaand, in volgorde van belang

**Groot, structureel**

1. Twee generaties van dezelfde module leven naast elkaar. `WL_blockWrite` en `WL_iHold` bestaan elk twee keer, en de volledige publiceermodule (`pubSchedule`, `pubRosterDay`, `pubApply`) ook. De laatste toewijzing wint, puur op volgorde in het bestand. Dit heeft al bijna een gat gemaakt: de rolgrendel moest in twee `push()`-functies. Opruimen van de dode v1-module is klein werk en controleerbaar af.
2. 184 van de 280 catch-blokken zijn leeg. Fouten worden ingeslikt, waardoor incidenten pas opvallen als er data weg is. Aanpakken op de tien plekken die ertoe doen: lezen, schrijven, publiceren, Excel inlezen, sjabloon toepassen.
3. Mensen worden op naam bijgehouden in vijf opslagsleutels, soms exact en soms genormaliseerd met `obNrm`. Een spellingsverschil tussen de twee Excels verbreekt de koppeling geruisloos. Eén sleutelfunctie plus een eenmalige omzetting.
4. `obApplyHours` muteert de brongegevens in plaats van er een weergave uit af te leiden. De bug van v3.92b was daar een symptoom van, geen uitzondering.

**Klein en concreet**

5. Excel-wachter: ook inlezen wanneer `__planTS` nog leeg is (ongeveer drie regels).
6. Zichtbare waarschuwing op de plan-pc wanneer de planning al uren niet meer ingelezen is.
7. Handmatig ingevuld einduur wordt op vrijdag twee uur vervroegd (`obWindowFor`). Die aftrek hoort alleen op de berekende standaard te slaan.
8. Het ⏰-venster overschrijft de sector-wissels en de terugkomdatum van dezelfde persoon in plaats van ze te laten staan (`obSetHours`).
9. `obComposeRoster` valt niet terug op de gepubliceerde planning bij een lege Excel-dag, want `[]` is truthy.
10. Validatie van een ingelezen Excel op de plan-pc: nul namen betekent weigeren in plaats van stil publiceren.
11. Slot tegen twee plan-pc's tegelijk. Nu wint de laatste.
12. Herstelknop voor de laatste vijf versies van het gedeelde document (die staan al in IndexedDB onder `vp_baks`, maar er is geen manier om ze terug te zetten).

**Al langer open**

13. STAP 2: het ingelezen Excel-roster volledig naar de database publiceren.
14. Privacy: rolgebaseerde toegang tot afwezigheidsdetails.
15. Roostercode uit de interim-Excel wordt ingelezen maar nergens gebruikt, waardoor avondmensen altijd op de berekende 21:00 uitkomen.

## Werkwijze

Staat in `CLAUDE.md`. Kort: bewerk `beta/index.html`, valideer de inline scripts (moet "fouten: 0" geven), bump de versie op drie plaatsen, commit in het Nederlands, push naar `main`, en controleer `beta/version.txt` live.

Let op: gebruik geen `sed` op `beta/index.html`. Dat stript alle CR-tekens en zet het hele bestand op LF, wat een diff van 5800 regels geeft. Bewerk met een editor die de regeleindes respecteert.

## Een nieuwe sessie starten

`CLAUDE.md` wordt automatisch ingelezen, dus de projectcontext is er meteen. Verwijs naar dit bestand en noem het punt waaraan je wil werken, bijvoorbeeld: "lees HANDOVER.md, we pakken punt 1 en 2 op". De inhoud van het gesprek van vandaag is er morgen niet meer, dus alles wat moet blijven staat in deze twee bestanden.
