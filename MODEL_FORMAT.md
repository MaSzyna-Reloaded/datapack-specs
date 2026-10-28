# Modele — wspólny profil glTF

Status: projekt specyfikacji 0.5.0. Format bazowy: glTF 2.0.

## 1. Zakres i reguły wspólne

Profil opisuje reguły wspólne dla [modeli statycznych](STATIC_MODEL_FORMAT.md) i [dynamicznych](DYNAMIC_MODEL_FORMAT.md).

Własne pola zdefiniowane tutaj znajdują się bezpośrednio w `extras` właściwego obiektu. Klucze opisują konkretne dane lub zachowania. Standardowe dane glTF są źródłem wartości geometrii, materiałów, świateł i animacji.

Dokument definiuje kontrakt docelowy. Zgodność implementacji jest sprawdzana dla wersji wskazanej w `model_format_version`. Obsługa rozszerzenia w formacie, eksporterze i mechanizmie renderowania jest weryfikowana oddzielnie podczas integracji.

## 2. Kontener, układ i zasoby

Model jest zapisany jako poprawny plik glTF 2.0: `.gltf` z zasobami albo `.glb`. Zewnętrzne adresy URI zasobów są względne wobec pliku glTF. Pakiet zawiera wszystkie zależności. Plik GLB może również mieć zależności zewnętrzne, które pakowarka rozwiązuje tak samo.

Profil przyjmuje metry, sekundy i radiany. Model ma jedną domyślną scenę z jednym korzeniem reprezentującym cały obiekt. Układ modelu: +Y w górę, +Z przód obiektu, +X w lewo z perspektywy patrzącego w kierunku +Z. Eksporter przelicza układ narzędzia autorskiego do tego układu.

Przekształcenie korzenia w pliku jest jednostkowe. Przekształcenie umieszczenia instancji przechowuje świat. Światła kierunkowe i kamery zachowują własne konwencje osi glTF. Ruchome węzły używają TRS z prawidłowo ustawionym punktem obrotu. Skala mechanizmów jest wypieczona przed eksportem.

Geometria, hierarchia, szkielety, cele morfingu i klipy korzystają z odpowiednich obiektów [glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html). Nazwy `name` służą prezentacji w narzędziach.

## 3. Standardowe reprezentacje

| Dane autora | Reprezentacja |
| --- | --- |
| Siatka i materiały PBR | `meshes`, `materials`, `textures` |
| Przekształcenie i hierarchia | `nodes` |
| Ruch / obrót / skala | `animations` z kanałami TRS |
| Szkielet i klucze kształtu | `skins`, węzły szkieletu, cele morfingu i animowane `weights` |
| Opcjonalny punkt obserwatora | Kamera glTF przypięta do węzła |
| Światło stożkowe, punktowe lub kierunkowe | `KHR_lights_punctual`: `spot` / `point` / `directional` |
| Silna emisja powierzchni | `emissiveFactor`, `emissiveTexture`, `KHR_materials_emissive_strength` |
| Parametry zmieniające się w czasie | `KHR_animation_pointer` |
| Widoczność gałęzi | `KHR_node_visibility` |
| Poziomy szczegółowości (LOD) | `MSFT_lod` na węźle albo materiale |

Każde rozszerzenie użyte w modelu znajduje się w `extensionsUsed`. Rozszerzenia niezbędne do poprawnej interpretacji modelu znajdują się również w `extensionsRequired`; importer sprawdza je przed utworzeniem instancji. Gdy warianty geometrii polegają na ukrywaniu części, wymagane jest `KHR_node_visibility`. Sterowanie przez wskaźniki wymaga `KHR_animation_pointer`.

Poziomy szczegółowości zapisuje [MSFT_lod](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/MSFT_lod). Węzeł albo materiał zawierający rozszerzenie jest najwyższym poziomem LOD. Tablica `ids` wskazuje kolejne węzły albo materiały w kolejności malejącej szczegółowości.

Poniższe rozszerzenia mogą być dobierane do materiałów i dystrybucji zasobów:

| Potrzeba | Rozszerzenie |
| --- | --- |
| Szkło, przechodzenie światła | `KHR_materials_transmission`, `KHR_materials_volume`, `KHR_materials_ior` |
| Lakier, odbicia | `KHR_materials_clearcoat`, `KHR_materials_specular` |
| Kierunkowe refleksy, tkaniny | `KHR_materials_anisotropy`, `KHR_materials_sheen` |
| Iryzacja, dyspersja | `KHR_materials_iridescence`, `KHR_materials_dispersion` |
| Materiał niezależny od oświetlenia | `KHR_materials_unlit` |
| Przekształcenie UV | `KHR_texture_transform` |
| Tekstury KTX2/Basis Universal | `KHR_texture_basisu` |
| Kompresja / kwantyzacja geometrii | `KHR_draco_mesh_compression`, `KHR_mesh_quantization` |
| Metadane autorstwa i praw | `KHR_xmp_json_ld` |

Dobór rozszerzeń opiera się na [rejestrze Khronos](https://github.com/KhronosGroup/glTF/blob/main/extensions/README.md). Lista powyżej jest wyborem funkcji dla tego profilu. Każde użycie stosuje semantykę i ograniczenia właściwej specyfikacji rozszerzenia.

Profil nie przenosi kodu shaderów ani grafów materiałów. Wygląd zależny od stanu świata opisują parametry, np. reakcję na pogodę w 4.10, a implementuje go renderer. Dołączanie programów GLSL do glTF próbowało rozszerzenie `KHR_techniques_webgl`, które Khronos zarchiwizował.

## 4. Struktura modelu

### 4.1. Nagłówek

Przykład nagłówka `asset`:

```json
{
  "version": "2.0",
  "extras": {
    "model_format_version": "0.5.0"
  }
}
```


| Klucz w `asset.extras` | Opisana wartość | Przykład | Wymaganie |
| --- | --- | --- | --- |
| `model_format_version` | Wersja profilu modelu. | `"0.5.0"` | Wymagane |
| `osm` | Tagi opisujące model. | `{"natural":"tree"}` | Opcjonalne |
| `geo` | Położenie i orientacja geograficzna modelu. | `{"lat":54.3520,"lng":18.6466,"elevation":12.5,"heading":90.0}` | Opcjonalne |
| `t3d` | Dane zgodności z T3D/E3D. | `{"skins":["body","cab"]}` | Opcjonalne |


`extras.semantic_id` w węzłach, materiałach i animacjach identyfikuje elementy używane przez symulację. Wartości są unikalne w obrębie danej kolekcji modelu i stabilne przy ponownym eksporcie. Typ referencji wynika z pola: `node` wskazuje węzeł, `material` materiał, a `animation` klip. Wartością jest zawsze `semantic_id` docelowego obiektu.

Identyfikator to segmenty rozdzielone kropką, zgodne z `[a-z0-9_]+`, np. `building.clock.minute`, `infrastructure.signal.0.red`, `vehicle.door.left.0`. Zarezerwowane znaczenia konkretnych ścieżek definiują profile. Pozostałe ścieżki są stabilnymi adresami autora. Indeksy porządkowe zaczynają się od zera.

Standardowe referencje glTF i JSON Pointer używają indeksów zgodnie ze standardem. Eksporter aktualizuje je podczas przebudowy tablic. Identyfikatory semantyczne służą wiązaniu modelu z konfiguracją i stanem świata.

### 4.2. Tagi

Model identyfikuje ścieżka jego pliku w pakiecie danych. Semantykę modelu opisują tagi w `asset.extras.osm`. Jest to obiekt, którego klucze i wartości odpowiadają tagom OSM, np. `{"natural":"tree"}` albo `{"building":"train_station"}`. Można podać kilka tagów jednocześnie.

| Klucz w `asset.extras` | Opisana wartość | Przykład | Wymaganie |
|---|---|---|---|
| `osm` | Mapa tagów opisujących przeznaczenie modelu. | `{"natural":"tree"}` | Opcjonalne |

#### 4.2.1. Well-known tag sets

Tabela podaje zestawy tagów rozpoznawane przez narzędzia. Powstała na podstawie kategorii i modeli z `nodebank.txt` MaSzyny oraz odpowiadających im tagów OSM.

| `asset.extras.osm` | Znaczenie | Grupa w `nodebank.txt` |
|---|---|---|
| `{"natural":"tree"}` | Drzewo | Małe drzewka, Duże drzewka |
| `{"natural":"shrub"}` | Krzew | Krzaki |
| `{"landuse":"grass"}` | Trawa | Trawy, Trawy atlas |
| `{"building":"house"}` | Dom mieszkalny | Domki |
| `{"building":"barn"}` | Stodoła | Stodoły |
| `{"building":"apartments"}` | Blok mieszkalny | Bloki, wieżowce |
| `{"building":"industrial"}` | Budynek przemysłowy | Przemysł |
| `{"building":"train_station"}` | Budynek stacyjny lub dworzec | Dworce |
| `{"railway":"signal_box"}` | Nastawnia | Nastawnie |
| `{"railway":"signal"}` | Semafor lub wskaźnik | Semafory i wskaźniki |
| `{"railway":"platform"}` | Peron | Perony |
| `{"railway":"buffer_stop"}` | Kozioł oporowy | Kozły oporowe |
| `{"railway":"level_crossing"}` | Przejazd kolejowo-drogowy | Przejazdy |
| `{"railway":"turntable"}` | Obrotnica | Mosty, wiadukty, tunele, obrotnice |
| `{"barrier":"gate"}` | Brama lub furtka | Bramy |
| `{"barrier":"fence"}` | Ogrodzenie | Ogrodzenia |
| `{"power":"pole"}` | Słup energetyczny | Słupy nN, SN |
| `{"power":"tower"}` | Słup wysokiego napięcia | Słupy WN i NN |
| `{"highway":"street_lamp"}` | Lampa lub latarnia | Lampy i latarnie |
| `{"man_made":"street_cabinet"}` | Szafa techniczna | Obiekty elektryczne, obiekty przytorowe |
| `{"man_made":"bridge"}` | Most lub wiadukt | Mosty, wiadukty, tunele, obrotnice |

### 4.3. Położenie geograficzne

Opcjonalne `asset.extras.geo` opisuje położenie początku układu modelu oraz jego orientację:

| Klucz w `geo` | Opisana wartość | Przykład | Wymaganie |
|---|---|---|---|
| `lat` | Szerokość geograficzna WGS84 w stopniach, od −90 do 90. | `54.3520` | Wymagane, gdy występuje `geo` |
| `lng` | Długość geograficzna WGS84 w stopniach, od −180 do 180. | `18.6466` | Wymagane, gdy występuje `geo` |
| `elevation` | Wysokość w metrach nad średnim poziomem morza. | `12.5` | Opcjonalne |
| `heading` | Kierunek lokalnej osi `+Z` w stopniach, zgodnie z ruchem wskazówek zegara od północy geograficznej. | `90.0` | Opcjonalne; domyślnie `0.0` |

Dla `heading: 0` lokalne `+Z` wskazuje północ, a dla `heading: 90` wschód. Lokalna oś `+Y` wskazuje górę.

### 4.4. Materiały

Materiały zapisuje standardowy obiekt [`material`](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html#materials) w tablicy `materials`. Pole `primitive.material` zawiera indeks materiału używanego przez dany prymityw siatki. Brak tego pola oznacza domyślny materiał glTF. Profil nie dodaje własnego formatu materiałów.

| Pole w `material` | Opisana wartość | Przykład | Domyślna wartość | Wymaganie |
| --- | --- | --- | --- | --- |
| `name` | Nazwa materiału. | `"paint_red"` | — | Opcjonalne |
| `pbrMetallicRoughness.baseColorFactor` | Mnożnik koloru bazowego RGBA. | `[0.8, 0.1, 0.1, 1.0]` | `[1.0, 1.0, 1.0, 1.0]` | Opcjonalne |
| `pbrMetallicRoughness.baseColorTexture.index` | Indeks tekstury koloru bazowego. | `0` | — | Opcjonalne |
| `pbrMetallicRoughness.metallicFactor` | Metaliczność od 0 do 1. | `0.0` | `1.0` | Opcjonalne |
| `pbrMetallicRoughness.roughnessFactor` | Szorstkość od 0 do 1. | `0.7` | `1.0` | Opcjonalne |
| `pbrMetallicRoughness.metallicRoughnessTexture.index` | Indeks wspólnej tekstury szorstkości i metaliczności. | `1` | — | Opcjonalne |
| `normalTexture.index` | Indeks tekstury normalnych. | `2` | — | Opcjonalne |
| `normalTexture.scale` | Mnożnik odchylenia normalnych. | `0.8` | `1.0` | Opcjonalne |
| `occlusionTexture.index` | Indeks tekstury przesłaniania światła otoczenia. | `1` | — | Opcjonalne |
| `occlusionTexture.strength` | Siła przesłaniania od 0 do 1. | `1.0` | `1.0` | Opcjonalne |
| `emissiveTexture.index` | Indeks tekstury emisji. | `3` | — | Opcjonalne |
| `emissiveFactor` | Mnożnik koloru emisji RGB. | `[1.0, 0.8, 0.6]` | `[0.0, 0.0, 0.0]` | Opcjonalne |
| `alphaMode` | Sposób obsługi kanału alfa: `OPAQUE`, `MASK` albo `BLEND`. | `"MASK"` | `"OPAQUE"` | Opcjonalne |
| `alphaCutoff` | Próg odrzucenia fragmentu przy `alphaMode: "MASK"`. | `0.5` | `0.5` | Opcjonalne |
| `doubleSided` | Rysowanie obu stron powierzchni. | `true` | `false` | Opcjonalne |

Każde pole tekstury zawiera `index` wskazujący element tablicy `textures`. Element `texture` wskazuje obraz przez `source` i opcjonalny sposób próbkowania przez `sampler`. Pole `texCoord` wybiera zestaw współrzędnych tekstury; jego domyślna wartość to `0`.

| Tekstura | Wykorzystanie kanałów | Przestrzeń barw |
| --- | --- | --- |
| `baseColorTexture` | RGB: kolor bazowy; A: przezroczystość | sRGB |
| `metallicRoughnessTexture` | G: szorstkość; B: metaliczność | liniowa |
| `normalTexture` | RGB: normalna w przestrzeni stycznej | liniowa |
| `occlusionTexture` | R: przesłanianie światła otoczenia | liniowa |
| `emissiveTexture` | RGB: kolor emisji | sRGB |

| Zastosowanie | Zapis |
|---|---|
| Pełna powierzchnia | `alphaMode: OPAQUE` |
| Wycinanie według kanału alfa: siatka, liść, ogrodzenie | `alphaMode: MASK`, próg `alphaCutoff` |
| Mieszanie przezroczystości | `alphaMode: BLEND` |
| Szkło optyczne | PBR z rozszerzeniami transmission / volume / IOR |
| Powierzchnia dwustronna | `doubleSided` |

Eksporter dobiera tryb do zamierzonego efektu. Przezroczystość pokrycia i transmisja optyczna są niezależnymi własnościami materiału. Szyba z zabrudzeniami może łączyć odpowiednio przygotowane warstwy.

Stała emisja powierzchni jest zapisana w materiale, z opcjonalnym [KHR_materials_emissive_strength](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_materials_emissive_strength). Oświetlenie otoczenia przypisuje się światłu powiązanemu z odpowiednim węzłem.

`emissiveTexture` jest zawsze renderowana zgodnie z materiałem i nie służy do włączania ani wyłączania świateł.

Reakcję materiału na pogodę opisuje opcjonalne `material.extras.weather` (4.10.2), a temperaturę powierzchni `material.extras.heat` (4.11.1).

### 4.5. Wymienne tekstury

Model zawiera domyślne materiały i tekstury glTF. Zestaw wymiennych tekstur jest mapą ich nazw na zasoby, np. `{"body":"skins/red/body.png"}`. Podmienia wskazane części modelu; pozostałe używają tekstur domyślnych. Opcjonalne `asset.extras.t3d.skins` mapuje numery starszego formatu na te nazwy, bez mieszania ich z numerami materiałów glTF.

| Klucz w `asset.extras.t3d` | Opisana wartość | Przykład | Wymaganie |
|---|---|---|---|
| `skins` | Nazwy slotów wymiennych tekstur w kolejności indeksów T3D/E3D; indeks 0 odpowiada `-1`. | `["body", "cab"]` | Opcjonalne |

| Klucz w `node.extras` | Opisana wartość | Przykład | Wymaganie |
|---|---|---|---|
| `replacable_texture` | Nazwa wymiennej tekstury części modelu reprezentowanej przez węzeł. | `"body"` | Opcjonalne; obecne dla części z wymienną teksturą |

### 4.6. Warstwy świateł

Model może mieć kilka niezależnych grup świateł, np. komory semafora lub lampy pojazdu. Każda grupa ma węzeł (podmodel) z siatką powierzchni świateł. Używa się dwóch atlasów: bazowego wyglądu i osobnej warstwy światła. Przypisanie powierzchni do grupy wynika z węzła. Sposób odwołania do atlasu światła pozostaje do ustalenia.

| Klucz w `node.extras` | Opisana wartość | Przykład | Domyślna wartość | Wymaganie |
|---|---|---|---|---|
| `light_id` | Identyfikator niezależnie sterowanej grupy świateł. | `0` | — | Wymagane w węźle z siatką świateł |
| `light_color` | Kolor światła jako RGB. | `[1.0, 0.0, 0.0]` | `[1.0, 1.0, 1.0]` | Opcjonalne w węźle z `light_id` |
| `light_state` | Stan światła przedstawiany przez siatkę węzła: `"on"` albo `"off"`. | `"on"` | `"on"` | Opcjonalne w węźle z `light_id` |
| `light_threshold` | Poziom oświetlenia środowiska, poniżej którego światło jest włączone. | `0.325` | `0.325` | Opcjonalne w węźle z `light_id` |

### 4.7. Definicje świateł (DRAFT)

Światła zapisuje [KHR_lights_punctual](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_lights_punctual). Eksport Blendera mapuje `POINT` na `point`, `SPOT` na `spot`, `SUN` na `directional`. Eksporter odpowiada za przeliczenie jednostek autora.

Definicja zawiera `type`, `color`, `intensity`, opcjonalne `range` oraz kąty `spot.innerConeAngle` i `spot.outerConeAngle`. `point` i `spot` używają kandeli, `directional` luksów. Przekształcenie węzła ustala pozycję i orientację; światło `spot` świeci w lokalnym kierunku −Z. Pominięte `range` oznacza nieskończony zasięg zgodnie z rozszerzeniem.

Przykład kompletnego glTF z lampą:

```json
{
  "asset": {"version": "2.0"},
  "extensionsUsed": ["KHR_lights_punctual"],
  "extensions": {
    "KHR_lights_punctual": {
      "lights": [{
        "type": "spot",
        "color": [1.0, 0.9, 0.75],
        "intensity": 1200.0,
        "range": 40.0,
        "spot": {"innerConeAngle": 0.25, "outerConeAngle": 0.5}
      }]
    }
  },
  "nodes": [{
    "extensions": {"KHR_lights_punctual": {"light": 0}},
    "extras": {"semantic_id": "infrastructure.lamp.0"}
  }],
  "scenes": [{"nodes": [0]}],
  "scene": 0
}
```

To przykład mechanizmu światła; nagłówek profilu dodaje się według sekcji 4.1. Zwykła lampa działa po załadowaniu tych danych. Sterowanie scenariuszem opisuje opcjonalne `light_control` w węźle:

```json
{
  "light_control": {
    "source": "lighting.platform.0.level",
    "default": 1.0
  }
}
```

`source` wskazuje kanał liczbowy instancji w zakresie [0,1]. Wartość domyślna jest wymagana. Silnik mnoży bazowe `intensity` światła przez wartość kanału. Regułę zmierzchu lub harmonogram dostarcza świat. Wspólna definicja światła jest instancjonowana z oddzielnym stanem dla każdego sterowanego węzła.

Profile fotometryczne zapisuje opcjonalne [EXT_lights_ies](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/EXT_lights_ies); zasoby oświetlenia środowiskowego opisuje [EXT_lights_image_based](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/EXT_lights_image_based). Ich użycie wymaga zadeklarowanego wsparcia w procesie przetwarzania.

### 4.8. Dym i inne emitery (DRAFT)

Pozycję emitera reprezentuje węzeł. Lokalny +Y określa kierunek początkowy. `node.extras.smoke_source` opisuje konkretny efekt cząstkowy:

```json
{
  "semantic_id": "building.chimney.0",
  "smoke_source": {
    "rate": 12.0,
    "max_particles": 400,
    "lifetime": [3.0, 6.0],
    "speed": [0.8, 1.4],
    "cone_angle": 0.2,
    "size": [0.25, 0.5],
    "growth": 0.15,
    "acceleration": [0.0, 0.1, 0.0],
    "drag": 0.1,
    "inherit_velocity": 0.0,
    "material": "effects.chimney_smoke",
    "source": "chimney.0.output",
    "default": 0.0
  }
}
```

Wymagane: `rate` w cząstkach/s, dodatnie całkowite `max_particles`, dodatni zakres `lifetime` w sekundach, nieujemne zakresy `speed` w m/s i `size` (średnica) w metrach, referencja `material`. Pozostałe pola mają wartości domyślne 0, z wyjątkiem `default` = 1. `source` jest opcjonalnym kanałem [0,1], który mnoży `rate`.

`cone_angle` to półkąt [0,π], rozkład kierunków jest jednostajny po kącie bryłowym. Parametry w zakresach są losowane jednostajnie i niezależnie przy narodzinach. `growth` jest przyrostem średnicy w m/s; średnica jest ograniczana od dołu zerem. `acceleration` jest wektorem m/s² układu modelu, `drag` ≥ 0 współczynnikiem oporu 1/s; równanie prędkości to `dv/dt = acceleration - drag*v`. `inherit_velocity` ∈ [0,1] mnoży prędkość emitera w układzie świata przy narodzinach.

Cząstki żyją w układzie świata i są płaszczyznami skierowanymi do kamery. Przy narodzinach kierunek prędkości jest przeliczany przekształceniem emitera, a przyspieszenie obrotem modelu do układu świata; te wartości są następnie używane przez integrator. Materiał glTF określa teksturę, barwę, emisję i kanał alfa. Kanał alfa jest dodatkowo mnożony przez `1-age/lifetime`. Po osiągnięciu limitu nowe cząstki są pomijane, a po wyłączeniu kanału emisji istniejące kończą życie. Ukrycie węzła przez `KHR_node_visibility` zatrzymuje narodziny i ukrywa cały efekt; wiek cząstek nadal narasta.

### 4.9. Animacja (DRAFT)

Klip przechowuje animację przygotowaną przez autora. Wynik więzów i odwrotnej kinematyki (IK) obliczonych w narzędziu jest wypiekany do kanałów. Animowanie parametrów materiałów i świateł używa [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer).

Klip wybierany przez świat zawiera `animation.extras.semantic_id`. Opcjonalne `animation.extras.playback` definiuje jego sterowanie:

```json
{
  "semantic_id": "building.gate.opening",
  "playback": {
    "mode": "position",
    "source": "gate.0.open_fraction",
    "input": [0.0, 1.0],
    "time": [0.0, 2.4],
    "default": 0.0
  }
}
```

W trybie `position` wejście x jest ograniczane do `[a,b] = input`, a czas próbki wynosi `t0 + (t1-t0)*(x-a)/(b-a)`. Wymagane są wszystkie pokazane pola; `a < b`, a `time` mieści się w klipie. Ten tryb umożliwia odwracanie i zatrzymywanie ruchu przy dowolnym otwarciu.

Tryb `loop` używa `source` jako wartości logicznej uruchomienia, `default` jako wartości logicznej, `time` jako zakresu klipu oraz opcjonalnego `speed` > 0 (domyślnie 1). Zmiana `false` → `true` rozpoczyna odtwarzanie od t0, `true` odtwarza cyklicznie, a `false` zatrzymuje na t0. Czas pochodzi z zegara symulacji. Stałą pętlę autor zapisuje przez pominięcie `source` i ustawienie `default: true`.

Widoczność i jej animacja używają [KHR_node_visibility](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_node_visibility). Domyślne `visible` zapisuje rozszerzenie węzła. Kanał logiczny może sterować nim przez `node.extras.visibility_control`:

```json
{
  "visibility_control": {
    "source": "season",
    "equals": "winter"
  }
}
```

Przy `equals` wynik jest ścisłym porównaniem wartości kanału z literałem. Przy samym `source` wymagany jest kanał logiczny. Przed pierwszą aktualizacją kanału obowiązuje `visible` z rozszerzenia. Taki zapis obsługuje warianty sezonowe i wygląd zależny od sygnału.

Zegar może używać klipów `position` próbkowanych kanałami `clock.second`, `clock.minute`, `clock.hour12`, `clock.hour24`. Świat dostarcza liczby z częścią ułamkową; autor ustala zakres i interpolację klipu. Zaprogramowane mruganie może używać klipu `loop`, podczas gdy aspekt sygnalizatora jest wejściem ze scenariusza.

#### 4.9.1. Kontrakt aktualizacji

Kanał `source` jest nazwą wartości dostarczanej przez świat lub symulację do konkretnej instancji. Konfiguracja instancji rozwiązuje wymagane kanały przed aktywacją. Kanały przekazują wartości logiczne, liczby albo tekst zgodnie z odbiorcą; kanały celów IK w profilu dynamicznym i `weather.wind` przekazują wektor trójelementowy (`vec3`). Kanały `clock.*` i `weather.*` dostarcza świat. Liczby są skończone.

Kolejność: odczyt spójnej migawki kanałów → próbkowanie klipów → sterowniki proceduralne → aktualizacja przekształceń w układzie świata → światła, widoczność, emisja i cząstki. Każda zapisywana właściwość ma jednego aktywnego autora w danej instancji. Konflikt dwóch klipów lub klipu i sterownika jest błędem konfiguracji. Współdzielone materiały mają stan instancji albo kopię przy zapisie. Dwie instancje zachowują niezależne stany odtwarzania.

### 4.10. Pogoda (DRAFT)

Opad, jego kierunek i wiatr symuluje świat. Model opisuje reakcję swoich powierzchni: moknięcie, spływ wody, zaleganie śniegu oraz odprowadzanie wody okapem i rynną. Renderer rysuje efekt na podstawie tych danych; model nie zawiera kodu shaderów (sekcja 3). Renderer bez obsługi tej sekcji pokazuje materiały bez zmian.

#### 4.10.1. Kanały pogodowe

Kanały `weather.*` są zarezerwowane. Świat dostarcza je każdej instancji bez dodatkowej konfiguracji i może nadać instancji wartości lokalne, np. zero pod zadaszeniem. Model może ich używać w każdym polu `source`, np. `smoke_source.source: "weather.rain"`.

| Kanał | Typ | Opisana wartość |
| --- | --- | --- |
| `weather.rain` | liczba [0,1] | Natężenie deszczu: `min(opad_mm_h / 20, 1)`. |
| `weather.snow` | liczba [0,1] | Natężenie opadu śniegu w ekwiwalencie wodnym: `min(opad_mm_h / 5, 1)`. |
| `weather.wetness` | liczba [0,1] | Zmoczenie odsłoniętej powierzchni: 0 sucha, 1 pełny film wody. |
| `weather.snow_cover` | liczba [0,1] | Pokrycie śniegiem odsłoniętej powierzchni poziomej. |
| `weather.wind` | `vec3` | Prędkość wiatru w m/s w układzie świata. |
| `weather.temperature` | liczba | Temperatura powietrza w °C. |

Narastanie i zanikanie `weather.wetness` oraz `weather.snow_cover` (moknięcie, schnięcie, topnienie) oblicza świat. Model nie przechowuje stanu pogody.

#### 4.10.2. Reakcja materiału

Opcjonalne `material.extras.weather` opisuje, jak materiał reaguje na kanały pogodowe:

```json
{
  "semantic_id": "building.roof_tiles",
  "weather": {
    "wet": {"porosity": 0.6, "roughness": 0.1, "mask": {"index": 4}},
    "runoff": {"flow": "slope", "speed": 1.1, "density": 6.0, "min_slope": 0.05},
    "snow": {"max_slope": 0.8}
  }
}
```

| Klucz w `weather` | Opisana wartość | Przykład | Domyślna wartość | Wymaganie |
| --- | --- | --- | --- | --- |
| `wet.porosity` | Porowatość od 0 do 1; określa ściemnienie koloru bazowego po zmoczeniu. | `0.6` | `0.0` | Opcjonalne |
| `wet.roughness` | Szorstkość powierzchni pokrytej filmem wody, od 0 do 1. | `0.1` | `0.1` | Opcjonalne |
| `wet.mask` | Tekstura maski: R ekspozycja na opad, G podatność na stojącą wodę. | `{"index": 4}` | R = 1, G = 0 | Opcjonalne |
| `runoff.flow` | Kierunek spływu: `"slope"` albo tekstura kierunku. | `"slope"` | — | Wymagane w `runoff` |
| `runoff.speed` | Prędkość strużek w m/s. | `1.1` | `1.0` | Opcjonalne |
| `runoff.density` | Przybliżona liczba strużek na metr w poprzek spływu. | `6.0` | `4.0` | Opcjonalne |
| `runoff.min_slope` | Najmniejsze nachylenie powierzchni w radianach, przy którym powstają strużki. | `0.05` | `0.02` | Opcjonalne |
| `snow.max_slope` | Największe nachylenie powierzchni w radianach, na którym zalega śnieg. | `0.8` | `0.8` | Opcjonalne |
| `snow.mask` | Tekstura maski: R możliwość zalegania śniegu. | `{"index": 5}` | R = 1 | Opcjonalne |

Pola tekstur mają postać `textureInfo` (`index`, opcjonalne `texCoord`) i są próbkowane w przestrzeni liniowej. Obecność `wet`, `runoff` albo `snow` włącza dany efekt.

Zmoczenie `w = weather.wetness × mask.R`. Kolor bazowy RGB jest mnożony przez `1 − 0.6 × w × porosity`, a szorstkość jest interpolowana liniowo od wartości materiału do `wet.roughness` z wagą `w`. Metaliczność nie zmienia się. Fragment z `mask.G` > 0 może pokryć stojąca woda: gładka powierzchnia z normalną geometrii. Przy `weather.wetness` = 1 pokrywa ona cały fragment z `mask.G` = 1. Krzywą wypełniania dobiera renderer.

Strużki płyną wzdłuż kierunku spływu. Ich natężenie wynosi `weather.rain × mask.R`. Przy `flow: "slope"` kierunkiem jest rzut światowego wektora grawitacji na powierzchnię, więc spływ odpowiada położeniu instancji. Tekstura kierunku koduje w RG jednostkowy kierunek w przestrzeni UV jako `0.5 + 0.5 × kierunek`. Na powierzchni o nachyleniu mniejszym niż `min_slope` strużki nie powstają, a zmoczenie działa nadal. Renderer może zwiększać prędkość wraz z nachyleniem.

Śnieg pokrywa fragmenty o nachyleniu nie większym niż `max_slope`, z udziałem `weather.snow_cover × snow.mask.R`. Ciepłe fragmenty (4.11.1) są wyłączone z pokrycia. Wygląd śniegu, kropli i strużek dobiera renderer.

Efekty pogodowe są nakładane w kolejności: zmoczenie → strużki → śnieg. Działają na wynikowych wartościach materiału po animacjach `KHR_animation_pointer` i nie są zapisem właściwości w rozumieniu 4.9.1. Ich stan zależy od kanałów instancji, więc współdzielony materiał podlega regule stanu instancji z 4.9.1.

#### 4.10.3. Odprowadzanie wody

Okap, z którego kapie woda, opisuje `node.extras.drip_line`. Wylot rynny lub rury spustowej opisuje `node.extras.water_outlet`. Powierzchnię zbierającą wodę podaje `catchment_m2` (rzut poziomy w m²). Przepływ wynosi `Q = 20 × x × catchment_m2 / 3 600 000` m³/s, gdzie `x` to wartość kanału `source`.

```json
{
  "semantic_id": "building.part_0.roof",
  "drip_line": {
    "edges": [
      {"a": [-5.0, 3.1, 4.2], "b": [5.0, 3.1, 4.2], "outward": [0.0, 0.0, 1.0]}
    ],
    "catchment_m2": 42.0
  }
}
```

```json
{
  "semantic_id": "building.part_0.downspout.0",
  "water_outlet": {
    "catchment_m2": 42.0,
    "diameter": 0.08,
    "speed": 0.5
  }
}
```

| Klucz w `drip_line` | Opisana wartość | Przykład | Domyślna wartość | Wymaganie |
| --- | --- | --- | --- | --- |
| `edges` | Odcinki okapu w układzie węzła: końce `a`, `b` i poziomy jednostkowy kierunek spływu `outward`. | jak wyżej | — | Wymagane |
| `catchment_m2` | Powierzchnia dachu odprowadzana przez całą linię. | `42.0` | — | Wymagane |
| `material` | Referencja materiału kropli. | `"effects.drop"` | materiał renderera | Opcjonalne |
| `source` | Kanał [0,1] natężenia. | `"weather.rain"` | `"weather.rain"` | Opcjonalne |

| Klucz w `water_outlet` | Opisana wartość | Przykład | Domyślna wartość | Wymaganie |
| --- | --- | --- | --- | --- |
| `catchment_m2` | Powierzchnia dachu odprowadzana do wylotu. | `42.0` | — | Wymagane |
| `diameter` | Średnica wylotu w metrach. | `0.08` | `0.08` | Opcjonalne |
| `speed` | Prędkość początkowa wody w m/s. | `0.5` | `0.5` | Opcjonalne |
| `material` | Referencja materiału strumienia. | `"effects.water"` | materiał renderera | Opcjonalne |
| `source` | Kanał [0,1] natężenia. | `"weather.rain"` | `"weather.rain"` | Opcjonalne |

Przepływ linii okapu rozkłada się równomiernie na długość jej odcinków. Krople odrywają się na krawędzi w kierunku `outward`. Okap z rynną nie należy do `drip_line`; wodę z niego odprowadza `water_outlet`. Wylot wypuszcza strumień wzdłuż lokalnego +Y węzła, jak emiter z 4.8. Ruch kropli i strumienia uwzględnia grawitację i `weather.wind`, a ich kontakt z terenem obsługuje świat. Ukrycie węzła przez `KHR_node_visibility` wyłącza odprowadzanie.

#### 4.10.4. Zakres modelu

Model nie opisuje spadającego opadu, rozbryzgów ani śladów na terenie wokół siebie; robi to świat. Model nie zawiera kodu shaderów, także dla wybranego silnika, ani stałego natężenia opadu. Nowy rodzaj reakcji na pogodę wchodzi do profilu jako nowy klucz z parametrami.

### 4.11. Ciepło (DRAFT)

Model może opisać temperaturę własnych powierzchni oraz ciepło oddawane do otoczenia. Temperatury są w stopniach Celsjusza, a temperaturę powietrza `T_air` podaje kanał `weather.temperature`. Model nie symuluje wymiany ciepła: opisuje źródła, a stan otoczenia oblicza świat.

#### 4.11.1. Temperatura powierzchni

Opcjonalne `material.extras.heat` opisuje ciepłe miejsca materiału, np. komin, ścianę kotłowni albo pokrywę silnika:

```json
{
  "semantic_id": "building.chimney_wall",
  "heat": {
    "temperature": 45.0,
    "mask": {"index": 6},
    "source": "building.heating.level",
    "default": 1.0
  }
}
```

| Klucz w `heat` | Opisana wartość | Przykład | Domyślna wartość | Wymaganie |
| --- | --- | --- | --- | --- |
| `temperature` | Temperatura powierzchni przy pełnym ogrzaniu, °C. | `45.0` | — | Wymagane |
| `mask` | Tekstura maski: R udział ciepła w danym miejscu. | `{"index": 6}` | R = 1 | Opcjonalne |
| `source` | Kanał [0,1] poziomu ogrzania. | `"building.heating.level"` | — | Opcjonalne |
| `default` | Poziom ogrzania przed pierwszą aktualizacją kanału albo bez `source`. | `1.0` | `1.0` | Opcjonalne |

Temperatura fragmentu wynosi `T = T_air + mask.R × x × (temperature − T_air)`, gdzie `x` to wartość kanału `source`. Maska jest próbkowana w przestrzeni liniowej. Śnieg z 4.10.2 nie zalega na fragmentach cieplejszych niż 0 °C; szerokość przejścia dobiera renderer. Renderer może też zmniejszać zmoczenie ciepłych fragmentów.

#### 4.11.2. Źródła ciepła w otoczeniu

`node.extras.heat_source` opisuje obszar, w którym model zmienia temperaturę otoczenia: gruntu, powietrza i sąsiednich obiektów. Kształt leży w układzie węzła i porusza się razem z nim.

```json
{
  "semantic_id": "building.boiler_room",
  "heat_source": {
    "shape": "box",
    "size": [4.0, 3.0, 5.0],
    "falloff": 2.5,
    "temperature": 25.0,
    "source": "building.heating.level",
    "default": 1.0
  }
}
```

| Klucz w `heat_source` | Opisana wartość | Przykład | Domyślna wartość | Wymaganie |
| --- | --- | --- | --- | --- |
| `shape` | Kształt obszaru: `"sphere"` albo `"box"`. | `"box"` | — | Wymagane |
| `radius` | Promień kuli w metrach. | `2.0` | — | Wymagane dla `sphere` |
| `size` | Pełne wymiary prostopadłościanu wzdłuż osi węzła, w metrach. | `[4.0, 3.0, 5.0]` | — | Wymagane dla `box` |
| `falloff` | Szerokość miękkiej krawędzi w metrach. | `2.5` | `1.0` | Opcjonalne |
| `temperature` | Temperatura wnętrza obszaru przy pełnym ogrzaniu, °C. | `25.0` | — | Wymagane |
| `source` | Kanał [0,1] poziomu ogrzania. | `"building.heating.level"` | — | Opcjonalne |
| `default` | Poziom ogrzania przed pierwszą aktualizacją kanału albo bez `source`. | `1.0` | `1.0` | Opcjonalne |

Waga wpływu wynosi 1 wewnątrz kształtu i maleje płynnie (`smoothstep`) do 0 w odległości `falloff` od jego powierzchni. Lokalna temperatura otoczenia to `T = T_air + Σ waga × x × (temperature − T_air)`, ograniczona do przedziału wyznaczonego przez `T_air` i temperatury źródeł. Temperatura poniżej `T_air` opisuje źródło chłodu. Świat może przybliżać źródła na własnej siatce próbkowania, której rozdzielczość dobiera; wynik wpływa na śnieg i schnięcie w otoczeniu tak jak temperatura powietrza. Ukrycie węzła przez `KHR_node_visibility` wyłącza źródło. Modele dynamiczne używają tych samych pól, a poziom ogrzania podaje symulacja, np. kanałem `engine.heat`.

## 5. Eksport i walidacja

1. Autor przygotowuje geometrię, materiały, światła, pivoty i klipy w narzędziu.
2. Eksport glTF obejmuje światła, animacje i własne właściwości użyte przez profil. Wypiekanie obejmuje wynik więzów i odwrotnej kinematyki (IK).
3. Etap pakowania normalizuje metadane do miejsc określonych w tej specyfikacji, nadaje stabilne identyfikatory i aktualizuje referencje. Ustawienia eksporterów i umiejscowienie własnych właściwości sprawdza się dla używanej wersji narzędzia.
4. Walidator glTF sprawdza standardową strukturę; walidator profilu sprawdza kontrakt modelu.
5. Test importu sprawdza scenę, skalę, emisję, kanał alfa, światła i reprezentatywne próbki klipów. Obsługę każdego użytego rozszerzenia weryfikuje się na eksportowanym pliku.

Walidacja profilu sprawdza:

- nagłówek, wersję, jednostkowy korzeń i rozwiązywalność zasobów;
- unikalność ID oraz typ i cel każdej referencji;
- zakresy wartości w opcjonalnym `geo`;
- typy kanałów, zakresy liczb i pojedynczego autora właściwości;
- przypisanie `light_control` do węzła ze światłem;
- wartość `light_state` oraz liczbową wartość `light_threshold`;
- istnienie materiału emitera i dodatnie limity jego życia;
- zakresy pól `material.extras.weather`: `porosity` i `roughness` w [0,1], `min_slope` i `max_slope` w [0,π/2], istnienie wskazanych tekstur;
- niepuste `edges` w `drip_line`, poziome jednostkowe `outward`, dodatnie `catchment_m2` w `drip_line` i `water_outlet`;
- skończone `temperature` w `heat` i `heat_source`, `shape` zgodny z `radius` albo `size` o dodatnich wartościach, nieujemne `falloff`;
- deklaracje rozszerzeń, ich ograniczenia i możliwości importera.

Wersja profilu jest niezależna od `asset.version`. `PATCH` doprecyzowuje kontrakt, `MINOR` dodaje opcjonalne możliwości, a `MAJOR` zmienia interpretację danych. W fazie 0.x importer jawnie negocjuje każdą wersję.

## 6. Źródła

- [glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html).
- [Rejestr rozszerzeń](https://github.com/KhronosGroup/glTF/blob/main/extensions/README.md).
- [Eksporter świateł Blendera](https://github.com/blender/blender-addons/blob/main/io_scene_gltf2/blender/exp/gltf2_blender_gather_lights.py) — przykład implementacji konwersji; proces przetwarzania przypina konkretną wersję eksportera.
