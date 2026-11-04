# Labor 09: Szenzorok Kalibrálása (Termisztor & Zéruspontos Kalibráció)
**Tananyag:** Kristálytiszta Elektronika 2 — 14. fejezet (Szenzorok kalibrálása)

---

## 🎯 A Laboratórium Célja
A kalibráció definíciójának, a mérési hibák (offset, meredekség) és a termisztoros hőmérsékletmérés alapjainak elsajátítása, feszültségosztó áramkör mérése 12 bites ADC-vel, valamint egypontos (zéruspontos) szoftveres kalibrációs állapotgép megvalósítása 0 °C-os jégvíz referenciával és OLED kijelzővel.

---

## 📋 Elvégzendő Feladatok
- [ ] **A kalibráció elméleti háttere:**
  - A kalibráció fogalma (etalonhoz hasonlítás, mérési hiba összefüggésének feltárása).
  - Szisztematikus és véletlen hibák: zérushiba (offset), meredekséghiba (gain), nemlinearitás.
  - Hőmérséklet-érzékelők: NTC vs. PTC termisztorok, nemlineáris karakterisztika, a Steinhart-Hart egyenlet és együtthatóinak (A, B, C) meghatározása három ellenállás-hőmérséklet értékpárból.
- [ ] **Mérési elrendezés és ellenállás-számítás:**
  - Vishay NTCLE400E3103H NTC termisztor ($R_{25} = 10\text{ k}\Omega$, B=3977 K) és $R_2 = 10\text{ k}\Omega$ referencia-ellenállás feszültségosztójának vizsgálata.
  - Feszültségosztó kimenetének beolvasása az STM32 ADC1 perifériájával (`SEN` címke; Panel: `PA1`, `ADC1_IN1`; Nucleo: `PA3`, `ADC1_IN3`) 12 bites felbontásban.
  - Az NTC pillanatnyi ellenállásának kiszámítása: $R_1 = (\frac{4095}{ADC} - 1) \cdot 10000\text{ }\Omega$.
- [ ] **Zéruspontos (egypontos) kalibrációs állapotgép megvalósítása:**
  - Szoftveres állapotgép működtetése nyomógombokkal és OLED kijelzővel:
    1. `SCRN_Start` és `SCRN_ADC` képernyők közötti váltás a fel (BTN1) / le (BTN5) gombbal.
    2. `DM_Temp`, `DM_Diff`, `DM_ConvRes`: a korrigált hőmérséklet, a tárolt kalibrációs eltolás, illetve a nyers ADC eredmény megjelenítési módjai közötti váltás a bal (BTN2) / jobb (BTN4) gombbal.
    3. Kalibráció indítása: a termisztor 0 °C-os jég-víz keverékbe merítése, majd `DM_Temp` módban a középső (BTN3) gomb megnyomásával a mért és a valós (0 °C) hőmérséklet különbségének (`temperature_correction`) eltárolása.
    4. A tárolt kalibrációs eltolás törlése `DM_Diff` módban a középső (BTN3) gomb megnyomásával.
- [ ] **Zajcsökkentés számtani középérték-számítással:**
  - A mintavételezési zaj elnyomása `SAMPLE_NUMBER` számú egymást követő ADC mérés összegzésével és átlagolásával.

---

## 🔌 Hardver Összeállítás

### 1. Kristálytiszta Elektronika Fejlesztői Panel (STM32F405VGT6)
A tananyaghoz készült hivatalos oktatópanelen az analóg termisztor-bemenet, az 5 navigációs nyomógomb és az OLED kijelző közvetlenül a nyomtatott áramköri lapra van integrálva:
* **Analóg mérőbemenet:** `PA1` (`ADC1_IN1`, `SEN` címke a sorkapocsnál)
* **Referencia-ellenállás:** $10\,\text{k}\Omega$ a mérőpont és a GND között a panelen
* **OLED kijelző (SSD1306):** `I2C1` (`PB6` = SCL, `PB7` = SDA)
* **Képernyőváltás (fel / le):** `PD8` (`BTN1_Pin`) / `PB12` (`BTN5_Pin`)
* **Megjelenítési mód (bal / jobb):** `PB15` (`BTN2_Pin`) / `PB13` (`BTN4_Pin`)
* **Kalibráció indítása / törlése (középső):** `PB14` (`BTN3_Pin`)
* **Példaprogram csomag:** `lecture_14_Panel_F405.zip` (kicsomagolva: `14_fejezet_kalibralas_peldakod/`)

---

### 2. STM32 Nucleo-F446ZE Próbapaneles (Breadboard) Kiépítés

A termisztor és a $10\,\text{k}\Omega$-os precíziós referencia-ellenállás sorba kötve feszültségosztót alkot a `3.3V` és `GND` sín között; a két alkatrész közös pontja (az osztó kimenete) adja az STM32 analóg bemenetére kötött mérőfeszültséget. Az OLED kijelző a közös **I2C2** buszra, a kalibrációs állapotgépet vezérlő 5 nyomógomb pedig a HMI panel gombjaira csatlakozik.

> 🎮 **[👉 Interaktív Próbapanel Szimulátor megnyitása (breadboard.html)](breadboard.html)** — a feszültségosztó pontos elhelyezési rajza, élő hőmérséklet-csúszka a nyers ADC érték és a számított hőmérséklet szemléltetésére, valamint kalibráló gomb a zéruspontos eltolás beállításához.

| Alkatrész | STM32 Pin | C Makró | Próbapanel |
| :--- | :--- | :--- | :--- |
| **NTC termisztor + $10\,\text{k}\Omega$ referencia (feszültségosztó)** | `PA3` (`ADC1_IN3`) | `SEN` | Osztó középpontja → `PA3`; felső/alsó láb → `3.3V` / `GND` sín |
| **OLED kijelző (SSD1306)** | `PF0` / `PF1` | `I2C2_SDA` / `I2C2_SCL` | VCC/GND sín + közös I2C busz-oszlop |
| **Képernyőváltás (fel / le)** | `PE5` / `PE4` | `HMI_BTN_1_Pin` / `HMI_BTN_5_Pin` | HMI panel nyomógombjai |
| **Megjelenítési mód (bal / jobb)** | `PE2` / `PE6` | `HMI_BTN_2_Pin` / `HMI_BTN_4_Pin` | HMI panel nyomógombjai |
| **Kalibráció indítása / törlése (középső)** | `PE3` | `HMI_BTN_3_Pin` | HMI panel nyomógombja |
| **Közös GND / VCC** | `GND` / `3.3V` | — | Próbapanel kék (-) / piros (+) sínje |

**Feszültségosztó iránya:** a termisztor csökkenő ellenállása (melegedéskor) az osztó középpontján mért feszültséget megemeli — az `ADC` érték ebből számolható vissza a $R_1 = (\frac{4095}{ADC} - 1) \cdot 10\,000\,\Omega$ képlettel.

> [!CAUTION]
> A jégvizes zéruspontos kalibrációnál csak a termisztor érzékelő fejét mártsd a vízbe — a próbapanel, a jumperek és az STM32 kártya vízmentesen, az edénytől távol maradjanak!

---

## 🚀 Teendők a Laboron (Lépésről lépésre)
1. Tanulmányozd át a mellékelt tananyag PDF-et (`14. Szenzorok kalibrálása.pdf`)!
2. Indítsd el az **STM32CubeIDE**-t, és nyisd meg a fejezethez tartozó kalibrációs projektet!
3. Vizsgáld meg a feszültségosztó bekötését (NTC és $10\text{ k}\Omega$-os ellenállás a `SEN` analóg lábra kötve: Panel: `PA1`, Nucleo: `PA3`)!
4. Futtasd a kalibrációs programot, és hajtsd végre a zéruspontos kalibrációt: merítsd a termisztort 0 °C-os jégvíz keverékbe, majd a középső (BTN3) nyomógombbal az OLED utasításai szerint tárold el a korrekciós értéket!
5. Figyeld meg az átlagolás (`SAMPLE_NUMBER`) hatását a mért értékek szórására és stabilitására!
6. A labor végén töltsd fel a megoldásodat a saját privát GitHub repódba a heti érdemjegyedért:
   ```bash
   git add .
   git commit -m "labor_09 zeruspontos termisztor kalibracio es oled megjelenites kesz"
   git push origin main
   ```
