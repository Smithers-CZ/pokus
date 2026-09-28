# Přepínač kernelů pro C128 = U32 (C64 kernel), U35 (C128 kernel) a U36 (funkční ROM – volitelně)  

![LED indikace v akci](<pictures/Test LED board.jpg>)

## Popis
Při procházení internetu jsem nalezl přepínače pro kernely a funkční ROM, ale buď to bylo jen pro kernely, nebo jen pro funkční ROM. Většinou se jednalo o přepínače ovládané z klávesnice s velkým množstvím vodičů uvnitř počítače.
Potřeboval jsem něco, co půjde ovládat jednoduše příkazy z klávesnice, nebo přes webové rozhraní. Taky mě děsila představa velkého množství vodičů (a tím pádem potencionálních problémů) uvnitř počítače. Proto jsem se rozhodl, že si postavím vlastní přepínač kernelů.
Začal jsem realizovat svoji představu, a sepsal jsem si požadavky na ovládání:
- ovládání z klávesnice nějakým příkazem
- ovládání přes webové rozhraní
- minimum vodičů uvnitř počítače
- co nejmenší dopad na vnější vzhled počítače  

![Ověřování nápadu na breadboardu](<pictures/Prototyping on breadboard.jpg>)

Nakonec se mi podařilo vytvořit řídící desku, která je s počítačem spojena pouze přes patice IO, které jsou na desce vhodně umístěné u sebe. Pro indikaci jsem navrhnul druhou desku, která je s řídící deskou spojena čtyřmi vodiči. Veškerá komunikace s počítačem probíhá pouze pomocí signálů na řídící desce pomocí vhodných propojení IO. Použití řídící desky neovlivní žádnou obvyklou funkci počítače. Funguje s jakýmikoli kernely a funkčními rom, (včetně JiffyDOS atp.) a to jak v módu C128, tak C64. A jak to celé funguje? Zde je popis jednotlivých částí:

> ### **Software**

Po zapnutí Commodore proběhne následující:
* Přepínač přidrží RESET na Commodoru
* Proběhne sekvence zelená – červená - modrá na všech LED
* Při připojování k WiFi běží modré „běžící světlo“ (až 50 pokusů po cca 1 s, tedy zhruba 50 s). IP adresa se nastaví podle DHCP.
* Po připojení k wifi se všechny LED krátce rozsvítí zeleně, potom se zobrazí barvy zvolených ROM.
* Při chybě v konfiguraci blikají červeně včechny tři diody současně.

#### Obslužný program v ESP32
Uvedený software je vytvořen v PlatformIO, ale měl by fungovat i v jiných IDE pro vývojové platformy s použitými knihovnami (ty najdete v platformio.ini v sekci lib_deps).

Program využívá mimo jiné knihovny i knihovnu [IECDevice](https://github.com/dhansel/IECDevice)

Před kompilací a nahráním do ESP32 je potřeba změnit hodnoty v jednotlivých konfiguračních souborech ve složce „data“. Po změně hodnot je potřeba nejdříve nahrát konfigurační soubory do ESP32 data do LittleFS, společně se soubory pro webového rozhraní. Webové rozhraní funguje v asynchronním režimu, takže se hodnoty změní na PC nebo chytrém telefonu bez nutnosti obnovovat stránku ručně.
Poté nahrát hlavní program do ESP32. Pokud pořadí obrátíte, nic se nestane, akorát se běh programu okamžitě zastaví, a Commodore zůstane v trvalém resetu.
Názvy konfiguračních souborů jsou pevně dané a nelze je měnit! (pokud tedy nezměníte tyto názvy i v samotném programu). Nastavené hodnoty (zvolené kernely a ROM) se uchovávají i po vypnutí počítače, takže při dalším spuštění zůstane C128 v poslední konfiguraci.

Standartně je program nastavený bez režimu ladění (na serial výstupu z ESP32 se neobjeví žádná data informující o průběhu programu). Pokud program z nějakého důvodu nefunguje, a jsou potřeba diagnostická data z běhu programu, je potřeba v souboru „settings.h“ odkomentovat řádek 47 (DebugON).

| Konfigurační soubor (název) | popis |
| ------------------ | -------------------------------------------------------- |
| devicenr.txt | v tomto souboru je číslo zařízení (DEVICE ID) pro změnu kernelů pomocí příkazu LOAD. V souboru je nastavena výchozí hodnota na 12. |
| U32ROM.txt, U35ROM.txt | počet a názvy kernelů.<br>  *Struktura souboru:*<br> Počet kernelů <br>  Jednotlivé názvy kernelů (limit 26 znaků) | 
| U36ROM.txt | počet a názvy funkčních ROM. Pokud nebudete U36 využívat, napište do souboru místo počtu ROM nulu. <br> *Struktura souboru:*<br> Počet funkčních ROM (bez osazené patice U36 zde napište 0)<br>  Názvy funkčních ROM (limit 26 znaků) | 
| U32stav.txt, U35stav.txt, U36stav.txt | soubory obsahují pouze číslo kernelu nebo ROM, které se spustí jako první (po každém uploadu LittleFS do ESP32). |  
| wifi.txt | parametry pro připojení k WiFi <br>  *Struktura souboru:*<br> Název WiFi (SSID)<br> Heslo k této WiFi |
| wifi_AP.txt | parametry pro vytvoření WiFi přístupového bodu (AP)<br>  *Struktura souboru:*<br> Název AP (SSID) (výchozí je „Commodore 128“) <br> Heslo k této WiFi – minimálně 8 znaků (výchozí je „Commodore“) |  

> **Poznámka 1:** Pokud program nalezne nějakou chybu v konfiguračních souborech, přejde Commodore to trvalého resetu a LED diody budou blikat červeně.<br> Možné chyby v konfiguraci:  
> 1. Chyba DEVICE ID (zakázány jsou tyto ID: 0, 1, 2, 4, 8, 9, 10, 11)  
> 2. Chybějící (nenalezené) konfigurační soubory  
> 3. Chybějící název ROM (popis) 
> 
> Pokud budou v konfiguračních souborech méně ROM, než je maximum povolených, neobsazené pozice budou mít automaticky název „NONE“  

> **Poznámka 2** pokud se nelze připojit k dané WiFi, bude fungovat ovládání přes internet pouze pomocí přístupu přes AP. Výchozí IP adresa je 10.10.10.10. Pokud chcete změnit IP adresu pro AP, změňte v src/settings.cpp řádky 5 až 8 (pouze v případě, že dochází ke konfliktu s jinými WiFi sítěmi).

> **POZOR!** Přepínač je navržen tak, aby při jakékoli chybě v softwaru nebo chybějícímu hardware zastavil činnost a držel Commodore v permanentním resetu! Nelze tedy provozovat přepínač bez desky indikace! Pokud by jste i přes to nepotřebovali indikaci, je potřeba změnit řídící software přepínače.  

#### **OVLÁDÁNÍ SWITCHERU:**

Kernely je možné měnit přímo z Commodoru pomocí příkazu (D)LOAD (nevýhodou je ovšem to, že je potřeba si napsat nebo pamatovat pozice jednotlivých kernelů pro každou EPROM), nebo programem ke switcheru, vytvořenému i pro přepínání kernelů.  

Změna kernelů a funkčních ROM pomocí příkazu (D)LOAD:  

*Syntaxe:*  
LOAD “U32#1U35#1U36#1“, ID  

Klíčové výrazy jsou **U32#n, U35#n, U36#n,** kde n udává číslo ROM, do které se má příslušný EPROM přepnout. ID je číslo zadané v konfiguračním souboru devicenr.txt. Pokud jsou ty to čísla mimo limit uvedený v konfiguračním souboru, nedojde k žádné změně. Nezáleží na pořadí přepínačů, ani na správnosti zadávaného textu, pokud se nenajde klíčový výraz, nic se nestane.  

Příklad: LOAD“JER**U32#3**GR**U36#5**“, ID  
Jedná se korektní výraz, U32 se přepne na kernel č.3 a U36 na funkční ROM č. 5
Poznámka:	pokud se jakýmkoli způsobem změní kernel nebo funkční ROM, provede se okamžitý RESET!

*Změna kernelů pomocí programu a webového rozhraní:*  
Pro získání IP adres slouží program „SWITCHER CONTROL.PRG“, který pro nahrání a spuštění vypíše na obrazovku IP adresy pro připojení z PC a IP adresu pro připojení přes AP (např. z chytrého telefonu).  

![Webové rozhraní](<pictures/web main screen.png>)  

Program „SWITCHER CONTROL.PRG“ komunikuje přes User port, sériově rychlostí 1200 baud (User port je ovládán přes optočleny, které fungují zároveň jako převodník úrovní). Program kromě IP adres načítá z přepínače aktuální čísla ROM, jejich názvy a device ID. Ve výchozím menu jsou zobrazeny aktuálně zvolené ROM, které lze pomocí šipek a klávesy Return dále měnit pomocí pull-down menu. Poslední volbou je „Confirm selected choice“, která po upozornění na okamžitý reset Commodoru zvolené hodnoty pošle zpět do ESP32.

![Webové rozhraní](<pictures/web main screen.png>)  

Kernely lze měnit pomocí jednoduchého webového rozhraní na PC nebo chytrém telefonu (klidně i současně). Na jediné obrazovce se objeví obrázek desky, kde jsou IO zabarveny odstínem podle zvoleného kernelu (barvy jsou stejné jak na RGB LED v Commodoru, tak i na webovém rozhraní). Pokud není osazena Function ROM U36, a je tato skutečnost nakonfigurována v souboru U36ROM.txt (počet kernelů nastaven na hodnotu 0), bude U36 na obrázku desky přeškrtnutá a nebude možno zvolit žádnou ROM (tlačítka budou neaktivní). Třetí RGB LED určená pro hodnotu ROM nebude svítit (jako by byl vybrán kernel č.8). Tuto skutečnost nelze žádným způsobem detekovat, prosím tedy o dodržení instrukcí a změnu v U36ROM.txt.  

![Start programu - načítají se data z přepínače](pictures/screen1.png)  
*Start programu - načítají se data z přepínače*  

![Hlavní menu - zobrazení aktuálních kernelů a ROM](pictures/screen2.png)  
*Hlavní menu - zobrazení aktuálních kernelů a ROM*

![Výběr kernalů mro mód C64 (příklad)](pictures/screen3.png)  
*Výběr kernalů mro mód C64 (příklad)*

![Výběr kernalů mro mód C128 (příklad)](pictures/screen4.png)  
*Výběr kernalů mro mód C128 (příklad)*

![Výběr funkčních ROM (příklad)](pictures/screen5.png)  
*Výběr funkčních ROM (příklad)*  

![Zapsání vybraných ROM do přepínače](pictures/screen7.png)  
*Zapsání vybraných ROM do přepínače*

### Naprogramování E(E)PROM  
Naprogramování EPROM, nebo lépe EEPROM záleží čistě na vašich potřebách, lze nalézt spoustu kernelů pro mód C64, pouze několik pro C128, a dostatek ROM pro funkční ROM.  
#### *Struktura kernelů pro C64:*
Každý slot je složený z kernelu pro basic (8k) a dalšího kernelu (např. JiffyDos) (8k), takže jeden slot má 16 kb. Všechny zvolené kernely je potřeba spojit do jednoho .bin souboru (lze použít např. [BIN Wizard](https://github.com/r1me/BINWizard)) a poté naprogramovat do EEPROM. Zvolil jsem EEPROM pro snadnější změnu kernelů (pokud bude potřeba).

<table border="1" cellpadding="4" cellspacing="0">
  <thead>
    <tr><th>č. kernelu</th><th>BIN</th><th>velikost (kb)</th></tr>
  </thead>
  <tbody>
    <tr><td rowspan="2" style="text-align: center;">1</td><td>C64 basic (901226-01)</td><td  style="text-align: center;">8</td></tr>
    <tr><td>C64 kernel (901227-03)</td><td style="text-align: center;">8</td></tr>
    <tr><td rowspan="2"  style="text-align: center;">2</td><td>C64 basic (901226-01)</td><td style="text-align: center;">8</td></tr>
    <tr><td>kernel 2</td><td style="text-align: center;">8</td></tr>
    <tr><td rowspan="2"  style="text-align: center;">3</td><td>C64 basic (901226-01)</td><td style="text-align: center;">8</td></tr>
    <tr><td>kernel 3</td><td style="text-align: center;">8</td></tr>
    <tr><td rowspan="2" style="text-align: center;">4</td><td>C64 basic (901226-01)</td><td style="text-align: center;">8</td></tr>
    <tr><td>kernel 4</td><td style="text-align: center;">8</td></tr>
    <tr><td rowspan="2" style="text-align: center;">5</td><td>C64 basic (901226-01)</td><td style="text-align: center;">8</td></tr>
    <tr><td>kernel 5</td><td style="text-align: center;">8</td></tr>
    <tr><td rowspan="2" style="text-align: center;">6</td><td>C64 basic (901226-01)</td><td style="text-align: center;">8</td></tr>
    <tr><td>kernel 6</td><td style="text-align: center;">8</td></tr>
    <tr><td rowspan="2" style="text-align: center;">7</td><td>C64 basic (901226-01)</td><td style="text-align: center;">8</td></tr>
    <tr><td>kernel 7</td><td style="text-align: center;">8</td></tr>
    <tr><td rowspan="2" style="text-align: center;">8</td><td>C64 basic (901226-01)</td><td style="text-align: center;">8</td></tr>
    <tr><td>kernel 8</td><td style="text-align: center;">8</td></tr>
  </tbody>
</table>

#### *Struktura kernelů pro C128:*
Lze nalézt již hotové kernely, které mají 16 kb 

<table border="1" cellpadding="4" cellspacing="0">
  <thead>
    <tr><th>č. kernelu</th><th>BIN</th><th>velikost (kb)</th></tr>
  </thead>
  <tbody>
    <tr><td style="text-align: center;">1</td><td>C128 kernel (318020-06)</td><td style="text-align: center;">16</td></tr>
    <tr><td style="text-align: center;">2</td><td>kernel 2</td><td style="text-align: center;">16</td></tr>
    <tr><td style="text-align: center;">3</td><td>kernel 3</td><td style="text-align: center;">16</td></tr>
    <tr><td style="text-align: center;">4</td><td>kernel 4</td><td style="text-align: center;">16</td></tr>
  </tbody>
</table>

#### *Struktura pro Funkční ROM:*
každá funkční ROM má 32 kb. Lze jich nalézt celkem dost.  
Pokud by jste chtěli nějaké vlastní programy, které by jste chtěli mít po ruce po spuštění, lze pro vytvoření vlastní funkční ROM využít program [StartApps](https://pastbytes.com/startapps/). Upravit a zkompilovat to pro vlastní programy není složité. Můžete také využít již hotové StartAppsy, které naleznete na webu. Pokud se rozhodnete pro vlastní programy, mějte na paměti, že se všechny musejí vejít do 32 kb včetně obslužného programu.  
Doporučuji každou vlastní StartApps vyzkoušet v emulátoru C128 na PC (např. VICE).

<table border="1" cellpadding="4" cellspacing="0">
  <thead>
    <tr><th>č. ROM</th><th>BIN</th><th>velikost (kb)</th></tr>
  </thead>
  <tbody>
    <tr><td style="text-align: center;">1</td><td>ROM 1</td><td style="text-align: center;">32</td></tr>
    <tr><td style="text-align: center;">2</td><td>ROM 2</td><td style="text-align: center;">32</td></tr>
    <tr><td style="text-align: center;">3</td><td>ROM 3</td><td style="text-align: center;">32</td></tr>
    <tr><td style="text-align: center;">4</td><td>ROM 4</td><td style="text-align: center;">32</td></tr>
    <tr><td style="text-align: center;">5</td><td>ROM 5</td><td style="text-align: center;">32</td></tr>
    <tr><td style="text-align: center;">6</td><td>ROM 6</td><td style="text-align: center;">32</td></tr>
    <tr><td style="text-align: center;">7</td><td>ROM 7</td><td style="text-align: center;">32</td></tr>
    <tr><td style="text-align: center;">8</td><td>ROM 8</td><td style="text-align: center;">32</td></tr>
  </tbody>
</table>

> **DŮLEŽITÉ UPOZORNĚNÍ:** Nevkládejte do přepínače jiné typy EPROMů, než je uvedeno v seznamu součástek nebo jinak naprogramované! Nebude to fungovat a mohlo by dojít k poškození počítače!  
###  

> ### **Hardware**  

#### **Návrh a realizace**
Pro ovládání jsem zvolil ESP32, kvůli kompatibilitě s knihovnami, které jsem chtěl použít.  
Pro ovládání se dá použít jakýkoli počítač nebo chytrý telefon připojený na stejnou WiFi jako je přepínač, nebo se lze připojit přímo na ESP32, který má vytvořen vlastní přístupový bod (AP).  
Pro indikaci, která ROM je aktuálně nastavena, slouží trojice RGB LED umístěná místo LED, která signalizovala zapnutí počítače. LED diody signalizují barvou stav aktivní ROM v pořadí U32 – U35 – U36  

<table border="1" cellpadding="4" cellspacing="0">
  <thead>
    <tr><th>číslo kernelu / ROM</th><th>barva</th></tr>
  </thead>
  <tbody>
    <tr><td style="text-align: center;">1</td><td>červená</td></tr>
    <tr><td style="text-align: center;">2</td><td>zelená</td></tr>
    <tr><td style="text-align: center;">3</td><td>modrá</td></tr>
    <tr><td style="text-align: center;">4</td><td>magenta</td></tr>
    <tr><td style="text-align: center;">5</td><td>žlutá</td></tr>
    <tr><td style="text-align: center;">6</td><td>cyan</td></tr>
    <tr><td style="text-align: center;">7</td><td>bílá</td></tr>
    <tr><td style="text-align: center;">8</td><td>černá (vypnutá)</td></tr>
  </tbody>
</table>

Pro kernely U32 a funkční ROM U36 lze využít maximálně 8 pozic, pro kernely U35 jsou to maximálně 4 pozice.

![Hlavní deska a deska pro LED indikaci](<pictures/Main board and LED board.jpg>)

## **Seznam potřebných součástek:**

| Symbol | hodnota | množství (ks) |
| ------ | :---: | ---: |
| **ŘÍDÍCÍ DESKA** | 
| U32 (C64 kernely) | 27C010 | 1 |
| U35 (C128 kernely) | 27C512 | 1 |
| U36 (Funkční ROM) | 27C020 | 1 |
| U30 | 7406 (pokud je potřeba) | 1 |
| MCU1 | ESP32 38 pin (WROOM) | 1 |
| Level converter | level konvertor pro 8 pozic | 1 |
| U1 | KB846 (4x optočlen) | 1 |
| U2 | MCP23017 | 1 |
| U3 | UA78M33 (nebo podobný 3V3 regulátor, TO-220) | 1 |
| R10 | 10k | 1 |
| R1, R4, R5 | 4k7 | 3 |
| R2 | 2k2 | 1 |
| R3, R8, R9 | 180R | 3 |
| R6 | 330R | 1 |
| PIN lišta M-M precizní** | 14 pinů | 14 |
| PIN lišta M-M precizní | 20 pinů | 2 |
| PIN lišta M-M precizní | 7 pinů | 2 |
| PIN lišta F-M precizní | pro MCU1 19 pinů | 2 |
| PIN lišta F-M precizní | pro level converter 8 pinů | 2 |
| Patice pro IO** | 28 pinů, široká | 7 |
| Patice pro IO | 32 pinů, široká | 1 |
| Patice pro IO | 40 pinů, široká | 1 |
| Patice pro IO | 28 pinů, úzká | 1 |
| Patice pro IO | 16 pinů, úzká | 1 |
| Patice pro IO | 14 pinů, úzká | 1 |
| **Konektory JST 2,54 mm** |
| Napájení* | 2 piny | 1 |
| Výstup pro LED desku | 4 piny | 1 |
| **Pinové lišty\*** |
| Výstup pro Serial + RESET | 5 pinů | 1 |
| Výstup pro Serial IEC | 4 piny | 1 |
| **Ostatní materiál\*\*\*** | 
| Mosazná závitová matice | M3 (pro snažší vyjmutí přepínače z patic) | 6 |
| Plastová podložka | s oboustrannou lepící páskou (cca 5 x5 mm) | 6 |  

> \* = pouze pro diagnostické účely, není nutno osazovat  
> \*\* = U33 aU34 jsou sice na desce navrženy k osazení, ale nemusí se osazovat, ale je třeba počítat s tím, že se pro manipulaci s těmito IO bude muset přepínač demontovat  
> \*\*\* = je dobré mít dostatečně dlouhé šroubky M3 na zdvihnutí celé desky z patic

| Symbol | hodnota | množství (ks) |
| ------ | :---: | ---: |
| **LED DESKA** |
| U1 | MCP23017 | 1 |
| Patice pro IO | 28 pinů, úzká | 1 |
| R1, R2, R4, R5, R7, R8 | 220R | 6
| R3, R6, R9 | 130 | 3 |
| R10 | 10k | 1 |
| D1, D2, D3 | RGB LED 5x2mm, společná katoda (BGKR) | 3 |
| **JST konektor:** |
| Vstup pro připojení | 4 piny | 1 |
| **Ostatní materiál** |
| distanční sloupky | M3, 10 mm | 2 |  

Po kompletním osazení doporučuji vše odzkoušet „na stole“, zda-li vše funguje tak, jak má. Pro napájení použijte externí zdroj. Teprve po úspěšném testu bych přepínač zabudoval do počítače.  
> **Pro instalaci přepínače je zapotřebí, aby IO U32, (U33, U34), U35, U36, U4 a U30 měly na desce C128 patice (nejlépe obyčejné, ne precizní).**

## Detaily z montáže

### Montáž LED desky

![Demontáž Power led](<pictures/keyboard power led.jpg>)
*Demontáž Power LED*

![Power LED demontována](<pictures/Power led dismounted.jpg>)
*Power LED demontována*

![Kompletní LED deska](<pictures/Complete LED board.jpg>)
*Kompletní LED deska*

![Detail distančních sloupků](<pictures/detail of spacer pillars.jpg>)
*Detail distančních sloupků*

![Detail ohnutí LED](<pictures/LED board - installing LED.jpg>)
*Detail ohnutí LED*

![Detail LED umístěných v krytu](<pictures/RGB LED detail.jpg>)
*Detail LED umístěných v krytu*

![Osazená deska LED na svém místě](<pictures/LED board mounted on place.jpg>)
*Osazená deska LED na svém místě*

![LED v provozu](<pictures/Switcher in action.jpg>)
*LED v provozu*

### Montáž hlavní desky

![Prostor pro umístění hlavní desky](<pictures/Place for switcher main board on C128 board.jpg>)
*Prostor pro umístění hlavní desky*

![Deska C128 připravená pro hlavní desku](<pictures/C 128 board prepared for insetring switcher board.jpg>)
*Deska C128 připravená pro hlavní desku: Přidána patice pro U30, odletovány piny pro původní Power LED*

![Detail patice pro U30](<pictures/Installed socket for U30.jpg>)
*Detail patice pro U30*

![Detail odletování pinů pro Power LED](<pictures/removed pins for power led.jpg>)
*Detail odletování pinů pro Power LED*

![Hlavní deska umístěná na své místo](<pictures/Switcher 2.0 board on place .jpg>)
*Hlavní deska umístěná na své místo*

![Umístění pomocí pinové lišty](<pictures/Positioning board.jpg>)
*Umístění pomocí pinové lišty nejlépe na čtyřech místech*

![Značení míst pro plastové podložky](<pictures/plastic pad marks on C128 board.jpg>)
*Značení míst pro plastové podložky*

![Matky M3 před pájením](<pictures/M3 nuts before soldering.jpg>)
*Matky M3 před pájením, vystřeďeno pomocí šroubu*

![Připájené matky M3 na hlavní desku](<pictures/Soldered M3 nuts.jpg>)
*Připájené matky M3 na hlavní desku*

![Doporučený postup pro letování pinových lišt](<pictures/Aligning pins vith IO socket as helper.jpg>)
*Doporučený postup pro letování pinových lišt*

![Detail naletování pinové lišty](<pictures/Detail of soldered pins - top.jpg>)
*Detail naletování pinové lišty \- vrchní strana*

![Detail naletování pinové lišty](<pictures/Detail of soldered pins.jpg>)
*Detail naletování pinové lišty \- spodní strana*

![Pájení pinů pro U32 pomocí obyčejné patice](<pictures/Soldering 2x14 pins - bottom.jpg>)
*Pájení pinů pro U32 pomocí obyčejné patice*

![Piny pro U32](<pictures/Soldered 2x14 pins - bottom.jpg>)
*Doporučený postup pro pájení pinových lišt - při pohledu na vrchní stranu hlevní desky zleva doprava (jako první pinové lišty)*

![Patice peo U32](<pictures/Soldered IC socket 2x14 pins - top.jpg>)
*Napájená patice pro EPROM pro U32 (na obrázku vzadu)*

![Piny pro U33](<pictures/Soldering procedure - 1 bottom.jpg>)  
*Piny pro U33*

![Patice pro U33](<pictures/Soldering procedure -1 top.jpg>)  
*Patice pro U33*

![Piny pro U34](<pictures/Soldering procedure -2 bottom.jpg>)  
*Piny pro U34*

![Patice pro U34](<pictures/Soldering procedure -2 top.jpg>)  
*Patice pro U34*

![Kompletně osazená hlavní deska bez IO](<pictures/Completed main board without IC.jpg>)
*Kompletně osazená hlavní deska bez IO*

![Doporučené piny pro ESP32 a převodník úrovní](<pictures/Recommended pins for MCU and level converter.jpg>)
*Doporučené piny pro ESP32 a převodník úrovní*

![Osazení hlevní desky ESP32 a převodníku úrovní](<pictures/Almost completed main board.jpg>)
*Osazení hlevní desky ESP32 a převodníku úrovní*

![Kompletně osazená hlavní deska s IO](<pictures/Completed main board with IC.jpg>)
*Kompletně osazená hlavní deska s IO*

![Kompletně osazená hlavní deska - spodní strana](<pictures/Completed main board - bottom.jpg>)
*Kompletně osazená hlavní deska - spodní strana*

![Hlavní deska usazená na desce C128](<pictures/main board on place.jpg>)
*Hlavní deska usazená na desce C128*

![Detail pinů v patici](<pictures/pins in socket detail.jpg>)
*Detail pinů v patici*

![Detail vytahovacího mechanismu](<pictures/detail of the pull-out mechanism.jpg>)
*Detail vytahovacího mechanismu - plastová podložka nalepená na desce; dlouhý šroub určený pro vytáhnutí pinů z patic (šrouby v desce jen pro vytažení hlavní desky)*

![Připraveno k použití](<pictures/Ready to run.jpg>)
*Připraveno k použití*

![Pohled z boku na uzavřený počítač](<pictures/side detail.jpg>)
*Pohled z boku na uzavřený počítač*


[![Přepínač v akci](https://img.youtube.com/vi/mKyUM33U5w/hqdefault.jpg)](https://www.youtube.com/watch?v=mKyUM33U5w)  
*Přepínač v akci*
