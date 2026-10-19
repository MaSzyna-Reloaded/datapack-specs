# Modele dynamiczne — profil glTF

Status: projekt specyfikacji 0.3.0. Format bazowy: glTF 2.0.

## 1. Zakres i część wspólna

Profil opisuje pojazd lub jego wyposażenie jako instancję powiązaną z symulacją. Geometria i standardowe efekty są tworzone w narzędziu autorskim. Metadane opisują, jak symulacja odnajduje części i steruje ich prezentacją.

Obowiązują sekcje 2–11 [profilu statycznego](STATIC_MODEL_FORMAT.md): kontener, układ, materiały, światła, identyfikatory, granice, georeferencja, źródła danych, klipy, kanały, widoczność i cząstki. Niniejszy dokument rozszerza te reguły. MUST, SHOULD i MAY mają znaczenie zdefiniowane w profilu wspólnym.

| Wspólne dla obu klas | Właściwe instancji pojazdu |
|---|---|
| Geometria, PBR, alpha i warianty materiałów | Powiązanie z pojazdem symulowanym |
| Światła `point`, `spot`, `directional` | Sygnały czoła/końca, reflektory i przygaszenie |
| Klipy, skinning i morph targets | Próbowanie klipów stanem drzwi, nastawnika, wycieraczek |
| Widoczność i emisja | Warianty sprzęgów, przewodów i wyposażenia |
| Dym, para i inne cząstki | Zależność emisji od pracy silnika |
| Granice i pochodzenie danych | Obwiednia ruchu mechanizmów i pantografów |
| Stabilne ID i kanały wejściowe | Osie, wózki, kabiny, punkty styku i zaczepy |

Nagłówek używa `model_class: "dynamic"`. `model_type` przyjmuje `vehicle`, `cab` albo `attachment`. Umieszczenie instancji aktualizuje symulacja. Georeferencja może dokumentować obiekt źródłowy pomiaru.

## 2. Nagłówek i osie pojazdu

Przykład obiektu `asset`:

```json
{
  "version": "2.0",
  "extras": {
    "model_format_version": "0.3.0",
    "model_class": "dynamic",
    "model_type": "vehicle",
    "model_id": "vehicle.example.locomotive",
    "bounds": {
      "min": [-1.5, 0.0, -9.0],
      "max": [1.5, 4.5, 9.0],
      "size": [3.0, 4.5, 18.0]
    },
    "motion_bounds": {
      "min": [-1.7, -0.1, -9.2],
      "max": [1.7, 6.5, 9.2],
      "size": [3.4, 6.6, 18.4]
    }
  }
}
```

Przód pojazdu jest w +Z, góra w +Y, lewa strona w +X. Lewa/prawa strona i przód/tył odnoszą się do konstrukcji modelu, zachowując znaczenie podczas zmiany kierunku jazdy. Zalecany początek układu leży na osi podłużnej, na poziomie główek szyn i w środku długości konstrukcyjnej.

Każdy niezależnie poruszany element ma node o pivocie w miejscu jego obrotu albo przesunięcia. Sztywne części mogą być zwykłymi node'ami; deformowane przewody, mieszki lub postacie mogą korzystać ze skinów.

`bounds` opisuje pozycję spoczynkową z zamkniętymi drzwiami i opuszczonymi pantografami. `motion_bounds` obejmuje wszystkie dopuszczone pozy, wysunięcia i odchylenia. Dekoracyjne cząstki oraz zasięgi światła mają własne obwiednie efektów.

## 3. Identyfikatory i budowa modelu

`semantic_id` jest zarówno stabilnym adresem, jak i nazwą funkcji elementu. Konwencje poniżej tworzą katalog rozpoznawany przez integrację pojazdu. Nazwa node'a w Blenderze może być dowolnym opisem dla autora.

| Wzorzec `semantic_id` | Znaczenie |
|---|---|
| `vehicle.body` | Pudło / główna konstrukcja |
| `vehicle.bogie.<n>` | Wózek |
| `vehicle.axle.<n>` | Zestaw kołowy |
| `vehicle.wheel.<n>.left` / `.right` | Osobne wizualne koła zestawu |
| `vehicle.door.left.<n>` / `.right.<n>` | Zespół drzwi |
| `vehicle.doorstep.left.<n>` / `.right.<n>` | Stopień |
| `vehicle.headlight.front.left` / `.right` / `.upper` | Reflektor przedni |
| `vehicle.headlight.rear.left` / `.right` / `.upper` | Reflektor tylny |
| `vehicle.high_beam.<end>.<side>` | Światło drogowe |
| `vehicle.marker.<end>.<side>` | Światło oznaczenia końca |
| `vehicle.auxiliary_light.<end>.<side>` | Światło pomocnicze |
| `vehicle.coupler.front` / `.rear` | Sprzęg |
| `vehicle.hose.<end>.<system>.<side>` | Przewód pneumatyczny |
| `vehicle.control_cable.<end>` | Przewód sterowania |
| `vehicle.gangway.<end>` | Przejście międzywagonowe |
| `vehicle.buffer.<end>.<side>` | Zderzak |
| `vehicle.pantograph.<n>` | Zespół pantografu |
| `vehicle.cab.<n>` | Zespół kabiny |
| `vehicle.compartment.<n>` | Przedział |
| `vehicle.corridor.<n>` | Korytarz |
| `vehicle.load.<n>` | Ładunek |
| `vehicle.exhaust.<n>` | Emiter spalin |
| `vehicle.driver.<n>` | Postać maszynisty |
| `vehicle.mirror.<end>.<side>` | Lustro |
| `vehicle.wiper.<n>` | Mechanizm wycieraczki |
| `vehicle.cooling_shutter.<n>` | Żaluzja chłodzenia |

`end` przyjmuje `front` lub `rear`, `side` przyjmuje `left`, `right` albo `center`; system przewodu identyfikuje np. `brake` lub `supply`. Indeksy `<n>` są liczone od zera, według kolejności od przodu ku tyłowi. Wzorce są notacją dokumentu, a eksporter zapisuje konkretne identyfikatory.

Zespół może mieć identyfikatory podrzędnych funkcji, np. `vehicle.headlight.front.left.source`, `.lens`, `.dimmed` albo `vehicle.pantograph.0.lower_arm`. Referencja wskazuje dokładną wartość ID. Dołączenie modelu `attachment` tworzy oddzielną przestrzeń ID instancji; konfiguracja mocowania podaje instancję właściciela i jej node docelowy.

Przykładowa hierarchia:

```text
vehicle.root
├── vehicle.body
│   ├── vehicle.cab.0
│   ├── vehicle.door.left.0
│   ├── vehicle.headlight.front.left
│   │   ├── vehicle.headlight.front.left.source
│   │   └── vehicle.headlight.front.left.lens
│   └── vehicle.pantograph.0
├── vehicle.bogie.0
│   ├── vehicle.axle.0
│   └── vehicle.axle.1
└── vehicle.bogie.1
    ├── vehicle.axle.2
    └── vehicle.axle.3
```

Faktyczne rodzicielstwo wynika z potrzeb transformacji. Zależności mechaniczne dodatkowo wskazują jawne referencje w sterownikach.

## 4. Kontrakt kanałów symulacji

Kanały są wartościami wejściowymi konkretnej instancji. Nazwy w tym dokumencie stanowią kontrakt integracji, który adapter fizyki i scenariusza MUST dostarczyć dla użytych funkcji.

| Kanał | Typ / jednostka | Znaczenie |
|---|---|---|
| `door.left.<n>.open_fraction` | number [0,1] | Rzeczywiste otwarcie drzwi |
| `doorstep.left.<n>.extension` | number [0,1] | Rzeczywiste wysunięcie stopnia |
| `axle.<n>.angle` | number, rad | Kąt obrotu zestawu |
| `bogie.<n>.yaw` | number, rad | Skręt wózka względem rodzica |
| `body.heave` | number, m | Pionowe przesunięcie pudła |
| `headlight.<end>.<side>.level` | number [0,1] | Poziom oświetlenia |
| `headlight.<end>.<side>.mode` | string | `off`, `dimmed`, `full` |
| `coupler.<end>.visual` | string | `disconnected`, `connected_owner`, `connected_peer`, `adapter` |
| `hose.<end>.<system>.<side>.visual` | string | `disconnected`, `straight`, `slanted` |
| `buffer.<end>.<side>.compression` | number, m | Ściśnięcie zderzaka |
| `pantograph.<n>.contact_height` | number, m | Zadana wysokość punktu kontaktu w układzie assetu |
| `pantograph.<n>.raised` | bool | Stan podniesienia |
| `engine.exhaust.<n>.output` | number [0,1] | Intensywność emisji spalin |
| `wiper.<n>.phase` | number [0,1] | Faza cyklu wycieraczki |
| `cab.<n>.occupied` | bool | Zajęcie kabiny |
| `display.destination.text` | string | Tekst kierunku |

Analogiczne kanały drzwi i stopni po prawej stronie używają `right`. Kanały nowych urządzeń MAY być dodawane w konfiguracji instancji z udokumentowanym typem i jednostką. Wszystkie użyte kanały MUST zostać rozwiązane. Kanał opisujący faktyczne otwarcie pochodzi z symulowanego mechanizmu i jego ograniczeń.

Wartości domyślne sterowników służą podglądowi i inicjalizacji. Stan pojazdu i stan wizualny są odrębne dla każdej instancji, również przy współdzielonej geometrii i materiałach.

## 5. Klipy: drzwi, stopnie, wycieraczki, kontrolki

Animacja glTF przechowuje ruch przygotowany przez autora. `animation.extras.playback` z profilu wspólnego wiąże ją z kanałem. Przykład extras klipu drzwi:

```json
{
  "semantic_id": "vehicle.door.left.0.opening",
  "playback": {
    "mode": "position",
    "source": "door.left.0.open_fraction",
    "input": [0.0, 1.0],
    "time": [0.0, 3.0],
    "default": 0.0
  }
}
```

Próbka przy 0 przedstawia drzwi zamknięte, a przy 1 otwarte. Jeden klip może poruszać kilka skrzydeł i cięgien. Prędkość, blokady i zatrzymanie drzwi ustala symulacja przez przebieg kanału.

Stopnie, lustra, żaluzje i nastawniki korzystają z tej samej reguły. Wycieraczka może mieć klip pełnego cyklu sterowany `wiper.<n>.phase`; początek i koniec mają tę samą pozę. Krzywki i przekładnie przygotowane w Blenderze są wypiekane do klipu.

Klipy animujące światło, materiał lub widoczność używają [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer). Wspólny klip może jednocześnie zmieniać emisję soczewki i natężenie światła. W konfiguracji każda właściwość ma jednego aktywnego autora: klip albo odpowiedni sterownik.

## 6. Ruch proceduralny: osie, wózki i zawieszenie

Ruchy wyznaczane bezpośrednio przez symulację opisuje `node.extras.rotation_control` lub `translation_control`. Przykład zestawu kołowego:

```json
{
  "semantic_id": "vehicle.axle.0",
  "rotation_control": {
    "source": "axle.0.angle",
    "axis": [1.0, 0.0, 0.0],
    "scale": 1.0,
    "offset": 0.0,
    "default": 0.0
  }
}
```

`source` i jednostkowa lokalna `axis` są wymagane. `scale` domyślnie 1, `offset` i `default` domyślnie 0. Kąt `θ = source*scale + offset`. Wynikowa rotacja lokalna to `q_rest * angleAxis(θ, axis)`, z kątem w radianach i regułą prawej dłoni. Opcjonalne `limits: [min,max]` ogranicza θ po przeliczeniu. Kąt koła może narastać ciągle; renderer redukuje go modulo 2π przy budowie kwaternionu.

Symulacja wyznacza kąt koła z prędkości obrotowej albo drogi i promienia toczenia, uwzględniając poślizg. Jednostki i znak kanału są częścią konfiguracji osi.

Dla wózka stosuje się `rotation_control` z osią [0,1,0] i kanałem `bogie.<n>.yaw`. Zestawy są potomkami wózka, dzięki czemu dziedziczą jego ruch.

`translation_control` ma te same pola. Wynik to `t_rest + axis*(source*scale+offset)` w metrach. Służy np. pionowemu ruchowi pudła, skokowi zawieszenia i ściskaniu zderzaka. Dla złożenia niezależnych obrotów wokół kilku osi autor tworzy kolejne node'y pivotów z osobnymi kanałami.

Skala node'ów sterowanych i ich przodków MUST być jednostkowa po eksportowym wypieczeniu. Pivoty, offsety i geometria definiują fizyczne położenie mechanizmu w modelu.

## 7. Światła pojazdu i przygaszenie

Reflektor jest node'em ze standardowym światłem `KHR_lights_punctual`; jego soczewka lub odbłyśnik jest geometrią z materiałem. Oba mogą być potomkami zespołu o wspólnym ID. Autor może przygotować ten zestaw bezpośrednio w Blenderze.

Przykład extras node'a źródła:

```json
{
  "semantic_id": "vehicle.headlight.front.left.source",
  "light_control": {
    "source": "headlight.front.left.level",
    "default": 0.0
  }
}
```

`light_control` mnoży intensywność zapisaną przez eksportera w definicji światła. Przygaszenie 0.2 daje 20% tej wartości. Materiał soczewki może używać `emission_control` tego samego kanału. Rozkład fotometryczny, barwa i stożek należą do standardowego opisu światła.

Przy odrębnej geometrii przygaszonej lampy jej node ma standardowe `KHR_node_visibility` oraz:

```json
{
  "semantic_id": "vehicle.headlight.front.left.dimmed",
  "visibility_control": {
    "source": "headlight.front.left.mode",
    "equals": "dimmed"
  }
}
```

Analogicznie wariant pełny porównuje z `full`, a zgaszony z `off`. Ta sama zasada obejmuje kontrolki kabiny. Stan domyślny zapisuje `visible` w rozszerzeniu node'a.

Osobna geometria aureoli może być billboardem z materiałem emissive/unlit stosownie do wyglądu. `node.extras.billboard` przyjmuje `"camera"` albo `"vertical"`: odpowiednio ustawia lokalny +Z ku kamerze, z górą kamery, albo obraca tylko wokół światowej osi +Y. Pozycja pochodzi z hierarchii. Billboard jest autorem rotacji tego node'a. Materiał określa alpha; widoczność może korzystać z tego samego kanału lampy.

Właściwości rendererowe, takie jak rozdzielczość map cieni i bias, są dobierane przez renderer do jakości i skali sceny.

## 8. Sprzęgi, przewody i punkty mocowania

Node zespołu sprzęgu ma `semantic_id: vehicle.coupler.front` albo `.rear`. Geometrie wariantów używają potomnych ID i `visibility_control` z kanałem `coupler.<end>.visual`. Przykład wariantu:

```json
{
  "semantic_id": "vehicle.coupler.front.connected_peer",
  "visibility_control": {
    "source": "coupler.front.visual",
    "equals": "connected_peer"
  }
}
```

Adapter jest odrębną wartością kanału `adapter`; autor może przypisać mu ten sam mesh co innemu wariantowi, zachowując oddzielne node'y lub przygotować klip widoczności.

Przewody mają kanały `hose.<end>.<system>.<side>.visual`: `straight`, `slanted`, `disconnected`. Położenie pivotów i końcówek wynika z geometrii. Warianty geometrii mogą być zastąpione deformacją skinu sterowaną mechanizmem, gdy model zawiera taki rig.

Punkty mocowania są node'ami transformacyjnymi z ID, np. `vehicle.coupler.front.anchor`, `vehicle.hose.front.brake.left.anchor`, `vehicle.pantograph.0.contact`. Lokalny +Z zaczepu wskazuje kierunek na zewnątrz, +Y jego górę. Instancja dołączana do zaczepu mnoży jego transform światowy przez swój transform lokalny.

## 9. Pantograf: geometria, ruch i kontakt

Pantograf ma node zespołu, części ruchome oraz node punktu kontaktu. Autor wybiera jeden z dwóch kontraktów mechanizmu: wypieczoną kinematykę sterowaną wysokością albo runtime IK. Wybór jest zapisany przez obecność odpowiedniego opisu sterowania.

### 9.1. Kinematyka przygotowana przez autora

To zalecana reprezentacja pantografów ze złożonymi cięgnami, równoległobokiem lub kilkoma ramionami. Autor rozwiązuje więzy w narzędziu, eksportuje klip wszystkich członów, a następnie wiąże wysokość punktu kontaktu z czasem klipu.

`node.extras.pantograph` na node'ie zespołu:

```json
{
  "semantic_id": "vehicle.pantograph.0",
  "pantograph": {
    "animation": "vehicle.pantograph.0.motion",
    "contact_node": "vehicle.pantograph.0.contact",
    "height_source": "pantograph.0.contact_height",
    "raised_source": "pantograph.0.raised",
    "rest_time": 0.0,
    "height_to_time": [
      [4.2, 0.2],
      [5.0, 0.8],
      [6.0, 1.7],
      [6.5, 2.0]
    ],
    "tolerance": 0.005
  }
}
```

Wszystkie pola są wymagane. Wiersze `height_to_time` to `[wysokość kontaktu Y w metrach układu assetu, czas klipu w sekundach]`. Obie kolumny MUST rosnąć ściśle. Minimum dwa wiersze. Runtime interpoluje czas liniowo między sąsiednimi wierszami i próbkuje klip. Poza zakresem używa najbliższej granicy. Przy `raised_source: false` próbkuje `rest_time`.

Kanał wysokości dostarcza punkt docelowy już przeliczony do układu assetu, z uwzględnieniem pozy i przechyłu pojazdu. Eksporter generuje dostatecznie gęstą tablicę, aby wysokość `contact_node` po interpolacji różniła się od zadanej najwyżej o `tolerance` w całym zakresie. Walidator sprawdza tę własność próbkowaniem i adaptacyjnym zagęszczaniem.

Klip sterowany przez `pantograph` jest własnością tego mechanizmu. Stan kontaktu z siecią i przewodzenie prądu wylicza symulacja. Po osiągnięciu granicy mechanicznej raportuje osiągalną pozycję kontaktu do symulacji; renderer pokazuje pozycję graniczną. Autor może dołączyć osobny obrót ślizgacza sterowany przechyłem kontaktu przez dedykowany node.

### 9.2. Runtime IK — łańcuch dwuczłonowy

Profil definiuje `node.extras.ik` dla dwóch sztywnych członów. Służy mechanizmom o odpowiedniej strukturze i rigom pomocniczym. Kości/node'y, skiny i ich pozy spoczynkowe są zapisane standardowo w glTF.

```json
{
  "ik": {
    "solver": "two_bone",
    "root": "vehicle.pantograph.0.lower_arm",
    "middle": "vehicle.pantograph.0.upper_arm",
    "tip": "vehicle.pantograph.0.contact",
    "target_source": "pantograph.0.target",
    "pole_source": "pantograph.0.pole",
    "enabled_source": "pantograph.0.raised",
    "tolerance": 0.001
  }
}
```

Wszystkie pola są wymagane. `middle` jest bezpośrednim potomkiem `root`, a `tip` potomkiem `middle`. Długości członów są dodatnimi odległościami między początkami node'ów w pozycji spoczynkowej. Ich skale i skale przodków są jednostkowe.

`target_source` i `pole_source` dostarczają vec3 punktów w układzie assetu. Integracja definiuje te kanały; `pole` wskazuje stronę zgięcia. `enabled_source` jest bool. Przy false obowiązuje poza spoczynkowa. Solver zapisuje rotacje `root` i `middle`, zachowując długości i pozycję początku łańcucha.

Odległość celu jest ograniczana do osiągalnego przedziału `[abs(L1-L2)+ε, L1+L2-ε]`, gdzie `ε = min(tolerance, 0.001*min(L1,L2))`. Dla celu pokrywającego się z początkiem łańcucha kierunek pochodzi z wektora root→tip pozy spoczynkowej. Rozwiązanie używa twierdzenia cosinusów w płaszczyźnie wyznaczonej przez kierunek celu i pole. Przy współliniowym pole wykorzystuje płaszczyznę zgięcia pozy spoczynkowej; autor MUST zapewnić jej określoność. Wybrana gałąź rozwiązania zachowuje stronę pole. Twist członów wynika z minimalnego obrotu ich osi spoczynkowych do osi rozwiązania.

Rotacja lokalna `tip` pozostaje spoczynkowa; osobny node ślizgacza może otrzymać sterowanie orientacją. Solver raportuje błąd osiągnięcia celu jako odległość tip–target, aby symulacja rozstrzygnęła kontakt. `tolerance` jest dodatnie i określa dopuszczalny błąd dla osiągalnego celu.

Eksporter MUST sprawdzić zgodność geometrii mechanizmu z wybranym solverem. Mechanizmy wieloczłonowe ze sprzężonymi więzami korzystają z kontraktu 9.1, który zapisuje pełną kinematykę autora.

## 10. Kabiny, kamery, wyświetlacze i ładunki

Kabina i przedziały są node'ami zespołów z odpowiednimi ID. Elementy ograniczone do wybranych widoków mogą mieć `node.extras.view_filter`, tablicę wartości `external`, `cab`, `passenger`, `reflection`, `shadow`, `editor`. Pominięcie oznacza wszystkie widoki. Filtr obowiązuje dla poddrzewa; wynik jest przecięciem filtrów przodków. Stan widoku dostarcza renderer dla konkretnego przebiegu.

Kamera stanowiska jest standardową kamerą glTF na node'ie, np. `vehicle.cab.0.camera`. Jej transform definiuje pozycję i orientację autora; symulator dobiera aktywną kamerę przez ID.

Wyświetlacz ma node z geometrią i materiałem. `node.extras.display` wiąże zawartość z kanałem:

```json
{
  "semantic_id": "vehicle.display.destination",
  "display": {
    "source": "display.destination.text",
    "material": "vehicle.display.destination.surface",
    "resolution": [1024, 128],
    "font": "fonts/destination.ttf",
    "font_size": 96,
    "color": [1.0, 0.7, 0.0, 1.0],
    "background": [0.0, 0.0, 0.0, 1.0]
  }
}
```

Wszystkie pola są wymagane. Tekst Unicode jest rasteryzowany do obrazu sRGB o dodatniej całkowitej rozdzielczości; `font_size` w pikselach. Kolory RGBA mają składowe [0,1], RGB sRGB. Tekst jest wyśrodkowany, pojedynczy wiersz, przy nadmiarze szerokości przycinany. Font jest zasobem pakietu. Wynik zastępuje obraz `baseColorTexture` wskazanego materiału w tej instancji; materiał MUST mieć tę teksturę, a geometria właściwe UV. Autor ustala emisję i pozostałe właściwości materiału. Początkowy obraz tekstury stanowi podgląd eksportu.

Cyfry mechaniczne i wskazówki korzystają z klipów `position`. Widoczność ładunku lub postaci może być związana z kanałem przez `visibility_control`; deformacja i ruch korzystają z klipów, morphów albo skinu.

## 11. Spaliny i para

Emiter `vehicle.exhaust.<n>` używa `smoke_source` z profilu wspólnego. Kanał `engine.exhaust.<n>.output` moduluje emisję, a `inherit_velocity` określa udział prędkości emitera przekazywany cząstkom przy narodzinach.

Przykład konfiguracji emitera w `node.extras`:

```json
{
  "semantic_id": "vehicle.exhaust.0",
  "smoke_source": {
    "rate": 30.0,
    "max_particles": 600,
    "lifetime": [2.0, 5.0],
    "speed": [1.0, 2.0],
    "size": [0.2, 0.4],
    "cone_angle": 0.15,
    "growth": 0.3,
    "acceleration": [0.0, 0.2, 0.0],
    "drag": 0.15,
    "inherit_velocity": 1.0,
    "material": "effects.diesel_smoke",
    "source": "engine.exhaust.0.output",
    "default": 0.0
  }
}
```

Pozycja i kierunek emisji dziedziczą ruch pudła. Prędkość emitera obejmuje ruch postępowy pojazdu i ruch punktu wynikający z obrotu. Kolor i alpha pochodzą z materiału; zmiany wynikające z obciążenia mogą być przygotowane przez autora w klipie parametrów materiału sterowanym osobnym kanałem.

## 12. Mapa danych autorskich do formatu

| Element przygotowany w Blenderze | Dane docelowe | Dodatkowy kontrakt symulacji |
|---|---|---|
| Point / Spot / Sun | `KHR_lights_punctual` | Opcjonalny `light_control` |
| Emisyjny materiał soczewki | PBR + emissive strength | Opcjonalny `emission_control` |
| Ruch drzwi wypieczony z constraints | Klip TRS | `playback` kanału otwarcia |
| Armatura i deformowany przewód | Skin i jointy | Klip albo mechanizm sterujący jointami |
| Shape keys ładunku | Morph targets | Klip próbkowany kanałem ładunku |
| Pantograf z IK autorskim | Wypieczony klip wszystkich członów | `pantograph.height_to_time` |
| Dwuczłonowy rig sterowany celem świata | Node'y / jointy | `ik` z kanałem celu i pole |
| Koło z poprawnym pivotem | Node | `rotation_control` |
| Kamera kabiny | `camera` i node | `semantic_id` stanowiska |
| Malowanie pojazdu | `KHR_materials_variants` | Wybór wariantu w konfiguracji |
| Empty wylotu spalin | Node transformacyjny | `smoke_source` |

Eksport MUST zachować node'y używane jako pivoty, punkty kontaktu i cele referencji. Optymalizator aktualizuje indeksy, JSON Pointer i mapowania wariantów, zachowując semantyczne ID. Importer tworzy mapy ID raz podczas ładowania assetu.

## 13. Mapowanie T3D/E3D i MMD

Mapowanie dotyczy funkcji rozpoznanej z nazwy bazowej i konfiguracji pojazdu. Konwerter ustala znaczenie sufiksu w tym kontekście.

### 13.1. Światła i sprzęgi

| Legacy | Znaczenie w sprawdzonym kodzie | Wynik |
|---|---|---|
| `headlamp*_off` | Zgaszony reflektor | Geometria sterowana `mode == off` |
| `headlamp*_on` | Pełne światło | Geometria sterowana `mode == full` |
| `headlamp*_xon` | Przygaszony reflektor | Geometria sterowana `mode == dimmed` |
| `headsignal*_xon` | Przygaszone światło pomocnicze | Analogiczny kanał pomocniczej lampy |
| `highbeam*_xon` | Wariant wybierany przy włączeniu długich | Kanał aktywności światła drogowego |
| `coupler*_off` | Rozłączony sprzęg | `visual == disconnected` |
| `coupler*_on` | Połączony, właściciel reprezentacji | `visual == connected_owner` |
| `coupler*_xon` | Druga strona połączenia lub adapter | Warianty `connected_peer` i `adapter` |
| `pneumatic*`, `cpneumatic*` z `_off` | Przewód odłożony | `visual == disconnected` |
| Te przewody z `_on` | Połączenie proste | `visual == straight` |
| Te przewody z `_xon` | Połączenie ukośne | `visual == slanted` |

Powiązania przód/tył oraz strona są odczytywane z konfiguracji legacy. W kodzie `headlamp11` to przedni górny, `12` prawy, `13` lewy; `21…23` odpowiadają tyłowi. Prefiksy `cpneumatic` i `pneumatic` konwerter mapuje zgodnie z funkcją przewodu w symulacji, odpowiednio kanałem połączenia hamulcowego i zasilającego.

Kod wybiera `_xon` dla reflektorów przy `DimHeadlights`, a dla `highbeam` podczas włączenia długich. Wspólna klasa obsługująca te warianty jest szczegółem implementacji legacy; wynik mapowania wynika z funkcji urządzenia.

### 13.2. Części i animacje

| Legacy / MMD | Wynik |
|---|---|
| `bogie1/2`, `boogie01/02` | `vehicle.bogie.0/1`, obrót z kanału wózka |
| `animwheelprefix:` | Odpowiedni zestaw lub koło, `rotation_control` |
| `animdoorprefix:` | Drzwi, klip `playback` stopnia otwarcia |
| `animstepprefix:` | Stopień, klip wysunięcia |
| `animmirrorprefix:` | Lustro, klip położenia |
| `animwiperprefix:` | Wycieraczka, klip fazy |
| `animpantrd1prefix:`, `animpantrd2prefix:` | Dolne człony pantografu |
| `animpantrg1prefix:`, `animpantrg2prefix:` | Górne człony pantografu |
| `animpantslprefix:` | Ślizgacz i node kontaktu |
| `animpendulumprefix:` | Człon mechanizmu biegowego, wypieczona kinematyka |
| `buffer_left01/02`, `buffer_right01/02` | Zderzaki, przesunięcie kanałem ściśnięcia |
| `cab0/1/2` | Kabiny z jawnym mapowaniem indeksów autora |
| `corridorN`, `korytarzN`, `compartmentN`, `przedzialN` | Zespoły korytarzy i przedziałów |
| `external_loadN` | Node ładunku z kanałami wizualizacji |
| `mechanik1/2` | Postać sterowana zajęciem kabiny |
| `shutters1/2` | Żaluzje sterowane klipem lub widocznością |
| `external_only` | `view_filter` zawierający wymagane widoki zewnętrzne |
| Materiały `-1…-4` / `replacableskin` | `material_slot` `skin.1…skin.4` |
| Tablica kierunkowa | `display` na rozpoznanej powierzchni |
| `cien` | Geometria pomocnicza klasyfikowana w raporcie migracji; cienie modelu w runtime wynikają z jego geometrii |

Konwerter MUST wygenerować tabelę użytych nazw legacy, docelowych ID, kanałów i przybliżeń. Rig pantografu wymaga odtworzenia kinematyki z parametrów modelu; same nazwy członów służą identyfikacji. Materiały i światła są kalibrowane do docelowych parametrów PBR i fotometrii.

## 14. Kolejność aktualizacji i własność danych

1. Symulacja wyznacza transform instancji i migawkę kanałów.
2. Runtime przywraca bazowe wartości sterowanych właściwości, następnie próbkuje klipy.
3. Sterowniki obrotu/przesunięcia oraz mechanizmy aktualizują przypisane node'y w porządku rodzic–potomek.
4. Runtime przelicza transformacje światowe oraz macierze skinów.
5. Sterowanie światłami, materiałami, widocznością i emiterami korzysta z tej samej migawki.
6. Renderer realizuje filtry widoku i oblicza billboardy dla konkretnej kamery.

Mechanizm może odczytywać wynik przodka, ale kolejność zależności MUST tworzyć graf acykliczny. Każdy zapisany TRS, parametr materiału, parametr światła lub widoczność ma jednego autora w danym przebiegu. Kolizja zakresów sterowania jest błędem walidacji.

Współdzielony asset przechowuje dane bazowe. Materiały, światła, klipy, mechanizmy i cząstki mają stan przypisany instancji. Despawn kończy emisję i zwalnia jej stan zgodnie z polityką świata; ponowne utworzenie instancji odtwarza wartości bazowe.

## 15. Walidacja i przykłady testów integracyjnych

Walidator profilu MUST dodatkowo sprawdzić:

- `model_class: dynamic` oraz właściwy `model_type`;
- zgodność ID części i rozwiązywalność kanałów zadeklarowanych przez model;
- poprawne pivoty, osie jednostkowe i wypieczoną skalę mechanizmów;
- zakresy klipów i jednoznaczny autor każdej właściwości;
- istnienie referencji `pantograph.animation` oraz `contact_node`;
- monotoniczność tabeli pantografu, jej pokrycie i tolerancję kontaktu;
- strukturę, długości i płaszczyznę spoczynkową łańcucha IK;
- domyślne stany widoczności wszystkich wariantów;
- zgodność granic z próbkami pozy drzwi, wózków, pantografów i innych mechanizmów;
- materiały wyświetlaczy, fonty i zasoby emiterów.

Testy integracyjne pipeline SHOULD obejmować:

| Próba | Oczekiwany wynik |
|---|---|
| Dwa pojazdy tego samego assetu z innym stanem świateł | Niezależne światła i emisja soczewek |
| Drzwi 0 → 0.5 → 1 → 0.2 | Dokładne próbkowanie pozy, zgodne odwrócenie ruchu |
| Koło przez kilka obrotów i zmianę znaku | Ciągły obrót zgodny z osią |
| Skręt wózka z obracającymi się kołami | Poprawne złożenie transformacji |
| Reflektor off → dimmed → full | Właściwa intensywność i wybrana geometria |
| Sprzęg peer / owner / adapter | Właściwa reprezentacja każdego końca |
| Pantograf na krańcach i pomiędzy wierszami tabeli | Kontakt w tolerancji; poprawna pozycja graniczna |
| Cel IK osiągalny, graniczny i współliniowy z pole | Stabilna, deterministyczna poza |
| Przełączenie kamery cab/external/reflection | Właściwe poddrzewa i billboardy |
| Emiter na jadącym i skręcającym pojeździe | Poprawne dziedziczenie prędkości |
| Ponowny eksport z inną kolejnością node'ów | Zachowane ID i poprawione referencje indeksowe |

## 16. Źródła standardów i mapowania

- [Reguły wspólne i źródła glTF](STATIC_MODEL_FORMAT.md).
- [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer).
- [KHR_node_visibility](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_node_visibility).
- [KHR_lights_punctual](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_lights_punctual).
- Źródła legacy: `vehicle/AirCoupler.cpp` (`Init`, `Update`), `vehicle/AirCoupler.h`, `vehicle/DynObj.cpp` (obsługa sprzęgów i `DimHeadlights`, inicjalizacja prefiksów części), `vehicle/Button.cpp`, `vehicle/Gauge.cpp`, `model/AnimModel.cpp`.
