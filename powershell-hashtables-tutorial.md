# Hashtables v PowerShellu: kompletní průvodce

Hashtable (hash tabulka, slovník, asociativní pole) je jedna z nejpoužívanějších datových struktur v PowerShellu. Neslouží jen k ukládání dat: PowerShell ji používá jako univerzální způsob, jak předat pojmenované hodnoty. Potkáte ji při předávání parametrů (splatting), při tvorbě objektů, ve vypočítaných sloupcích `Select-Object`, v konfiguračních souborech `.psd1` i při práci s JSON.

Příklady jsou ověřené v PowerShellu 7.4. Naprostá většina funguje stejně i ve Windows PowerShellu 5.1; kde je potřeba novější verze, je to u příkladu uvedeno.

## Obsah

1. [Co je hashtable a kdy ji použít](#1-co-je-hashtable-a-kdy-ji-použít)
2. [Vytvoření](#2-vytvoření)
3. [Čtení hodnot](#3-čtení-hodnot)
4. [Přidávání, změna a mazání](#4-přidávání-změna-a-mazání)
5. [Zjišťování, co v tabulce je](#5-zjišťování-co-v-tabulce-je)
6. [Procházení](#6-procházení)
7. [Pořadí položek a `[ordered]`](#7-pořadí-položek-a-ordered)
8. [Klíče podrobně](#8-klíče-podrobně)
9. [Vnořené struktury](#9-vnořené-struktury)
10. [Splatting: hashtable jako sada parametrů](#10-splatting-hashtable-jako-sada-parametrů)
11. [Hashtable a PSCustomObject](#11-hashtable-a-pscustomobject)
12. [Hashtable jako předpis: vypočítané vlastnosti](#12-hashtable-jako-předpis-vypočítané-vlastnosti)
13. [Vyhledávací tabulky a seskupování dat](#13-vyhledávací-tabulky-a-seskupování-dat)
14. [Praktické vzory](#14-praktické-vzory)
15. [Ukládání a načítání: JSON, PSD1, CLIXML](#15-ukládání-a-načítání-json-psd1-clixml)
16. [Kopírování a porovnávání](#16-kopírování-a-porovnávání)
17. [Příbuzné typy](#17-příbuzné-typy)
18. [Nejčastější chyby](#18-nejčastější-chyby)
19. [Tahák](#19-tahák)

---

## 1. Co je hashtable a kdy ji použít

Hashtable je kolekce dvojic **klíč → hodnota**. Místo pořadového čísla (jako u pole) se k hodnotě dostanete přes klíč, typicky text.

```powershell
$pole    = @('ovoce', 'zvíře', 'barva')          # přístup přes pozici: $pole[0]
$slovnik = @{ jablko = 'ovoce'; pes = 'zvíře' }  # přístup přes klíč:   $slovnik['jablko']
```

Technicky jde o .NET typ `System.Collections.Hashtable`. Název vychází z toho, jak funguje uvnitř: z klíče se spočítá číslo (hash) a to určí, do které „přihrádky" se hodnota uloží. Z toho plynou tři vlastnosti, které se v celém tutoriálu vracejí:

- **Vyhledání podle klíče je velmi rychlé** a prakticky nezávisí na počtu položek. Pole musíte projít prvek po prvku, hashtable sáhne rovnou do správné přihrádky.
- **Klíče jsou jedinečné.** Jeden klíč má vždy právě jednu hodnotu.
- **Pořadí položek není zaručené.** Řídí se hashem, ne pořadím vložení (řešení je v kapitole 7).

Kdy sáhnout po které struktuře:

| Potřebuji | Vhodná struktura |
|---|---|
| Seznam hodnot v daném pořadí | pole `@()` nebo `List[T]` |
| Rychle najít hodnotu podle názvu nebo ID | hashtable `@{}` |
| Sadu pojmenovaných hodnot, kterou průběžně měním | hashtable `@{}` |
| Záznam (řádek dat) pro výpis, export do CSV, pipeline | `[pscustomobject]` |
| Slovník se zachovaným pořadím | `[ordered]@{}` |

## 2. Vytvoření

### Literál `@{}`

```powershell
$mojeSlovnik = @{
    jablko = 'ovoce'
    pes    = 'zvíře'
    modrá  = 'barva'
}
```

Prázdná tabulka, kterou naplníte později:

```powershell
$prazdna = @{}
```

Na jednom řádku se položky oddělují středníkem:

```powershell
$bod = @{ X = 10; Y = 20 }
```

### Kdy klíč potřebuje uvozovky

Jednoduchý klíč (písmena, číslice, podtržítko) uvozovky nepotřebuje. Nutné jsou, jakmile klíč obsahuje mezeru, pomlčku nebo jiný speciální znak:

```powershell
$hlavicky = @{
    Accept         = 'application/json'
    'Content-Type' = 'application/json'
    'Moje hodnota' = 123
}
```

### Hodnotou může být cokoli

Hodnota je libovolný výraz: číslo, pole, výsledek příkazu, další hashtable nebo blok kódu.

```powershell
$info = @{
    Jmeno     = 'Jan'
    Vek       = 30
    Jazyky    = @('čeština', 'angličtina')
    Vytvoreno = Get-Date
    Adresa    = @{ Mesto = 'Praha'; PSC = '11000' }
    Pozdrav   = { param($kdo) "Ahoj, $kdo!" }
}
```

### Vytvoření ze dvou polí nebo z dat

Tabulku často nestavíte ručně, ale plníte ji ve smyčce:

```powershell
$klice   = 'a', 'b', 'c'
$hodnoty = 1, 2, 3

$zPoli = @{}
for ($i = 0; $i -lt $klice.Count; $i++) {
    $zPoli[$klice[$i]] = $hodnoty[$i]
}
$zPoli['b']   # 2
```

Pro text ve tvaru `klíč=hodnota` existuje hotový příkaz `ConvertFrom-StringData`:

```powershell
$zTextu = ConvertFrom-StringData @'
server = db01
port = 5432
'@
$zTextu['server']   # db01
```

Všechny hodnoty jsou v tomto případě řetězce, tedy i `port` je text `'5432'`.

## 3. Čtení hodnot

### Hranaté závorky a tečka

```powershell
$mojeSlovnik['jablko']   # ovoce
$mojeSlovnik.jablko      # ovoce
```

Oba zápisy dělají totéž. Tečková notace je kratší, hranaté závorky jsou univerzálnější. Klíč se speciálními znaky jde zapsat oběma způsoby:

```powershell
$hlavicky['Content-Type']   # application/json
$hlavicky.'Content-Type'    # application/json
```

Klíč uložený v proměnné:

```powershell
$hledam = 'pes'
$mojeSlovnik[$hledam]   # zvíře
$mojeSlovnik.$hledam    # zvíře
```

### Neexistující klíč vrací `$null`

Čtení klíče, který v tabulce není, neskončí chybou. Dostanete `$null`:

```powershell
$vysledek = $mojeSlovnik['hruška']
$null -eq $vysledek   # True
```

Je to pohodlné, ale překlep v klíči tak snadno projde bez povšimnutí. Jak spolehlivě rozlišit „klíč chybí" od „hodnota je `$null`", ukazuje kapitola 5.

Přísný režim chování mění, ale jen u tečkové notace:

```powershell
& {
    Set-StrictMode -Version Latest
    $h = @{ a = 1 }
    $h['neexistuje']                                      # $null, bez chyby
    try { $h.neexistuje } catch { 'Tečka v přísném režimu: chyba' }
}
```

### Více klíčů najednou

Do hranatých závorek lze dát pole klíčů a zpět dostanete pole hodnot:

```powershell
$mojeSlovnik['jablko', 'pes']   # ovoce, zvíře
```

### Hodnoty v textovém řetězci

Uvnitř dvojitých uvozovek musíte přístup k hashtable obalit do `$( )`. Bez toho PowerShell dosadí jen samotnou proměnnou a zbytek bere jako obyčejný text:

```powershell
"Jablko je $($mojeSlovnik['jablko'])"   # Jablko je ovoce
"Jablko je $($mojeSlovnik.jablko)"      # Jablko je ovoce
"Jablko je $mojeSlovnik.jablko"         # Jablko je System.Collections.Hashtable.jablko
```

### Když se klíč jmenuje stejně jako vlastnost

Hashtable má vlastní vlastnosti jako `Count`, `Keys` a `Values`. Pokud má stejný název i některý klíč, tečková notace dá přednost klíči:

```powershell
$past = @{ Count = 'já jsem klíč'; a = 1 }
$past.Count          # já jsem klíč
$past.psbase.Count   # 2
```

Přes `psbase` se dostanete ke skutečné vlastnosti objektu. U tabulek, jejichž klíče neovlivníte (data ze souboru, z API), je `psbase.Count` a `psbase.Keys` jistější volba.

## 4. Přidávání, změna a mazání

### Přiřazení: přidá, nebo přepíše

```powershell
$mojeSlovnik['strom'] = 'rostlina'        # nový klíč
$mojeSlovnik['pes']   = 'čtyřnohé zvíře'  # existující klíč se přepíše
$mojeSlovnik.kočka    = 'zvíře'           # totéž tečkovou notací
```

### Metoda `Add`: přidá, ale nepřepíše

`Add` se od přiřazení liší tím, že u existujícího klíče vyhodí chybu. Hodí se, když by duplicita znamenala chybu ve vstupních dat a chcete se o ní dozvědět:

```powershell
$mojeSlovnik.Add('růže', 'rostlina')

try {
    $mojeSlovnik.Add('růže', 'květina')
} catch {
    "Chyba: klíč 'růže' už existuje"
}
```

### Odstranění

```powershell
$mojeSlovnik.Remove('modrá')       # odstraní jednu položku
$mojeSlovnik.Remove('neexistuje')  # neexistující klíč nevadí, nic se nestane
```

`Clear()` vyprázdní celou tabulku:

```powershell
$docasna = @{ a = 1; b = 2 }
$docasna.Clear()
$docasna.Count   # 0
```

### Sloučení dvou tabulek

Operátor `+` vytvoří novou tabulku s položkami obou. Pokud se některý klíč opakuje, skončí chybou:

```powershell
$zaklad   = @{ server = 'db01'; port = 5432 }
$doplnek  = @{ ssl = $true }
$slouceno = $zaklad + $doplnek
$slouceno.Count   # 3

try {
    $zaklad + @{ port = 9999 }
} catch {
    'Chyba: klíč port je v obou tabulkách'
}
```

Sloučení, při kterém druhá tabulka přepisuje hodnoty první, je v kapitole 14.

### Úprava hodnoty na místě

Hodnotu lze měnit přímo, včetně složených operátorů:

```powershell
$sklad = @{ jablka = 10; hrušky = 4 }
$sklad['jablka'] += 5
$sklad['hrušky']--
$sklad['jablka']   # 15
$sklad['hrušky']   # 3
```

## 5. Zjišťování, co v tabulce je

### Počet položek

```powershell
$sklad.Count   # 2
```

### Existence klíče

```powershell
$sklad.ContainsKey('jablka')   # True
$sklad.Contains('jablka')      # True
$sklad.ContainsKey('švestky')  # False
```

Obě metody dělají u hashtable totéž. Rozdíl je v tom, že `Contains` funguje i u `[ordered]` slovníku, který `ContainsKey` nemá. Kdo si zvykne na `Contains`, nemusí rozdíl řešit.

### Proč nestačí `if ($tabulka[$klic])`

Podmínka testuje pravdivost hodnoty, ne existenci klíče. Selže proto u hodnot `0`, `$false`, `''` a `$null`:

```powershell
$stav = @{ chyby = 0; aktivni = $false; poznamka = $null }

if ($stav['chyby']) { 'má chyby' } else { 'podmínka neprošla, i když klíč existuje' }

$stav.ContainsKey('chyby')      # True
$stav.ContainsKey('poznamka')   # True  (klíč existuje, hodnota je $null)
$stav.ContainsKey('neznamy')    # False
```

### Existence hodnoty

```powershell
$sklad.ContainsValue(15)   # True
```

Na rozdíl od hledání klíče musí `ContainsValue` projít celou tabulku, u velkých tabulek je tedy pomalé.

### Prázdná tabulka je v podmínce pravdivá

Prázdné pole se v podmínce chová jako `$false`, prázdná hashtable jako `$true`. Prázdnost testujte přes `Count`:

```powershell
$nic = @{}
if ($nic)              { 'prázdná hashtable je pravdivá' }
if ($nic.Count -eq 0)  { 'správný test prázdnosti' }
```

## 6. Procházení

### Přes klíče

```powershell
$zvirata = @{ pes = 'haf'; kočka = 'mňau'; kráva = 'bú' }

foreach ($klic in $zvirata.Keys) {
    Write-Output "${klic}: $($zvirata[$klic])"
}
```

Pozor na zápis `"${klic}: ..."`. Varianta `"$klic: ..."` skončí chybou parseru, protože dvojtečku za názvem proměnné PowerShell chápe jako oddělovač oboru (jako v `$env:PATH` nebo `$script:pocet`). Složené závorky jednoznačně určí, kde název proměnné končí.

### Přes dvojice klíč–hodnota

`GetEnumerator()` vrací položky jako objekty s vlastnostmi `Key` a `Value` (`Name` je alias pro `Key`):

```powershell
foreach ($polozka in $zvirata.GetEnumerator()) {
    Write-Output "$($polozka.Key) dělá $($polozka.Value)"
}
```

### Hashtable v pipeline

Pole se v pipeline rozpadne na jednotlivé prvky, hashtable ne. Projde jako jeden objekt:

```powershell
($zvirata | Measure-Object).Count                   # 1
($zvirata.GetEnumerator() | Measure-Object).Count   # 3
```

Kdykoli chcete položky zpracovat pomocí `Where-Object`, `Sort-Object` nebo `ForEach-Object`, začněte tedy `GetEnumerator()`:

```powershell
$zvirata.GetEnumerator() | ForEach-Object { "$($_.Key) = $($_.Value)" }
```

### Řazení

Protože pořadí položek není zaručené, pro čitelný výpis se obvykle řadí:

```powershell
$body = @{ Petr = 72; Jana = 95; Karel = 58; Eva = 88 }

# podle klíče
$body.GetEnumerator() | Sort-Object Key | ForEach-Object { "$($_.Key): $($_.Value)" }

# podle hodnoty sestupně
$body.GetEnumerator() | Sort-Object Value -Descending | ForEach-Object { "$($_.Key): $($_.Value)" }
```

### Filtrování

Výsledkem filtru nad `GetEnumerator()` jsou jednotlivé položky, ne nová hashtable. Pokud chcete zase tabulku, složte ji znovu:

```powershell
$uspesni = @{}
$body.GetEnumerator() |
    Where-Object { $_.Value -ge 80 } |
    ForEach-Object { $uspesni[$_.Key] = $_.Value }

$uspesni.Keys | Sort-Object   # Eva, Jana
```

### Změna tabulky během procházení

Tabulku nelze měnit, dokud ji smyčka prochází. Následující kód skončí chybou „Collection was modified":

```powershell
$ceny = @{ chléb = 40; mléko = 25; máslo = 60 }

try {
    foreach ($klic in $ceny.Keys) {
        $ceny[$klic] = $ceny[$klic] * 1.1
    }
} catch {
    'Chyba: kolekce byla během procházení změněna'
}
```

Řešením je procházet kopii klíčů. Operátor `@( )` vytvoří z klíčů samostatné pole, které už na tabulce nezávisí:

```powershell
$ceny = @{ chléb = 40; mléko = 25; máslo = 60 }

foreach ($klic in @($ceny.Keys)) {
    $ceny[$klic] = [math]::Round($ceny[$klic] * 1.1, 1)
}
$ceny['chléb']   # 44
```

Stejně se postupuje při mazání položek podle podmínky:

```powershell
foreach ($klic in @($ceny.Keys)) {
    if ($ceny[$klic] -lt 30) { $ceny.Remove($klic) }
}
$ceny.Keys | Sort-Object   # chléb, máslo
```

### `Keys` není pole

`Keys` a `Values` jsou kolekce bez indexování. Výraz `$tabulka.Keys[0]` proto nevrátí první klíč, ale všechny. Pro přístup přes pozici je převeďte na pole:

```powershell
$klicePole = @($zvirata.Keys)
$klicePole[0]       # první klíč (který to bude, není u běžné hashtable zaručeno)
$klicePole.Count    # 3
```

## 7. Pořadí položek a `[ordered]`

Běžná hashtable vrací položky v pořadí daném vnitřním uspořádáním, ne v pořadí vložení:

```powershell
$bezPoradi = @{ první = 1; druhý = 2; třetí = 3; čtvrtý = 4 }
$bezPoradi.Keys -join ', '   # pořadí je libovolné a může se lišit
```

Pokud na pořadí záleží (výpis, export, tvorba objektu), napište před literál `[ordered]`:

```powershell
$sPoradim = [ordered]@{ první = 1; druhý = 2; třetí = 3; čtvrtý = 4 }
$sPoradim.Keys -join ', '   # první, druhý, třetí, čtvrtý
```

`[ordered]` musí stát přímo před literálem `@{}`. Výsledkem je jiný typ, `System.Collections.Specialized.OrderedDictionary`, který se používá téměř stejně jako hashtable, s několika rozdíly:

```powershell
$sPoradim.GetType().Name        # OrderedDictionary

$sPoradim[0]                    # 1  (přístup přes pozici)
$sPoradim['třetí']              # 3  (přístup přes klíč)

$sPoradim.Insert(0, 'nultý', 0) # vložení na určitou pozici
$sPoradim.Keys -join ', '       # nultý, první, druhý, třetí, čtvrtý

$sPoradim.Contains('druhý')     # True (metoda ContainsKey zde neexistuje)
```

### Past: číselné klíče v `[ordered]`

Protože `[ordered]` umí číst přes pozici, je celé číslo v hranatých závorkách vždy chápáno jako pozice, ne jako klíč:

```powershell
$mista = [ordered]@{ 1 = 'zlato'; 2 = 'stříbro'; 3 = 'bronz' }
$mista[1]           # stříbro  (položka na pozici 1, tedy druhá)
$mista[[object]1]   # zlato    (klíč 1)
```

Jednodušší je se číselným klíčům v `[ordered]` vyhnout a použít textové.

## 8. Klíče podrobně

### Velikost písmen

Tabulka vytvořená literálem `@{}` velikost písmen v klíčích nerozlišuje:

```powershell
$h = @{ Jablko = 'ovoce' }
$h['JABLKO']   # ovoce
$h['jablko']   # ovoce
```

Tabulka vytvořená konstruktorem `[hashtable]::new()` ji ale rozlišuje. Je to častý zdroj záhadných chyb, když někdo oba zápisy považuje za rovnocenné:

```powershell
$citliva = [hashtable]::new()
$citliva['Jablko'] = 'ovoce'
$null -eq $citliva['JABLKO']   # True (klíč nenalezen)
$citliva['Jablko']             # ovoce
```

Rozlišování velikosti písmen se hodí třeba pro klíče, jako jsou hesla, tokeny nebo cesty v Linuxu.

### Typ klíče: `1` není `'1'`

Klíč zapsaný bez uvozovek jako číslo je opravdu číslo. Text `'1'` je jiný klíč, takže v tabulce mohou být oba vedle sebe:

```powershell
$cisla = @{}
$cisla[1]   = 'číslo jedna'
$cisla['1'] = 'text jedna'
$cisla.Count   # 2
$cisla[1]      # číslo jedna
$cisla['1']    # text jedna
```

V praxi na to narazíte při načítání dat: ID z CSV souboru je vždy text, ID z databáze nebo JSON bývá číslo. Pokud tabulku naplníte jedním a hledáte druhým, nenajdete nic:

```powershell
$uzivatele = @{}
$uzivatele[42] = 'Jan'

$idZCsv = '42'
$null -eq $uzivatele[$idZCsv]   # True (nenalezeno)
$uzivatele[[int]$idZCsv]        # Jan
```

Nejjednodušší obrana je klíče při plnění i hledání sjednotit, například vždy převádět na text.

### Jiné typy klíčů

Klíčem může být hodnota libovolného typu. Dobře fungují typy s hodnotovým porovnáním: text, čísla, datum, `[guid]`, výčtové typy.

```powershell
$svatky = @{}
$svatky[[datetime]'2026-12-24'] = 'Štědrý den'
$svatky[[datetime]'2026-12-25'] = '1. svátek vánoční'

$svatky[[datetime]'2026-12-24']   # Štědrý den
```

Složené objekty, například jiná hashtable, se jako klíč použít dají, ale porovnávají se podle identity objektu, ne podle obsahu. Jiný objekt se stejným obsahem tedy klíč nenajde:

```powershell
$klicTabulka = @{ a = 1 }
$podleObjektu = @{}
$podleObjektu[$klicTabulka] = 'nalezeno'

$podleObjektu[$klicTabulka]           # nalezeno (tentýž objekt)
$null -eq $podleObjektu[@{ a = 1 }]   # True     (jiná tabulka se stejným obsahem)
```

Pole se jako klíč nehodí vůbec: pole v hranatých závorkách PowerShell chápe jako seznam několika klíčů (viz kapitola 3), ne jako jeden klíč.

Pro složený klíč proto raději spojte části do textu:

```powershell
$trzby = @{}
$trzby['Praha|2026'] = 1500000
$trzby['Brno|2026']  = 900000

$mesto = 'Praha'; $rok = 2026
$trzby["$mesto|$rok"]   # 1500000
```

## 9. Vnořené struktury

Hodnotou může být další hashtable nebo pole, takže lze stavět stromové struktury. Je to přirozený způsob zápisu konfigurace.

```powershell
$konfigurace = @{
    Aplikace = 'Sklad'
    Databaze = @{
        Server = 'db01'
        Port   = 5432
        Volby  = @{ SSL = $true; Timeout = 30 }
    }
    Servery = @(
        @{ Jmeno = 'web01'; Role = 'web'; Porty = 80, 443 }
        @{ Jmeno = 'app01'; Role = 'aplikace'; Porty = @(8080) }
    )
}
```

### Čtení

```powershell
$konfigurace.Databaze.Server             # db01
$konfigurace['Databaze']['Volby']['SSL'] # True
$konfigurace.Servery[0].Jmeno            # web01
$konfigurace.Servery[0].Porty[1]         # 443
$konfigurace.Servery.Jmeno               # web01, app01
```

Poslední řádek využívá toho, že tečková notace nad polem vrátí danou hodnotu ze všech jeho prvků.

### Zápis

```powershell
$konfigurace.Databaze.Port = 5433
$konfigurace.Databaze.Volby.Timeout = 60
$konfigurace.Servery += @{ Jmeno = 'db01'; Role = 'databáze'; Porty = @(5432) }
$konfigurace.Servery.Count   # 3
```

Mezilehlá úroveň musí existovat. Přiřazení do klíče, který ještě není hashtable, selže:

```powershell
$strom = @{}

try {
    $strom.Uroven1.Uroven2 = 'hodnota'
} catch {
    'Chyba: Uroven1 neexistuje, nelze do ní zapisovat'
}

$strom.Uroven1 = @{}
$strom.Uroven1.Uroven2 = 'hodnota'
$strom.Uroven1.Uroven2   # hodnota
```

### Bezpečné čtení hluboko do struktury

Čtení neexistující větve chybu nevyvolá, každý krok jen vrátí `$null`:

```powershell
$null -eq $konfigurace.Neexistuje.Take.Ne   # True
```

Výjimkou je přísný režim (`Set-StrictMode`), kde tečková notace na chybějícím klíči selže. Ve skriptech s přísným režimem proto pro volitelné klíče používejte hranaté závorky nebo `Contains`.

### Procházení vnořené struktury

```powershell
foreach ($server in $konfigurace.Servery) {
    "$($server.Jmeno) [$($server.Role)]: porty $($server.Porty -join ', ')"
}
```

### Slovník seznamů

Častý vzor: ke každému klíči sbíráte více hodnot. Před prvním přidáním je potřeba seznam založit:

```powershell
$soubory = 'a.txt', 'b.log', 'c.txt', 'd.csv', 'e.log'
$podlePripony = @{}

foreach ($soubor in $soubory) {
    $pripona = [IO.Path]::GetExtension($soubor)
    if (-not $podlePripony.ContainsKey($pripona)) {
        $podlePripony[$pripona] = [System.Collections.Generic.List[string]]::new()
    }
    $podlePripony[$pripona].Add($soubor)
}

$podlePripony['.txt'] -join ', '   # a.txt, c.txt
$podlePripony['.log'].Count        # 2
```

## 10. Splatting: hashtable jako sada parametrů

Splatting je nejspíš nejčastější využití hashtable v běžných skriptech. Parametry příkazu připravíte do tabulky a předáte je najednou. Rozdíl je v jediném znaku: při předání píšete `@` místo `$`.

Nejdřív pracovní složka, kterou využijí i další příklady:

```powershell
$slozka = Join-Path ([IO.Path]::GetTempPath()) 'hashtable-demo'
New-Item -ItemType Directory -Path $slozka -Force | Out-Null
'první', 'druhý', 'třetí' | ForEach-Object { Set-Content -Path (Join-Path $slozka "$_.txt") -Value "obsah $_" }
Set-Content -Path (Join-Path $slozka 'poznamky.log') -Value 'log'
```

Bez splattingu vznikají dlouhé řádky:

```powershell
Get-ChildItem -Path $slozka -Filter '*.txt' -File -ErrorAction Stop | Select-Object -ExpandProperty Name
```

Se splattingem je každý parametr na vlastním řádku. Klíče odpovídají názvům parametrů, přepínače (switch) dostanou hodnotu `$true`:

```powershell
$parametry = @{
    Path        = $slozka
    Filter      = '*.txt'
    File        = $true
    ErrorAction = 'Stop'
}
Get-ChildItem @parametry | Select-Object -ExpandProperty Name
```

### Podmíněné parametry

Hlavní přínos není v úhlednosti, ale v tom, že sadu parametrů můžete sestavit programově. Bez splattingu byste museli příkaz napsat několikrát v různých větvích `if`:

```powershell
function Get-DemoSoubor {
    param(
        [string]$Cesta,
        [string]$Filtr,
        [switch]$Rekurzivne
    )

    $parametry = @{ Path = $Cesta; File = $true }
    if ($Filtr)      { $parametry.Filter  = $Filtr }
    if ($Rekurzivne) { $parametry.Recurse = $true }

    Get-ChildItem @parametry
}

(Get-DemoSoubor -Cesta $slozka).Count                  # 4
(Get-DemoSoubor -Cesta $slozka -Filtr '*.log').Count   # 1
```

### Kombinace a opakované použití

Splatting lze kombinovat s běžnými parametry a jednu tabulku použít pro více příkazů:

```powershell
$spolecne = @{ ErrorAction = 'Stop'; Encoding = 'utf8' }

Set-Content -Path (Join-Path $slozka 'a.txt') -Value 'A' @spolecne
Set-Content -Path (Join-Path $slozka 'b.txt') -Value 'B' @spolecne
```

### `$PSBoundParameters`

Uvnitř funkce existuje automatická proměnná `$PSBoundParameters`. Je to slovník parametrů, které volající skutečně zadal. Hodí se pro obalové funkce, které parametry předávají dál:

```powershell
function Get-TextovySoubor {
    param(
        [string]$Path,
        [switch]$Recurse
    )
    "Zadané parametry: $($PSBoundParameters.Keys -join ', ')"
    Get-ChildItem @PSBoundParameters -Filter '*.txt' | Select-Object -ExpandProperty Name
}

Get-TextovySoubor -Path $slozka
```

Další hashtable, která řídí chování parametrů, je `$PSDefaultParameterValues`. Nastavuje výchozí hodnoty parametrů pro celou relaci, klíč má tvar `Příkaz:Parametr`:

```powershell
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8'
```

## 11. Hashtable a PSCustomObject

Obě struktury drží pojmenované hodnoty, ale slouží jinému účelu:

| | Hashtable | PSCustomObject |
|---|---|---|
| Účel | slovník, vyhledávání, průběžné změny | záznam, řádek dat |
| Přidání položky | `$h.Novy = 1` | `Add-Member` |
| Pořadí | nezaručené (pokud není `[ordered]`) | zachované |
| Výpis více kusů | každá tabulka zvlášť | jedna tabulka se sloupci |
| `Export-Csv`, `Sort-Object`, `Where-Object` | nefunguje podle očekávání | funguje přirozeně |

### Z hashtable objekt

Přetypování literálu je standardní způsob, jak v PowerShellu vytvořit vlastní objekt. Pořadí vlastností se při tomto zápisu zachová:

```powershell
$osoba = [pscustomobject]@{
    Jmeno = 'Jan'
    Vek   = 30
    Mesto = 'Praha'
}
$osoba.Jmeno   # Jan
```

Pokud přetypujete hotovou hashtable uloženou v proměnné, pořadí vlastností už zaručené není. Kde na něm záleží, stavte tabulku jako `[ordered]`:

```powershell
$data = [ordered]@{ Jmeno = 'Eva'; Vek = 28 }
$data.Mesto = 'Brno'
$osoba2 = [pscustomobject]$data
$osoba2.psobject.Properties.Name -join ', '   # Jmeno, Vek, Mesto
```

Rozdíl mezi oběma strukturami je nejlépe vidět na seznamu. Pole objektů se vypíše jako tabulka se sloupci a jde řadit i exportovat:

```powershell
$lide = @(
    [pscustomobject]@{ Jmeno = 'Jan';   Oddeleni = 'IT';      Plat = 52000 }
    [pscustomobject]@{ Jmeno = 'Eva';   Oddeleni = 'IT';      Plat = 61000 }
    [pscustomobject]@{ Jmeno = 'Karel'; Oddeleni = 'Obchod';  Plat = 48000 }
    [pscustomobject]@{ Jmeno = 'Jana';  Oddeleni = 'Obchod';  Plat = 55000 }
    [pscustomobject]@{ Jmeno = 'Petr';  Oddeleni = 'Finance'; Plat = 58000 }
)

$lide | Sort-Object Plat -Descending | Select-Object -First 2 | Format-Table
```

```text
Jmeno Oddeleni  Plat
----- --------  ----
Eva   IT       61000
Petr  Finance  58000
```

### Z objektu hashtable

Opačný směr jde přes vlastnosti objektu:

```powershell
$zObjektu = @{}
foreach ($vlastnost in $osoba.psobject.Properties) {
    $zObjektu[$vlastnost.Name] = $vlastnost.Value
}
$zObjektu['Mesto']   # Praha
```

### Hashtable jako počáteční hodnoty objektu

Hashtable lze přetypovat i na běžné .NET třídy a vlastní třídy. Klíče se přiřadí do stejnojmenných vlastností:

```powershell
class Server {
    [string]$Jmeno
    [int]$Port = 80
}

$s = [Server]@{ Jmeno = 'web01'; Port = 8080 }
"$($s.Jmeno):$($s.Port)"   # web01:8080
```

## 12. Hashtable jako předpis: vypočítané vlastnosti

Řada příkazů přijímá hashtable ne jako data, ale jako popis toho, co mají udělat. Nejznámější je vypočítaná vlastnost v `Select-Object`. Tabulka má klíč `Name` (název sloupce) a `Expression` (blok kódu, který hodnotu spočítá; `$_` je aktuální objekt):

```powershell
$lide | Select-Object Jmeno, @{ Name = 'RocniPlat'; Expression = { $_.Plat * 12 } } | Format-Table
```

```text
Jmeno RocniPlat
----- ---------
Jan      624000
Eva      732000
Karel    576000
Jana     660000
Petr     696000
```

Klíče lze zkrátit na `n` a `e`. Ve skriptech je čitelnější plný zápis, v konzoli se hodí zkratka:

```powershell
Get-ChildItem -Path $slozka -File |
    Select-Object Name, @{ n = 'Bajtu'; e = { $_.Length } }, @{ n = 'Pripona'; e = { $_.Extension } } |
    Format-Table
```

Stejný princip má `Sort-Object`, kde lze každému kritériu určit vlastní směr řazení:

```powershell
$lide |
    Sort-Object @{ Expression = 'Oddeleni'; Descending = $false }, @{ Expression = 'Plat'; Descending = $true } |
    Format-Table
```

A `Format-Table`, kde tabulka navíc řídí šířku, zarovnání a formát sloupce:

```powershell
$lide | Format-Table Jmeno, @{ Name = 'Plat'; Expression = { $_.Plat }; FormatString = 'N0'; Alignment = 'Right'; Width = 10 }
```

Další místa, kde hashtable slouží jako předpis nebo sada pojmenovaných hodnot:

- `Get-WinEvent -FilterHashtable @{ LogName = 'System'; Level = 2 }` filtruje události Windows.
- `Invoke-RestMethod -Headers @{ Authorization = "Bearer $token" } -Body @{ q = 'hledání' }` předává hlavičky a tělo požadavku.
- `New-Object -TypeName ... -Property @{ ... }` nastavuje vlastnosti nového objektu.
- `Group-Object`, `Measure-Object` a `Compare-Object` přijímají vypočítané vlastnosti stejně jako `Select-Object`.

## 13. Vyhledávací tabulky a seskupování dat

### Index podle jedinečného klíče

Pokud potřebujete v seznamu opakovaně hledat podle ID nebo jména, vyplatí se jednou sestavit index:

```powershell
$podleJmena = @{}
foreach ($clovek in $lide) {
    $podleJmena[$clovek.Jmeno] = $clovek
}

$podleJmena['Karel'].Oddeleni   # Obchod
$podleJmena['Eva'].Plat         # 61000
```

V indexu jsou odkazy na původní objekty, ne kopie. Změna přes index se tedy projeví i v původním poli:

```powershell
$podleJmena['Karel'].Plat = 50000
($lide | Where-Object Jmeno -eq 'Karel').Plat   # 50000
```

### Seskupení: `Group-Object -AsHashTable`

Když klíč není jedinečný, chcete ke každému klíči seznam záznamů. To umí `Group-Object` přímo:

```powershell
$podleOddeleni = $lide | Group-Object -Property Oddeleni -AsHashTable -AsString

$podleOddeleni['IT'].Jmeno -join ', '   # Jan, Eva
$podleOddeleni['Obchod'].Count          # 2
```

Přepínač `-AsString` zajistí, že klíče budou obyčejné řetězce. Bez něj si klíče ponechají původní typ vlastnosti (číslo, výčtovou hodnotu jako stav služby) a hledání textem pak nic nenajde.

Souhrn po skupinách:

```powershell
foreach ($oddeleni in $podleOddeleni.Keys | Sort-Object) {
    $prumer = ($podleOddeleni[$oddeleni] | Measure-Object Plat -Average).Average
    "{0,-8} {1,8:N0}" -f $oddeleni, $prumer
}
```

### Proč na rychlosti záleží

Propojení dvou seznamů pomocí `Where-Object` ve smyčce znamená pro každý záznam projít celý druhý seznam. Počet porovnání roste s druhou mocninou velikosti dat. S indexem se druhý seznam projde jednou a každé hledání je pak okamžité.

```powershell
$zakaznici  = 1..2000 | ForEach-Object { [pscustomobject]@{ Id = $_; Jmeno = "Zákazník $_" } }
$objednavky = 1..2000 | ForEach-Object { [pscustomobject]@{ Cislo = $_; ZakaznikId = (Get-Random -Minimum 1 -Maximum 2001) } }

# Pomalu: pro každou objednávku se prohledá celý seznam zákazníků
$pomalu = Measure-Command {
    foreach ($o in $objednavky) {
        $z = $zakaznici | Where-Object { $_.Id -eq $o.ZakaznikId }
    }
}

# Rychle: jednorázový index, pak přímé hledání
$rychle = Measure-Command {
    $index = @{}
    foreach ($z in $zakaznici) { $index[$z.Id] = $z }
    foreach ($o in $objednavky) {
        $z = $index[$o.ZakaznikId]
    }
}

"Where-Object: {0:N0} ms, hashtable: {1:N0} ms" -f $pomalu.TotalMilliseconds, $rychle.TotalMilliseconds
```

Při ověřování tohoto příkladu (PowerShell 7.4, 2 000 × 2 000 záznamů) trvala první varianta přes 20 sekund a druhá kolem 20 milisekund. Konkrétní čísla závisí na počítači, řádový rozdíl ne, a s rostoucím objemem dat se dál zvětšuje.

## 14. Praktické vzory

### Počítání výskytů

Operátor `++` na neexistujícím klíči začne od nuly, takže není potřeba klíč předem zakládat:

```powershell
$text  = 'kdo chce kam pomozme mu tam kdo chce psa bít hůl si najde kdo'
$pocty = @{}

foreach ($slovo in $text -split '\s+') {
    $pocty[$slovo]++
}

$pocty.GetEnumerator() |
    Sort-Object Value -Descending |
    Select-Object -First 3 |
    ForEach-Object { "$($_.Key): $($_.Value)x" }
```

### Převodní tabulka místo `switch`

Dlouhý `switch` nebo řetěz `if/elseif`, který jen převádí jednu hodnotu na druhou, nahradí tabulka. Data jsou oddělená od logiky a snadno se rozšiřují:

```powershell
$stavy = @{
    200 = 'OK'
    301 = 'Přesměrování'
    404 = 'Nenalezeno'
    500 = 'Chyba serveru'
}

function Get-PopisStavu {
    param([int]$Kod)
    if ($stavy.ContainsKey($Kod)) { $stavy[$Kod] } else { "Neznámý kód $Kod" }
}

Get-PopisStavu 404   # Nenalezeno
Get-PopisStavu 418   # Neznámý kód 418
```

### Tabulka akcí

Hodnotou může být blok kódu. Vznikne tak tabulka, která podle klíče vybere, co se má provést. Blok se spouští operátorem `&`:

```powershell
$operace = @{
    '+' = { param($a, $b) $a + $b }
    '-' = { param($a, $b) $a - $b }
    '*' = { param($a, $b) $a * $b }
    '/' = { param($a, $b) if ($b -eq 0) { 'dělení nulou' } else { $a / $b } }
}

& $operace['+'] 6 3   # 9
& $operace['*'] 6 3   # 18
& $operace['/'] 6 0   # dělení nulou
```

### Výchozí hodnoty a jejich přepsání

Typická úloha: funkce má výchozí nastavení a uživatel z něj přepíše jen něco. Operátor `+` tu nepomůže, protože u shodných klíčů končí chybou. Stačí krátká funkce:

```powershell
function Merge-Hashtable {
    param(
        [hashtable]$Vychozi,
        [hashtable]$Prepsat
    )
    $vysledek = $Vychozi.Clone()
    foreach ($klic in $Prepsat.Keys) {
        $vysledek[$klic] = $Prepsat[$klic]
    }
    $vysledek
}

$vychozi    = @{ Server = 'localhost'; Port = 5432; SSL = $false }
$uzivatel   = @{ Server = 'db01'; SSL = $true }
$nastaveni  = Merge-Hashtable -Vychozi $vychozi -Prepsat $uzivatel

"$($nastaveni.Server):$($nastaveni.Port), SSL=$($nastaveni.SSL)"   # db01:5432, SSL=True
```

### Mezipaměť výsledků

Pokud je výpočet nebo dotaz drahý a opakuje se se stejnými vstupy, výsledek si zapamatujte. Funkce může tabulku z nadřazeného oboru přímo měnit, protože hashtable je odkazový typ (viz kapitola 16):

```powershell
$cache = @{}

function Get-Fibonacci {
    param([int]$N)
    if ($N -le 1) { return [bigint]$N }
    if (-not $cache.ContainsKey($N)) {
        $cache[$N] = (Get-Fibonacci ($N - 1)) + (Get-Fibonacci ($N - 2))
    }
    $cache[$N]
}

Get-Fibonacci 90   # 2880067194370816120
$cache.Count       # 89
```

Bez mezipaměti by tento výpočet rekurzí prakticky neskončil. Stejný vzor se používá pro dotazy do Active Directory, DNS nebo na REST API uvnitř smyčky.

### Vrácení více hodnot z funkce

```powershell
function Get-Statistika {
    param([int[]]$Cisla)
    $m = $Cisla | Measure-Object -Minimum -Maximum -Average
    @{
        Minimum = $m.Minimum
        Maximum = $m.Maximum
        Prumer  = $m.Average
    }
}

$st = Get-Statistika 4, 8, 15, 16, 23, 42
"min $($st.Minimum), max $($st.Maximum), průměr $($st.Prumer)"   # min 4, max 42, průměr 18
```

Pokud má výsledek putovat dál do pipeline nebo se vypisovat, vraťte raději `[pscustomobject]`.

### Odstranění duplicit se zachováním prvního výskytu

```powershell
$vstup  = 'b', 'a', 'B', 'c', 'a', 'd'
$videno = @{}
$unikatni = foreach ($prvek in $vstup) {
    if (-not $videno.ContainsKey($prvek)) {
        $videno[$prvek] = $true
        $prvek
    }
}
$unikatni -join ', '   # b, a, c, d
```

`'B'` vypadlo, protože `@{}` nerozlišuje velikost písmen. S `[hashtable]::new()` by zůstalo.

## 15. Ukládání a načítání: JSON, PSD1, CLIXML

### JSON

Hashtable se na JSON převádí přímo. Pozor na parametr `-Depth`: výchozí hloubka je 2 a hlubší úrovně se nepřevedou správně, místo obsahu se uloží jen název typu. PowerShell 7 na to upozorní varováním, Windows PowerShell 5.1 mlčí.

```powershell
$nastaveniApp = [ordered]@{
    Aplikace = 'Sklad'
    Databaze = [ordered]@{
        Server = 'db01'
        Volby  = [ordered]@{
            SSL      = $true
            Timeouty = [ordered]@{ Pripojeni = 5; Dotaz = 30 }
        }
    }
}

# Výchozí hloubka: nejhlubší úroveň se ztratí
$nastaveniApp | ConvertTo-Json -WarningAction SilentlyContinue

# Správně: hloubku nastavit s rezervou
$json = $nastaveniApp | ConvertTo-Json -Depth 10
$json
```

Uložení a načtení:

```powershell
$souborJson = Join-Path $slozka 'nastaveni.json'
$json | Set-Content -Path $souborJson -Encoding utf8

# PowerShell 6 a novější: rovnou jako hashtable (v novějších verzích se zachovaným pořadím klíčů)
$nacteno = Get-Content -Path $souborJson -Raw | ConvertFrom-Json -AsHashtable
$nacteno.Databaze.Volby.Timeouty.Dotaz   # 30
$nacteno.Contains('Aplikace')            # True
```

Bez `-AsHashtable` vrací `ConvertFrom-Json` objekty `PSCustomObject`. Čtení tečkovou notací funguje stejně, ale nejde přidávat klíče přiřazením ani použít `Contains` a splatting. Ve Windows PowerShellu 5.1, kde přepínač chybí, pomůže převodní funkce:

```powershell
function ConvertTo-Hashtable {
    param($Vstup)

    if ($null -eq $Vstup) { return $null }

    if ($Vstup -is [System.Collections.IDictionary]) {
        $h = @{}
        foreach ($klic in $Vstup.Keys) { $h[$klic] = ConvertTo-Hashtable $Vstup[$klic] }
        return $h
    }
    if ($Vstup -is [System.Management.Automation.PSCustomObject]) {
        $h = @{}
        foreach ($v in $Vstup.psobject.Properties) { $h[$v.Name] = ConvertTo-Hashtable $v.Value }
        return $h
    }
    if ($Vstup -is [System.Collections.IEnumerable] -and $Vstup -isnot [string]) {
        return , @(foreach ($prvek in $Vstup) { ConvertTo-Hashtable $prvek })
    }
    return $Vstup
}

$objekt  = Get-Content -Path $souborJson -Raw | ConvertFrom-Json
$tabulka = ConvertTo-Hashtable $objekt

$tabulka.GetType().Name                 # Hashtable
$tabulka.Databaze.Volby.Timeouty.Dotaz  # 30
```

### Datové soubory PSD1

Soubor `.psd1` obsahuje hashtable zapsanou přímo v syntaxi PowerShellu. Tento formát používají manifesty modulů. Pro konfiguraci má dvě výhody: lze v něm psát komentáře a `Import-PowerShellDataFile` soubor načte bezpečně, tedy bez spuštění kódu (povolené jsou jen konstanty, pole a vnořené tabulky).

```powershell
$souborPsd1 = Join-Path $slozka 'nastaveni.psd1'

@'
@{
    # Připojení k databázi
    Server  = 'db01'
    Port    = 5432
    Tabulky = @('zakaznici', 'objednavky')
    Volby   = @{ SSL = $true }
}
'@ | Set-Content -Path $souborPsd1 -Encoding utf8

$zPsd1 = Import-PowerShellDataFile -Path $souborPsd1
$zPsd1.Port                  # 5432
$zPsd1.Port.GetType().Name   # Int32
$zPsd1.Tabulky[1]            # objednavky
```

Na rozdíl od `ConvertFrom-StringData` se zachovají datové typy. Opačný směr, tedy zápis hashtable do `.psd1`, PowerShell vestavěný nemá.

### CLIXML

`Export-Clixml` uloží data včetně typů (datum zůstane datem, číslo číslem) a hashtable se po načtení vrátí jako hashtable. Formát je určený pro PowerShell, ne pro ruční úpravy ani jiné programy. Hodí se pro mezivýsledky a stav skriptu mezi spuštěními.

```powershell
$souborXml = Join-Path $slozka 'stav.xml'

@{ PosledniBeh = Get-Date; Zpracovano = 128; Chyby = @('a.txt', 'b.txt') } |
    Export-Clixml -Path $souborXml

$stavSkriptu = Import-Clixml -Path $souborXml
$stavSkriptu.GetType().Name               # Hashtable
$stavSkriptu.PosledniBeh.GetType().Name   # DateTime
$stavSkriptu.Zpracovano + 1               # 129
```

### CSV

`Export-Csv` s hashtable přímo nepracuje, vypsal by vlastnosti objektu tabulky místo dat. Položky nejdřív převeďte na objekty:

```powershell
$souborCsv = Join-Path $slozka 'body.csv'

$body.GetEnumerator() |
    Sort-Object Key |
    ForEach-Object { [pscustomobject]@{ Jmeno = $_.Key; Body = $_.Value } } |
    Export-Csv -Path $souborCsv -NoTypeInformation -Encoding utf8

Get-Content -Path $souborCsv
```

A zpět, z CSV do vyhledávací tabulky:

```powershell
$bodyZCsv = @{}
Import-Csv -Path $souborCsv | ForEach-Object { $bodyZCsv[$_.Jmeno] = [int]$_.Body }
$bodyZCsv['Jana']   # 95
```

Přetypování `[int]` je tu nutné, protože z CSV přichází všechno jako text.

## 16. Kopírování a porovnávání

### Přiřazení nekopíruje

Hashtable je odkazový typ. Přiřazením do jiné proměnné nevznikne kopie, obě proměnné ukazují na tutéž tabulku:

```powershell
$original = @{ barva = 'modrá' }
$druha    = $original
$druha.barva = 'červená'

$original.barva   # červená
```

Totéž platí při předání do funkce. Funkce pracuje s původní tabulkou a její změny uvidí i volající:

```powershell
function Set-Zpracovano {
    param([hashtable]$Zaznam)
    $Zaznam.Zpracovano = $true
}

$zaznam = @{ Id = 1 }
Set-Zpracovano -Zaznam $zaznam
$zaznam.Zpracovano   # True
```

Někdy se to hodí (mezipaměť v kapitole 14), jindy je to nečekaný vedlejší účinek.

### Mělká kopie: `Clone()`

`Clone()` vytvoří novou tabulku se stejnými položkami. Kopíruje se ale jen první úroveň. Vnořené tabulky a pole zůstávají sdílené:

```powershell
$vzor = @{
    Jmeno = 'šablona'
    Volby = @{ SSL = $true }
}
$kopie = $vzor.Clone()

$kopie.Jmeno     = 'kopie'    # první úroveň: nezávislá
$kopie.Volby.SSL = $false     # vnořená tabulka: sdílená

$vzor.Jmeno       # šablona
$vzor.Volby.SSL   # False
```

### Hluboká kopie

Pro úplně nezávislou kopii je potřeba projít strukturu rekurzivně:

```powershell
function Copy-HashtableDeep {
    param($Vstup)

    if ($Vstup -is [System.Collections.IDictionary]) {
        $kopie = if ($Vstup -is [System.Collections.Specialized.OrderedDictionary]) { [ordered]@{} } else { @{} }
        foreach ($klic in $Vstup.Keys) { $kopie[$klic] = Copy-HashtableDeep $Vstup[$klic] }
        return $kopie
    }
    if ($Vstup -is [array]) {
        return , @(foreach ($prvek in $Vstup) { Copy-HashtableDeep $prvek })
    }
    return $Vstup
}

$vzor   = @{ Jmeno = 'šablona'; Volby = @{ SSL = $true }; Porty = 80, 443 }
$hlubka = Copy-HashtableDeep $vzor
$hlubka.Volby.SSL = $false

$vzor.Volby.SSL   # True
```

Funkce kopíruje tabulky a pole. Jiné objekty uvnitř (například `PSCustomObject`) zůstanou sdílené.

### Porovnání

Operátor `-eq` u hashtable neporovnává obsah. Dvě tabulky se stejnými položkami se nerovnají:

```powershell
$a = @{ x = 1; y = 2 }
$b = @{ x = 1; y = 2 }
$a -eq $b   # False
```

Obsah jednoúrovňových tabulek porovnáte třeba takto:

```powershell
function Test-HashtableEqual {
    param([hashtable]$Prvni, [hashtable]$Druha)

    if ($Prvni.Count -ne $Druha.Count) { return $false }
    foreach ($klic in $Prvni.Keys) {
        if (-not $Druha.ContainsKey($klic))     { return $false }
        if ($Prvni[$klic] -ne $Druha[$klic])    { return $false }
    }
    $true
}

Test-HashtableEqual $a $b                 # True
Test-HashtableEqual $a @{ x = 1; y = 3 }  # False
```

## 17. Příbuzné typy

### Typovaný slovník `Dictionary[TKey, TValue]`

Hashtable přijme jako klíč i hodnotu cokoli. Generický slovník má typy pevně dané a hlídá je:

```powershell
$vek = [System.Collections.Generic.Dictionary[string, int]]::new()
$vek['Jan'] = 30
$vek['Eva'] = '28'      # text se převede na číslo
$vek['Eva'] + 1         # 29

try {
    $vek['Petr'] = 'neznámý'
} catch {
    'Chyba: hodnotu nelze převést na int'
}
```

Rozdíly oproti `@{}`:

- Klíče **rozlišují velikost písmen**. Pro opačné chování předejte porovnávač:

```powershell
$bezRozliseni = [System.Collections.Generic.Dictionary[string, int]]::new([StringComparer]::OrdinalIgnoreCase)
$bezRozliseni['Jan'] = 30
$bezRozliseni['JAN']   # 30
```

- U velkých objemů jednotně typovaných dat bývá úspornější a rychlejší.
- Při procházení má položka vlastnosti `Key` a `Value` stejně jako u hashtable.

Pro běžné skriptování je `@{}` pohodlnější. Generický slovník se vyplatí, když chcete typovou kontrolu nebo předáváte data do .NET metody, která ho vyžaduje.

### Slovníky pro paralelní zpracování

Hashtable není bezpečná pro souběžný zápis z více vláken. Při použití `ForEach-Object -Parallel` (PowerShell 7) nebo runspaces použijte `ConcurrentDictionary`:

```powershell
$vysledky = [System.Collections.Concurrent.ConcurrentDictionary[string, int]]::new()

1..5 | ForEach-Object -Parallel {
    $slovnik = $using:vysledky
    $null = $slovnik.TryAdd("uloha$_", $_ * $_)
} -ThrottleLimit 5

$vysledky.Count      # 5
$vysledky['uloha4']  # 16
```

### Přehled

| Typ | Zápis | Pořadí | Velikost písmen v klíčích |
|---|---|---|---|
| `Hashtable` | `@{}` | nezaručené | nerozlišuje |
| `Hashtable` | `[hashtable]::new()` | nezaručené | rozlišuje |
| `OrderedDictionary` | `[ordered]@{}` | podle vložení | nerozlišuje |
| `Dictionary[K,V]` | `[...Dictionary[string,int]]::new()` | nezaručené | rozlišuje |
| `ConcurrentDictionary[K,V]` | `[...ConcurrentDictionary[string,int]]::new()` | nezaručené | rozlišuje |

## 18. Nejčastější chyby

| Chyba | Co se stane | Správně |
|---|---|---|
| `"$klic: $hodnota"` | chyba parseru kvůli dvojtečce | `"${klic}: $hodnota"` |
| `"Hodnota: $h.klic"` | vypíše název typu a `.klic` | `"Hodnota: $($h.klic)"` |
| `$h \| Where-Object ...` | tabulka projde jako jeden objekt | `$h.GetEnumerator() \| Where-Object ...` |
| změna tabulky ve `foreach ($k in $h.Keys)` | chyba „Collection was modified" | `foreach ($k in @($h.Keys))` |
| `if ($h[$k])` jako test existence | selže u hodnot `0`, `$false`, `''` | `$h.ContainsKey($k)` nebo `$h.Contains($k)` |
| `if ($h)` jako test neprázdnosti | prázdná tabulka je pravdivá | `$h.Count -gt 0` |
| `$kopie = $h` | obě proměnné sdílejí jednu tabulku | `$h.Clone()` nebo hluboká kopie |
| spoléhání na pořadí v `@{}` | pořadí je libovolné | `[ordered]@{}` |
| `$h.Keys[0]` | vrátí všechny klíče | `@($h.Keys)[0]` |
| hledání `'42'` v tabulce s klíčem `42` | nenalezeno | sjednotit typ klíčů |
| `[hashtable]::new()` místo `@{}` | klíče rozlišují velikost písmen | `@{}`, pokud to není záměr |
| `ConvertTo-Json` bez `-Depth` | hlubší úrovně se ztratí | `ConvertTo-Json -Depth 10` |
| `$a + $b` se shodnými klíči | chyba duplicitního klíče | sloučení smyčkou (kapitola 14) |
| `$h.ContainsKey()` u `[ordered]` | metoda neexistuje | `$h.Contains()` |

## 19. Tahák

| Operace | Zápis |
|---|---|
| Vytvoření | `$h = @{ a = 1; b = 2 }` |
| Vytvoření se zachovaným pořadím | `$h = [ordered]@{ a = 1; b = 2 }` |
| Čtení | `$h['a']`, `$h.a` |
| Přidání nebo přepsání | `$h['c'] = 3`, `$h.c = 3` |
| Přidání s chybou při duplicitě | `$h.Add('c', 3)` |
| Odstranění | `$h.Remove('a')` |
| Vyprázdnění | `$h.Clear()` |
| Počet položek | `$h.Count` |
| Existence klíče | `$h.ContainsKey('a')`, `$h.Contains('a')` |
| Existence hodnoty | `$h.ContainsValue(1)` |
| Klíče a hodnoty | `$h.Keys`, `$h.Values` |
| Procházení | `foreach ($p in $h.GetEnumerator()) { $p.Key; $p.Value }` |
| Řazení podle klíče | `$h.GetEnumerator() \| Sort-Object Key` |
| Sloučení bez shodných klíčů | `$a + $b` |
| Mělká kopie | `$h.Clone()` |
| Splatting | `Příkaz @h` |
| Převod na objekt | `[pscustomobject]$h` |
| Do JSON a zpět | `ConvertTo-Json -Depth 10`, `ConvertFrom-Json -AsHashtable` |
| Načtení `.psd1` | `Import-PowerShellDataFile -Path soubor.psd1` |
| Seskupení dat | `$data \| Group-Object Vlastnost -AsHashTable -AsString` |
| Vypočítaný sloupec | `Select-Object @{ Name = 'X'; Expression = { ... } }` |
