# Modele statyczne — profil glTF

Status: projekt specyfikacji 0.3.0. Format bazowy: glTF 2.0.

## 1. Zakres i reguły wspólne

Profil opisuje modele związane z miejscem w świecie: budynki, perony, słupy, sygnalizatory, roślinność i wyposażenie. Model statyczny może zawierać ruchome części, światła, wyświetlacze i emitery dymu. `model_class` określa sposób umieszczania i zarządzania instancją.

Sekcje 2–11 stanowią reguły wspólne dla obu profili. [Profil dynamiczny](DYNAMIC_MODEL_FORMAT.md) stosuje je z `model_class: "dynamic"` i dodaje kontrakt pojazdu.

MUST oznacza wymaganie, SHOULD zalecenie, MAY możliwość. Własne pola zdefiniowane tutaj znajdują się bezpośrednio w `extras` właściwego obiektu. Klucze opisują konkretne dane lub zachowania. Standardowe dane glTF są źródłem wartości geometrii, materiałów, świateł i animacji.

Dokument definiuje kontrakt docelowy. Zgodność implementacji jest sprawdzana dla wersji wskazanej w `model_format_version`. Obsługa rozszerzenia w formacie, eksporterze i rendererze jest weryfikowana oddzielnie podczas integracji.

## 2. Kontener, układ i zasoby

Asset MUST być poprawnym glTF 2.0 zapisanym jako `.gltf` z zasobami albo `.glb`. Zewnętrzne URI zasobów są względne wobec pliku glTF. Pakiet MUST zawierać wszystkie zależności. GLB może również mieć zależności zewnętrzne, które pakowarka rozwiązuje tak samo.

Profil przyjmuje metry, sekundy i radiany. Model ma jedną domyślną scenę z jednym korzeniem reprezentującym cały obiekt. Układ assetu: +Y w górę, +Z przód obiektu, +X w lewo z perspektywy patrzącego w kierunku +Z. Eksporter przelicza układ narzędzia autorskiego do tego układu.

Transform korzenia w pliku jest jednostkowy. Transform umieszczenia instancji przechowuje świat. Światła kierunkowe i kamery zachowują własne konwencje osi glTF. Ruchome node'y używają TRS z prawidłowo ustawionym pivotem. Skala mechanizmów jest wypieczona przed eksportem.

Geometria, hierarchia, skiny, morphy i klipy korzystają z odpowiednich obiektów [glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html). Nazwy `name` służą prezentacji w narzędziach.

## 3. Standardowe reprezentacje

| Dane autora | Reprezentacja |
|---|---|
| Mesh i materiały PBR | `meshes`, `materials`, `textures` |
| Transform i hierarchia | `nodes` |
| Ruch / obrót / skala | `animations` z kanałami TRS |
| Szkielet i shape keys | `skins`, jointy, morph targets i animowane `weights` |
| Kamera | `cameras` przypięte do node'a |
| Spotlight / omni / słońce | `KHR_lights_punctual`: `spot` / `point` / `directional` |
| Silna emisja powierzchni | `emissiveFactor`, `emissiveTexture`, `KHR_materials_emissive_strength` |
| Parametry zmieniające się w czasie | `KHR_animation_pointer` |
| Widoczność gałęzi | `KHR_node_visibility` |
| Zestaw wariantów materiałowych | `KHR_materials_variants` |

Rozszerzenia użyte w assetcie MUST wystąpić w `extensionsUsed`. Rozszerzenia niezbędne do poprawnej interpretacji tego assetu MUST dodatkowo wystąpić w `extensionsRequired`; importer sprawdza je przed utworzeniem instancji. Gdy warianty geometrii polegają na ukrywaniu części, `KHR_node_visibility` jest wymagane. Gdy sterowanie wymaga kanałów pointer, `KHR_animation_pointer` jest wymagane.

Poniższe rozszerzenia mogą być dobierane do materiałów i dystrybucji zasobów:

| Potrzeba | Rozszerzenie |
|---|---|
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

## 4. Nagłówek i identyfikatory

Przykład obiektu `asset`:

```json
{
  "version": "2.0",
  "extras": {
    "model_format_version": "0.3.0",
    "model_class": "static",
    "model_type": "building",
    "model_id": "station.example.building",
    "bounds": {
      "min": [-12.0, 0.0, -4.0],
      "max": [12.0, 9.0, 4.0],
      "size": [24.0, 9.0, 8.0]
    }
  }
}
```

Wszystkie pięć pól nagłówka jest wymaganych. `model_type` przyjmuje `building`, `infrastructure`, `vegetation`, `terrain`, `prop`; profil dynamiczny rozszerza katalog. `model_id` identyfikuje asset w datapacku; instancja otrzymuje osobny identyfikator świata.

`extras.semantic_id` na node'ach, materiałach i animacjach identyfikuje elementy używane przez symulację. Wartości MUST być unikalne w obrębie danej kolekcji assetu i stabilne przy ponownym eksporcie. Referencja ma typ wynikający z pola: `node` wskazuje node, `material` materiał, `animation` klip. Wartością jest zawsze `semantic_id` docelowego obiektu.

Identyfikator to segmenty rozdzielone kropką, zgodne z `[a-z0-9_]+`, np. `building.clock.minute`, `infrastructure.signal.0.red`, `vehicle.door.left.0`. Zarezerwowane znaczenia konkretnych ścieżek definiują profile. Pozostałe ścieżki są stabilnymi adresami autora. Indeksy porządkowe zaczynają się od zera.

Standardowe referencje glTF i JSON Pointer używają indeksów zgodnie ze standardem. Eksporter MUST aktualizować je podczas przebudowy tablic. Identyfikatory semantyczne służą wiązaniu modelu z konfiguracją i stanem świata.

## 5. Granice, rozmiary i rzędne

`asset.extras.bounds` MUST zawierać `min`, `max`, `size`: trzy skończone liczby w kolejności X, Y, Z. `size = max - min`, porównywane z tolerancją 0.001 m. Granice są liczone w układzie assetu, po zastosowaniu hierarchii transformacji, dla geometrii domyślnego stanu. Uwzględniają skinning i morphy tego stanu.

Opcjonalne `motion_bounds` ma tę samą strukturę i obejmuje sumę przestrzeni zajmowanej przez dozwolone animacje i mechanizmy. Jest wymagane dla części wychodzących poza `bounds`. Autor lub eksporter wyznacza konserwatywną obwiednię również między klatkami. Zasięgi świateł i cząstek służą osobnym obliczeniom cullingu efektów.

Opcjonalne `ground_level` to lokalna rzędna Y posadowienia. Globalna rzędna pomiarowa należy do georeferencji.

## 6. Georeferencja i pochodzenie danych

Opcjonalne `asset.extras.georeference` wiąże punkt assetu z pomiarem terenowym. Przykład zawartości `asset.extras`:

```json
{
  "georeference": {
    "horizontal_crs": "EPSG:2180",
    "axis_order": ["easting", "northing"],
    "position": [637421.32, 486205.77],
    "vertical_crs": "PL-EVRF2007-NH",
    "elevation": 112.48,
    "anchor": [0.0, 0.0, 0.0],
    "heading": 0.0,
    "horizontal_accuracy": 0.25,
    "vertical_accuracy": 0.15,
    "source": "survey.0"
  },
  "data_sources": [
    {
      "id": "survey.0",
      "provider": "Geoportal Krajowy",
      "dataset": "identyfikator warstwy",
      "object_id": "identyfikator obiektu",
      "uri": "https://example.org/dataset/record",
      "retrieved_at": "2026-10-19",
      "license": "identyfikator licencji",
      "used_for": ["position", "elevation", "bounds"]
    }
  ]
}
```

Przykład zawiera ilustracyjne pomiary i adres źródła. W realnym assetcie autor zapisuje dane pomiarowe i licencję właściwego zbioru.

W georeferencji wymagane są `horizontal_crs`, `axis_order`, `position`, `anchor`, `heading`. Współrzędne poziome używają jednostek CRS oraz jawnej kolejności osi. Gdy występuje `elevation`, wymagane jest `vertical_crs`, którego jednostką wysokości dla tego profilu jest metr. `horizontal_accuracy` i `vertical_accuracy` to opcjonalne nieujemne dokładności w metrach.

`anchor` jest punktem lokalnym glTF. `heading` określa kierunek lokalnego +Z, zgodnie z ruchem wskazówek zegara od północy w lokalnym układzie geodezyjnym ENU. Dla heading 0: +Z wskazuje północ, +X zachód, +Y górę. Położenie przelicza importer CRS do lokalnego świata; duże współrzędne geodezyjne pozostają w nagłówku.

`source` wskazuje `id` wpisu `data_sources`. Każdy wpis wymaga `id`, `provider`, `dataset`, `retrieved_at`, `used_for`; pozostałe pola są opcjonalne. Różne wpisy mogą dokumentować pomiar bryły, rzędną, fotografie i materiały.

## 7. Materiały, przezroczystość i warianty

| Zastosowanie | Zapis |
|---|---|
| Pełna powierzchnia | `alphaMode: OPAQUE` |
| Alpha scissor: siatka, liść, ogrodzenie | `alphaMode: MASK`, próg `alphaCutoff` |
| Mieszanie alpha | `alphaMode: BLEND` |
| Szkło optyczne | PBR z rozszerzeniami transmission / volume / IOR |
| Powierzchnia dwustronna | `doubleSided` |

Eksporter dobiera tryb do zamierzonego efektu. Przezroczystość pokrycia i transmisja optyczna są niezależnymi własnościami materiału. Szyba z zabrudzeniami może łączyć odpowiednio przygotowane warstwy.

Warianty zestawu materiałów, np. elewacje lub malowania, używają [KHR_materials_variants](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_materials_variants). Nazwa wariantu jest stabilnym kluczem wyboru w konfiguracji instancji; mapowania przypisują materiały primitive'om. Wariant bazowy korzysta z `primitive.material`.

Wymienny materiał z datapacku oznacza `material.extras.material_slot`, np. `building.facade`. Konfiguracja instancji mapuje ten klucz na zasób materiału. Podmiana zachowuje kontrakt UV i atrybutów siatki.

Emisja powierzchni lampy jest zapisana w materiale, z opcjonalnym [KHR_materials_emissive_strength](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_materials_emissive_strength). Oświetlenie otoczenia przypisuje się światłu powiązanemu z odpowiednim node'em.

## 8. Światła

Światła zapisuje [KHR_lights_punctual](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_lights_punctual). Eksport Blendera mapuje `POINT` na `point`, `SPOT` na `spot`, `SUN` na `directional`. Eksporter odpowiada za przeliczenie jednostek autora.

Definicja zawiera `type`, `color`, `intensity`, opcjonalne `range` oraz kąty `spot.innerConeAngle` i `spot.outerConeAngle`. `point` i `spot` używają kandeli, `directional` luksów. Transform node'a ustala pozycję i orientację; światło `spot` świeci w lokalnym kierunku −Z. Pominięte `range` oznacza nieskończony zasięg zgodnie z rozszerzeniem.

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

To przykład mechanizmu światła; nagłówek profilu dodaje się według sekcji 4. Zwykła lampa działa po załadowaniu tych danych. Sterowanie scenariuszem dopina do node'a opcjonalne `light_control`:

```json
{
  "light_control": {
    "source": "lighting.platform.0.level",
    "default": 1.0
  }
}
```

`source` wskazuje kanał liczbowy instancji w zakresie [0,1]. Wartość domyślna jest wymagana. Runtime mnoży bazowe `intensity` światła przez wartość kanału. Regułę zmierzchu lub harmonogram dostarcza świat. Wspólna definicja światła jest instancjonowana z oddzielnym stanem dla każdego sterowanego node'a.

Powierzchnia lampy może mieć na materiale `emission_control` o tych samych polach; kanał mnoży bazowy `emissiveStrength` (domyślnie 1). Materiał używa rozszerzenia emissive strength, jeśli jego wynik wymaga wartości ponad 1.

Profile fotometryczne zapisuje opcjonalne [EXT_lights_ies](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/EXT_lights_ies); zasoby środowiskowego oświetlenia opisuje [EXT_lights_image_based](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Vendor/EXT_lights_image_based). Ich użycie wymaga zadeklarowanego wsparcia w pipeline.

## 9. Animacje, widoczność i sterowanie

Klip przechowuje animację przygotowaną przez autora. Wynik constraints i IK obliczonych w narzędziu jest wypiekany do kanałów. Animowanie parametrów materiałów i świateł używa [KHR_animation_pointer](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer).

Klip wybierany przez świat MUST mieć `animation.extras.semantic_id`. Opcjonalne `animation.extras.playback` definiuje jego sterowanie:

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

Tryb `loop` używa `source` jako bool uruchomienia, `default` jako bool, `time` jako zakres klipu, opcjonalnego `speed` > 0 (domyślnie 1). Zbocze false→true rozpoczyna od t0, true odtwarza cyklicznie, false zatrzymuje na t0. Czas pochodzi z zegara symulacji. Stałą pętlę autor zapisuje przez pominięcie `source` i `default: true`.

Widoczność i jej animacja używają [KHR_node_visibility](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_node_visibility). Domyślne `visible` zapisuje rozszerzenie node'a. Kanał boolean może sterować nim przez `node.extras.visibility_control`:

```json
{
  "visibility_control": {
    "source": "season",
    "equals": "winter"
  }
}
```

Przy `equals` wynik jest ścisłym porównaniem wartości kanału z literałem. Przy samym `source` wymagany jest kanał bool. Przed pierwszą aktualizacją kanału obowiązuje `visible` z rozszerzenia. Taki zapis obsługuje warianty sezonowe i wygląd zależny od sygnału.

Zegar może używać klipów `position` próbkowanych kanałami `clock.second`, `clock.minute`, `clock.hour12`, `clock.hour24`. Świat dostarcza liczby z częścią ułamkową; autor ustala zakres i interpolację klipu. Zaprogramowane mruganie może używać klipu `loop`, podczas gdy aspekt sygnalizatora jest wejściem ze scenariusza.

### 9.1. Kontrakt aktualizacji

Kanał `source` jest nazwą wartości dostarczanej przez świat lub symulację do konkretnej instancji. Konfiguracja instancji MUST rozwiązać wymagane kanały przed aktywacją. Kanały używają bool, liczby albo string zgodnie z konsumentem; kanały celów IK w profilu dynamicznym używają vec3. Liczby są skończone.

Kolejność: odczyt spójnej migawki kanałów → próbkowanie klipów → sterowniki proceduralne → aktualizacja transformacji światowych → światła, widoczność, emisja i cząstki. Każda zapisywana właściwość ma jednego aktywnego autora w danej instancji. Konflikt dwóch klipów lub klipu i sterownika jest błędem konfiguracji. Współdzielone materiały mają stan instancji albo kopię przy zapisie. Odtwarzanie dwóch instancji MUST zachować ich niezależność.

## 10. Dym i inne emitery

Pozycję emitera reprezentuje node. Lokalny +Y określa kierunek początkowy. `node.extras.smoke_source` opisuje konkretny efekt cząstkowy:

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

`cone_angle` to półkąt [0,π], rozkład kierunków jest jednostajny po kącie bryłowym. Parametry w zakresach są losowane jednostajnie i niezależnie przy narodzinach. `growth` jest przyrostem średnicy w m/s; średnica jest ograniczana od dołu zerem. `acceleration` jest wektorem m/s² układu assetu, `drag` ≥ 0 współczynnikiem oporu 1/s; równanie prędkości to `dv/dt = acceleration - drag*v`. `inherit_velocity` ∈ [0,1] mnoży prędkość światową emitera przy narodzinach.

Cząstki żyją w układzie świata i są billboardami skierowanymi do kamery. Przy narodzinach kierunek prędkości jest przeliczany transformem emitera, a przyspieszenie rotacją assetu do układu świata; te wartości światowe są następnie używane przez integrator. Materiał glTF określa teksturę, barwę, emisję i alpha. Alpha jest dodatkowo mnożona przez `1-age/lifetime`. Po osiągnięciu limitu nowe cząstki są pomijane, a po wyłączeniu kanału emisji istniejące kończą życie. Ukrycie node'a przez `KHR_node_visibility` zatrzymuje narodziny i ukrywa cały efekt; wiek cząstek nadal narasta.

## 11. Eksport i walidacja

1. Autor przygotowuje geometrię, materiały, światła, pivoty i klipy w narzędziu.
2. Eksport glTF obejmuje światła, animacje i custom properties użyte przez profil. Wypiekanie obejmuje wynik constraints i IK.
3. Etap pakowania normalizuje metadane do miejsc określonych w tej specyfikacji, nadaje stabilne ID, aktualizuje referencje i wylicza granice. Ustawienia eksporterów i umiejscowienie custom properties sprawdza się dla używanej wersji narzędzia.
4. Walidator glTF sprawdza standardową strukturę; walidator profilu sprawdza kontrakt assetu.
5. Test importu sprawdza scenę, skalę, emisję, alpha, światła i reprezentatywne próbki klipów. Obsługę każdego użytego rozszerzenia weryfikuje się na eksportowanym pliku.

Walidacja profilu MUST sprawdzić:

- nagłówek, wersję, jednostkowy korzeń i rozwiązywalność zasobów;
- unikalność ID oraz typ i cel każdej referencji;
- kompletność granic i poprawność arytmetyczną `size`;
- wymagane pola georeferencji oraz odwołania do źródeł;
- typy kanałów, zakresy liczb i pojedynczego autora właściwości;
- przypięcie `light_control` do node'a ze światłem;
- istnienie materiału emitera i dodatnie limity jego życia;
- deklaracje rozszerzeń, ich ograniczenia i możliwości importera.

Wersja profilu jest niezależna od `asset.version`. PATCH doprecyzowuje kontrakt, MINOR dodaje opcjonalne możliwości, MAJOR zmienia interpretację danych. W fazie 0.x każdą wersję importer negocjuje jawnie.

## 12. Mapowanie modeli statycznych T3D/E3D

| Element legacy | Wynik konwersji |
|---|---|
| Mesh, transform, hierarchia | `mesh` i `node` glTF |
| `Light_On00…07` | Node wizualnej lampy, `visibility_control` kanału lampy; ewentualny klip przejścia alpha |
| `Light_Off00…07` | Node z `visibility_control` tego kanału, `equals: false` |
| `winter_variant`, `spring_variant`, `summer_variant`, `autumn_variant` | `visibility_control` z kanałem `season` i właściwą wartością |
| `seconds`, `minutes`, `hours`, `hours24` | Klip wskazówki sterowany właściwym kanałem zegara |
| Odpowiedniki `*_jump` | Klip o skokowym próbkowaniu wskazówki |
| `smokesource_<template>` | Node emitera z rozwiniętą definicją `smoke_source` i materiałem glTF |
| Materiał wycinany alphą | `MASK` oraz ustalony próg |
| Materiał transparentny | `BLEND` |
| Flagi przebiegu opaque/alpha | Tryb alpha materiału ustalony z parametrów i użycia |
| Flaga geometrii indeksowanej | `primitive.indices` |

Konwerter MUST interpretować `freespotlight` na podstawie typu renderowanego efektu i parametrów: wizualna poświata staje się geometrią z materiałem, a rzeczywiste źródło oświetlenia odpowiednim światłem glTF. Nazywanie obu obiektów lampą w legacy wymaga ich rozróżnienia podczas konwersji.

Parametry legacy oświetlenia i materiałów wymagają kalibracji do jednostek i modelu PBR. Raport konwersji wskazuje przyjęty próg alpha, konwersję osi i przybliżenia. Dokładne dwufazowe renderowanie legacy wymaga przetworzenia materiału/geometrii podczas migracji; docelowe powierzchnie otrzymują jawny tryb alpha.

## 13. Źródła

- [glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html).
- [Rejestr rozszerzeń](https://github.com/KhronosGroup/glTF/blob/main/extensions/README.md).
- [Eksporter świateł Blendera](https://github.com/blender/blender-addons/blob/main/io_scene_gltf2/blender/exp/gltf2_blender_gather_lights.py) — przykład implementacji konwersji; pipeline przypina konkretną wersję eksportera.
- Źródła mapowania legacy: `model/Model3d.cpp`, `model/AnimModel.cpp`, `rendering/particles.cpp`.
