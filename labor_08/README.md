# Labor 08: Szenzortípusok és Alkalmazásuk (I2C Szenzorok)
**Tananyag:** Kristálytiszta Elektronika 2 — 13. fejezet (Szenzortípusok és alkalmazásuk)

---

## 🎯 A Laboratórium Célja
Különböző fizikai mérési elvek és szenzortípusok (MEMS nyomás- és gyorsulásmérők, fotoelektromos és hőmérséklet érzékelők) megismerése, valamint a panelen található 3 I2C szenzor (APDS9300 fénymérő, LM75A hőmérő és HIH8120 páratartalom-mérő) kezelése és az eredmények OLED kijelzőn történő megjelenítése.

---

## 📋 Elvégzendő Feladatok
- [ ] **Szenzortechnológiai alapfogalmak:**
  - Passzív és aktív szenzorok, analóg és digitális interfészek, statikus átviteli karakterisztika (érzékenység, linearitás, hiszterézis, telítés).
  - MEMS szenzorok felépítése (nyomásmérő, gyorsulásmérő), fotoelektromos eszközök (fotodióda, fototranzisztor).
- [ ] **LM75A digitális hőmérő szenzor kezelése (`13_fejezet_temperature_sensor`):**
  - Hőmérséklet kiolvasása a `TEMPERATURE` regiszterből 11 bites kettes komplemens formátumban (0.125 °C felbontás).
  - Túlhőmérséklet (`TOS`) és hiszterézis (`THYST`) határértékek felprogramozása (0.5 °C felbontás).
  - Az OS kimenet és LED riasztásának tesztelése (lehelettel kiváltott túllépés).
  - Mért hőmérséklet és fok szimbólum kirajzolása az I2C OLED kijelzőre.
- [ ] **HIH8120 páratartalom- és hőmérséklet szenzor kezelése:**
  - 4 bájtos mérési adatfolyam beolvasása I2C-n (`HumTempSensor_GetValues()`).
  - Státuszbitek kiértékelése, relatív páratartalom (%) és hőmérséklet (°C) átszámítása a gyári képletek alapján.
- [ ] **Gyakorló feladat: Összetett környezetmonitorozó:**
  - A panelen lévő mindhárom I2C szenzor (APDS9300 fénymérő, LM75A hőmérő, HIH8120 páratartalom-mérő) együttes kiolvasása és értékeik formázott megjelenítése a grafikus OLED kijelzőn.

---

## 🔌 Hardver Összeállítás

### 1. Kristálytiszta Elektronika Fejlesztői Panel (STM32F405VGT6)
A tananyaghoz készült hivatalos oktatópanelen a három I2C szenzor és az OLED kijelző közvetlenül a nyomtatott áramköri lapra van integrálva, külön huzalozást nem igényelnek:
* **I2C busz:** `I2C1` (`PB6` = SCL, `PB7` = SDA)
* **APDS9300 (fénymérő):** `I2C1`, cím: `0x39` (`LightSensor_ADDR_FLOAT`), INT: `PB8`
* **LM75A (hőmérő):** `I2C1`, cím: `0x48`, OS riasztási kimenet a panel túlhőmérséklet-jelző LED-jére kötve
* **HIH8120 (páratartalom- és hőmérséklet-mérő):** `I2C1`, cím: `0x27` (fejlécben `0x4E`)
* **OLED kijelző (SSD1306):** `I2C1`, cím: `0x3C` (`0x78`)
* **Példaprogram csomag:** `lecture_13_Panel_F405.zip` — kicsomagolva öt projekt: `13_fejezet_temperature_sensor/`, `13_fejezet_humidity_sensor_magyar/`, `13_fejezet_light_sensor/`, `13_fejezet_all_i2c_sensors_magyar/`, valamint a panelre igazított, kijelzős fénymérő példa: `lecture_8_Panel_F405/`

---

### 2. STM32 Nucleo-F446ZE Próbapaneles (Breadboard) Kiépítés

A panelen található APDS9300 fénymérő, LM75A hőmérő és HIH8120 páratartalom-mérő modul egyetlen, közös **I2C2** buszra van kötve: mindhárom modul SDA, illetve SCL lába ugyanabba a próbapanel-oszlopba csatlakozik, ahonnan egy-egy jumper vezet az STM32 megfelelő lábára. A modulok VCC/GND lába a próbapanel piros (+3.3V) és kék (GND) tápsínjére kerül.

> 🎮 **[👉 Interaktív Próbapanel Szimulátor megnyitása (breadboard.html)](breadboard.html)** — a közös I2C busz és a 3 szenzormodul pontos elhelyezési rajza, élő szimulációs mód a fénymérő/hőmérő/páratartalom-mérő kiolvasásához.

| Alkatrész | STM32 Pin | C Makró | Próbapanel |
| :--- | :--- | :--- | :--- |
| **APDS9300 (fénymérő)** | `PF0` / `PF1` | `I2C2_SDA` / `I2C2_SCL` | VCC/GND sín + közös SDA/SCL busz-oszlop |
| **LM75A (hőmérő)** | `PF0` / `PF1` | `I2C2_SDA` / `I2C2_SCL` | VCC/GND sín + közös SDA/SCL busz-oszlop |
| **HIH8120 (páratartalom-mérő)** | `PF0` / `PF1` | `I2C2_SDA` / `I2C2_SCL` | VCC/GND sín + közös SDA/SCL busz-oszlop |
| **Fénymérő INT kimenet** *(opcionális)* | `PF8` | `LightSensor_Int` (EXTI8) | Csak vezetékezési lehetőség, jelen gyakorlat nem használja szoftveresen |
| **Közös GND / VCC** | `GND` / `3.3V` | — | Próbapanel kék (-) / piros (+) sínje |

**I2C busz:** mindhárom szenzor ugyanazt a két vezetéket (SDA, SCL) osztja meg egymással és az STM32-vel — csak a VCC/GND, illetve az I2C-címek térnek el modulonként.

> [!CAUTION]
> A szenzormodulok VCC lába kizárólag `3.3V`-ra köthető — `5V`-ra kötve a chip tartósan károsodhat. Ügyelj rá, hogy egyik szenzor SDA/SCL lába se maradjon lógva vagy legyen felcserélve a busz oszlopában.

---

## 🚀 Teendők a Laboron (Lépésről lépésre)
1. Tanulmányozd át a mellékelt tananyag PDF-et (`13. Szenzortípusok és alkalmazásuk.pdf`)!
2. Indítsd el az **STM32CubeIDE**-t, és nyisd meg a `13_fejezet_temperature_sensor` példaprojektet!
3. Figyeld meg az LM75A regisztereinek kezelését és az OLED kijelzőre való kiíratást!
4. Állítsd be a `THYST` és `TOS` értékeket úgy, hogy melegítésre bekapcsoljon az OS riasztási LED!
5. Valósítsd meg a gyakorló feladatot: olvasd ki a fénymérőt és a páratartalom-mérőt is, és jelenítsd meg mindhárom adatot az OLED képernyőn!
6. A labor végén töltsd fel a megoldásodat a saját privát GitHub repódba a heti érdemjegyedért:
   ```bash
   git add .
   git commit -m "labor_08 harom i2c szenzor es oled megjelenites kesz"
   git push origin main
   ```
