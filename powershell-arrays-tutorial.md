# Pole (arrays) v PowerShellu: kompletní průvodce

Pole je nejzákladnější datová struktura PowerShellu a zároveň ta, se kterou pracujete, i když o tom nevíte. Kdykoli příkaz vrátí víc než jeden výsledek a vy ho uložíte do proměnné, dostanete pole. Proto se vyplatí rozumět nejen zápisu, ale i tomu, jak se pole chová v pipeline, proč se někdy „samo rozbalí" a proč je operátor `+=` ve velkých smyčkách pomalý.

Příklady jsou ověřené v PowerShellu 7.4. Naprostá většina funguje stejně i ve Windows PowerShellu 5.1; kde je potřeba novější verze, je to u příkladu uvedeno.

Tutoriál navazuje na průvodce hashtables. Hashtable hledá hodnotu podle klíče, pole drží hodnoty v pořadí a hledá podle pozice.

## Obsah

1. [Co je pole a kdy ho použít](#1-co-je-pole-a-kdy-ho-použít)
2. [Vytvoření](#2-vytvoření)
3. [Čtení prvků](#3-čtení-prvků)
4. [Změna, přidávání a odebírání](#4-změna-přidávání-a-odebírání)
5. [Procházení](#5-procházení)
6. [Hledání a filtrování](#6-hledání-a-filtrování)
7. [Operátory nad polem](#7-operátory-nad-polem)
8. [Řazení, unikátní hodnoty a statistika](#8-řazení-unikátní-hodnoty-a-statistika)
9. [Pole a pipeline: rozbalování](#9-pole-a-pipeline-rozbalování)
10. [Pole objektů](#10-pole-objektů)
11. [Vnořená a vícerozměrná pole](#11-vnořená-a-vícerozměrná-pole)
12. [Typovaná pole](#12-typovaná-pole)
13. [Kopírování a porovnávání](#13-kopírování-a-porovnávání)
14. [Výkon: proč je `+=` pomalé](#14-výkon-proč-je--pomalé)
15. [Příbuzné typy: List, HashSet, Queue, Stack](#15-příbuzné-typy-list-hashset-queue-stack)
16. [Praktické vzory](#16-praktické-vzory)
17. [Pole a soubory: text, CSV, JSON](#17-pole-a-soubory-text-csv-json)
18. [Nejčastější chyby](#18-nejčastější-chyby)
19. [Tahák](#19-tahák)

---

## 1. Co je pole a kdy ho použít

Pole je uspořádaná řada hodnot. Každá hodnota má pořadové číslo (index), které začíná nulou.

```powershell
$ovoce = 'jablko', 'hruška', 'švestka'

$ovoce[0]                   # jablko
$ovoce.Count                # 3
$ovoce.GetType().FullName   # System.Object[]
```

Typ `System.Object[]` říká, že jde o pole obecných objektů. Do jednoho pole proto můžete uložit hodnoty různých typů:

```powershell
$smes = 42, 'text', (Get-Date), $true, $null
$smes[0].GetType().Name   # Int32
$smes[1].GetType().Name   # String
```

Čtyři vlastnosti, které se v tutoriálu vracejí:

- **Pořadí je zaručené.** Prvky zůstávají tam, kam jste je dali.
- **Přístup přes index je okamžitý**, hledání podle hodnoty naopak znamená projít pole prvek po prvku.
- **Velikost je pevná.** Pole nejde zvětšit ani zmenšit. Operace, které vypadají jako přidání prvku, ve skutečnosti vytvoří nové pole (kapitola 4).
- **Hodnoty se mohou opakovat.**

Kdy sáhnout po které struktuře:

| Potřebuji | Vhodná struktura |
|---|---|
| Seznam hodnot v daném pořadí, který se po vytvoření moc nemění | pole `@()` |
| Seznam, do kterého průběžně přidávám nebo z něj odebírám | `List[T]` |
| Rychle zjistit, jestli hodnota v sadě je; jen unikátní hodnoty | `HashSet[T]` |
| Najít hodnotu podle názvu nebo ID | hashtable `@{}` |
| Frontu úkolů nebo zásobník | `Queue[T]`, `Stack[T]` |

## 2. Vytvoření

### Čárka a `@()`

Pole vytváří čárka, ne závorky. Zápis `@( )` je navíc operátor, který z čehokoli uvnitř udělá pole:

```powershell
$cisla  = 1, 2, 3
$stejne = @(1, 2, 3)
```

Ve víceřádkovém zápisu uvnitř `@( )` čárky nejsou potřeba, každý řádek je jeden prvek:

```powershell
$mesta = @(
    'Praha'
    'Brno'
    'Ostrava'
)
$mesta.Count   # 3
```

### Prázdné a jednoprvkové pole

```powershell
$prazdne = @()
$prazdne.Count   # 0

$jeden  = @('jediný')
$jeden2 = , 'jediný'      # unární čárka: pole s jedním prvkem
$jeden.Count              # 1

$neniPole = 'jediný'
$neniPole -is [array]     # False
$jeden -is [array]        # True
```

Hodnota bez čárky a bez `@()` pole není. To je důležité u výsledků příkazů, kde předem nevíte, kolik jich přijde (kapitola 9).

### Rozsahy

Operátor `..` vytvoří pole po sobě jdoucích celých čísel, i sestupně:

```powershell
(1..5) -join ' '     # 1 2 3 4 5
(5..1) -join ' '     # 5 4 3 2 1
(-2..2) -join ' '    # -2 -1 0 1 2
```

Od PowerShellu 6 fungují rozsahy i pro znaky:

```powershell
('a'..'e') -join ' '   # a b c d e
```

Rozsah nikdy není prázdný. `1..0` není „nic", ale pole `1, 0`. Na to se snadno narazí ve smyčce přes indexy prázdného pole:

```powershell
$zadnaData = @()
(0..($zadnaData.Count - 1)) -join ' '   # 0 -1
```

### Pole předem dané velikosti

```powershell
$nuly = @(0) * 5
$nuly -join ' '              # 0 0 0 0 0

$typovane = [int[]]::new(5)  # pět nul, pole typu int
$typovane.Count              # 5

$texty = [string[]]::new(3)  # tři prázdné pozice ($null)
$null -eq $texty[0]          # True
```

### Pole z výstupu příkazu a z textu

Operátor `@( )` posbírá vše, co příkazy uvnitř vypíšou:

```powershell
$vystup = @(
    'začátek'
    1..3
    'konec'
)
$vystup.Count   # 5
```

Všimněte si, že rozsah `1..3` se do výsledku rozbalil jako tři samostatné prvky. Vnořené pole se zapisuje jinak (kapitola 11).

Text se na pole dělí operátorem `-split`:

```powershell
$casti = 'jablko,hruška,švestka' -split ','
$casti.Count    # 3

$slova = -split '  dvě   slova  navíc '   # unární -split dělí podle bílých znaků
$slova -join '|'                          # dvě|slova|navíc
```

## 3. Čtení prvků

Pro příklady v této kapitole:

```powershell
$pismena = 'a', 'b', 'c', 'd', 'e'
```

### Index od začátku a od konce

```powershell
$pismena[0]    # a   (první)
$pismena[2]    # c
$pismena[-1]   # e   (poslední)
$pismena[-2]   # d   (předposlední)
```

### Více indexů a řezy

Do hranatých závorek lze dát pole indexů nebo rozsah. Výsledkem je nové pole:

```powershell
$pismena[0, 2, 4] -join ' '    # a c e
$pismena[1..3] -join ' '       # b c d
$pismena[-3..-1] -join ' '     # c d e   (poslední tři)
$pismena[3..1] -join ' '       # d c b   (v obráceném pořadí)
```

Řez „od indexu 2 do konce" se nedá zapsat jako `2..-1`. Rozsah `2..-1` znamená čísla 2, 1, 0, -1, takže dostanete něco úplně jiného:

```powershell
$pismena[2..-1] -join ' '                      # c b a e
$pismena[2..($pismena.Count - 1)] -join ' '    # c d e
```

Čitelnější alternativa pro začátek a konec pole je `Select-Object`:

```powershell
($pismena | Select-Object -First 2) -join ' '      # a b
($pismena | Select-Object -Last 2) -join ' '       # d e
($pismena | Select-Object -Skip 2) -join ' '       # c d e
($pismena | Select-Object -SkipLast 1) -join ' '   # a b c d
($pismena | Select-Object -Index 0, 4) -join ' '   # a e
```

### Index mimo rozsah

Čtení neexistujícího indexu chybu nevyvolá, vrátí `$null`. Řez přes hranici pole vrátí jen to, co existuje:

```powershell
$null -eq $pismena[10]        # True
$pismena[3..10] -join ' '     # d e
```

V přísném režimu je čtení mimo rozsah chyba:

```powershell
& {
    Set-StrictMode -Version Latest
    $p = 'a', 'b'
    try { $p[10] } catch { 'Přísný režim: index mimo rozsah je chyba' }
}
```

### Počet prvků

```powershell
$pismena.Count    # 5
$pismena.Length   # 5
```

Obě vlastnosti vrací u pole totéž. `Count` je univerzálnější, protože ji PowerShell doplňuje i jednotlivým hodnotám a `$null`:

```powershell
$jednaHodnota = 'text'
$jednaHodnota.Count   # 1
$nic = $null
$nic.Count            # 0
```

`Length` u textu naopak vrátí počet znaků, takže se s ní u proměnné, která může být polem i jedním řetězcem, snadno spletete.

### Pole v textovém řetězci

Pole vložené do dvojitých uvozovek se spojí mezerami. Přístup k prvku je potřeba obalit do `$( )`:

```powershell
"Písmena: $pismena"            # Písmena: a b c d e
"První: $($pismena[0])"        # První: a
"Počet: $($pismena.Count)"     # Počet: 5
```

Bez `$( )` PowerShell dosadí celé pole a zbytek zápisu vypíše jako obyčejný text:

```powershell
"První: $pismena[0]"       # První: a b c d e[0]
"Počet: $pismena.Count"    # Počet: a b c d e.Count
```

## 4. Změna, přidávání a odebírání

### Změna prvku

```powershell
$barvy = 'červená', 'zelená', 'modrá'
$barvy[1] = 'žlutá'
$barvy -join ', '   # červená, žlutá, modrá
```

Zápis mimo rozsah je na rozdíl od čtení chyba, protože pole se nemůže zvětšit:

```powershell
try {
    $barvy[5] = 'fialová'
} catch {
    'Chyba: index je mimo hranice pole'
}
```

### Pole má pevnou velikost

Metody `Add` a `Remove` pole sice má, ale obě skončí chybou:

```powershell
$barvy.IsFixedSize   # True

try {
    $barvy.Add('fialová')
} catch {
    'Chyba: kolekce má pevnou velikost'
}
```

### Přidání prvku: `+=` vytvoří nové pole

```powershell
$barvy += 'fialová'
$barvy.Count   # 4

$barvy += 'bílá', 'černá'   # lze přidat i více prvků najednou
$barvy.Count   # 6
```

Vypadá to jako přidání, ale PowerShell ve skutečnosti vytvoří nové, o prvek delší pole, zkopíruje do něj všechny původní prvky a staré pole zahodí. Pro pár prvků je to v pořádku. Ve smyčce s tisíci opakováními jde o známou výkonnostní past (kapitola 14).

Stejně funguje operátor `+`, který spojí dvě pole do nového:

```powershell
$teple   = 'červená', 'oranžová'
$studene = 'modrá', 'zelená'
$vsechny = $teple + $studene
$vsechny.Count   # 4
```

### Odebrání prvku

Odebrat prvek z pole nejde, lze jen vytvořit nové pole bez něj. Nejčastěji filtrem:

```powershell
$bezModre = $vsechny | Where-Object { $_ -ne 'modrá' }
$bezModre -join ', '   # červená, oranžová, zelená
```

Odebrání podle pozice jde složením dvou řezů:

```powershell
$pismena = 'a', 'b', 'c', 'd', 'e'
$bezTretiho = $pismena[0..1 + 3..4]
$bezTretiho -join ' '   # a b d e
```

### Vložení doprostřed

```powershell
$sVlozenym = $pismena[0..1] + 'X' + $pismena[2..4]
$sVlozenym -join ' '   # a b X c d e
```

Pokud přidáváte, odebíráte nebo vkládáte často, pole není správná struktura. Použijte `List[T]`, který tyto operace umí přímo (kapitola 15).

## 5. Procházení

### `foreach`

Nejčitelnější a nejrychlejší způsob, pokud nepotřebujete index:

```powershell
$mesta = 'Praha', 'Brno', 'Ostrava'

foreach ($mesto in $mesta) {
    "Město: $mesto"
}
```

### `for` s indexem

Použijte, když potřebujete znát pozici, měnit prvky nebo procházet dvě pole souběžně:

```powershell
for ($i = 0; $i -lt $mesta.Count; $i++) {
    "$($i + 1). $($mesta[$i])"
}
```

### `ForEach-Object` v pipeline

```powershell
$mesta | ForEach-Object { $_.ToUpper() }
```

`ForEach-Object` zpracovává prvky postupně, jak pipeline přicházejí, takže nepotřebuje celé pole v paměti. U velkých dat ze souboru nebo ze sítě je to výhoda. Smyčka `foreach` je naopak rychlejší nad daty, která už v paměti jsou.

### Metoda `.ForEach()`

Pole má i vlastní metodu, která je rychlejší než `ForEach-Object` a umí několik zkratek:

```powershell
$mesta.ForEach({ $_.Length }) -join ' '    # 5 4 7
$mesta.ForEach('ToLower') -join ' '        # praha brno ostrava   (zavolá metodu)
('1', '2', '3').ForEach([int]) -join ' '   # 1 2 3                (převede typ)
```

### Pozpátku

```powershell
for ($i = $mesta.Count - 1; $i -ge 0; $i--) {
    $mesta[$i]
}
```

### Změna prvků během procházení

Proměnná ve `foreach` je kopie hodnoty. Přiřazením do ní pole nezměníte:

```powershell
$ceny = 100, 200, 300

foreach ($cena in $ceny) {
    $cena = $cena * 2
}
$ceny -join ' '   # 100 200 300
```

Pro změnu na místě použijte `for` a index:

```powershell
for ($i = 0; $i -lt $ceny.Count; $i++) {
    $ceny[$i] = $ceny[$i] * 2
}
$ceny -join ' '   # 200 400 600
```

Nebo vytvořte nové pole z výstupu smyčky, což je v PowerShellu obvyklejší:

```powershell
$sDph = foreach ($cena in $ceny) { $cena * 1.21 }
$sDph -join ' '   # 242 484 726
```

### Prázdné pole a `$null`

`foreach` nad prázdným polem ani nad `$null` neproběhne ani jednou. `ForEach-Object` se chová jinak: `$null` poslaný do pipeline je jeden prvek.

```powershell
foreach ($x in $null) { 'foreach: tohle se nevypíše' }

$null | ForEach-Object { 'ForEach-Object: proběhne jednou s $null' }
@()   | ForEach-Object { 'tohle se nevypíše' }
```

## 6. Hledání a filtrování

### Je hodnota v poli? `-contains` a `-in`

```powershell
$povolene = 'txt', 'csv', 'json'

$povolene -contains 'csv'      # True
'csv' -in $povolene            # True   (totéž, jen obráceně)
$povolene -notcontains 'exe'   # True
'CSV' -in $povolene            # True   (velikost písmen se nerozlišuje)
$povolene -ccontains 'CSV'     # False  (varianta rozlišující velikost písmen)
```

`-in` se dobře čte v podmínkách: `if ($pripona -in $povolene)`.

Pozor na záměnu s `-like` a `-match`. `-contains` hledá celý prvek, ne část textu:

```powershell
'Praha', 'Brno' -contains 'Pra'   # False
```

### Pozice prvku

```powershell
$povolene.IndexOf('json')           # 2
$povolene.IndexOf('exe')            # -1   (nenalezeno)
$povolene.IndexOf('JSON')           # -1   (IndexOf rozlišuje velikost písmen)
```

Pozici prvního prvku, který splňuje podmínku, najde `[array]::FindIndex`:

```powershell
$teploty = 12, 18, 25, 31, 22
[array]::FindIndex($teploty, [Predicate[object]] { param($t) $t -gt 20 })   # 2
```

### Filtrování: `Where-Object` a `.Where()`

```powershell
$cisla = 1..10

($cisla | Where-Object { $_ % 2 -eq 0 }) -join ' '   # 2 4 6 8 10
$cisla.Where({ $_ % 2 -eq 0 }) -join ' '             # 2 4 6 8 10
```

Metoda `.Where()` je rychlejší a navíc má režimy, které `Where-Object` neumí:

```powershell
$cisla.Where({ $_ -gt 3 }, 'First') -join ' '       # 4          (první vyhovující)
$cisla.Where({ $_ -gt 3 }, 'First', 2) -join ' '    # 4 5        (první dva)
$cisla.Where({ $_ -gt 3 }, 'Last') -join ' '        # 10         (poslední vyhovující)
$cisla.Where({ $_ -gt 3 }, 'Until') -join ' '       # 1 2 3      (dokud podmínka neplatí)
$cisla.Where({ $_ -gt 7 }, 'SkipUntil') -join ' '   # 8 9 10     (od prvního vyhovujícího)
```

Režim `Split` rozdělí pole na dvě části, vyhovující a nevyhovující:

```powershell
$suda, $licha = $cisla.Where({ $_ % 2 -eq 0 }, 'Split')
$suda -join ' '    # 2 4 6 8 10
$licha -join ' '   # 1 3 5 7 9
```

Rozdíl proti `Where-Object`: `.Where()` vrací vždy kolekci, i když vyhoví jediný prvek nebo žádný. Odpadá tím problém s rozbalováním z kapitoly 9.

### Porovnávací operátory filtrují

Pokud je vlevo od porovnávacího operátoru pole, výsledkem není `True` nebo `False`, ale pole prvků, které podmínce vyhovují:

```powershell
$znamky = 1, 3, 2, 5, 1, 4

($znamky -eq 1) -join ' '     # 1 1
($znamky -gt 3) -join ' '     # 5 4
($znamky -ne 1) -join ' '     # 3 2 5 4
($znamky -eq 1).Count         # 2   (počet jedniček)
```

Totéž platí pro `-like`, `-match` a jejich negace:

```powershell
$soubory = 'report.csv', 'data.json', 'report-2025.csv', 'README.md'

($soubory -like '*.csv') -join ', '       # report.csv, report-2025.csv
($soubory -match '^report') -join ', '    # report.csv, report-2025.csv
($soubory -notlike '*.csv') -join ', '    # data.json, README.md
```

### Past: test na `$null`

Z předchozího plyne častá chyba. Výraz `$pole -eq $null` neříká, jestli je proměnná `$null`. Vrátí prvky pole, které jsou `$null`:

```powershell
$data = @($null, $null)

if ($data -eq $null) { 'Vypadá to jako $null, ale je to pole se dvěma prvky' }
if ($null -eq $data) { 'tohle se nevypíše' } else { 'Správný test: proměnná $null není' }
```

Proto se `$null` píše vždy vlevo: `$null -eq $promenna`.

### Pole v podmínce

Prázdné pole je nepravdivé. Pole s jedním prvkem se vyhodnotí podle toho prvku. Pole se dvěma a více prvky je vždy pravdivé:

```powershell
if (@())      { 'ano' } else { 'prázdné pole: ne' }
if (@(0))     { 'ano' } else { 'pole s jednou nulou: ne' }
if (@(0, 0))  { 'pole se dvěma nulami: ano' }
```

Na „je něco v poli" se proto ptejte přes `Count`:

```powershell
$vysledky = @(0)
if ($vysledky.Count -gt 0) { 'pole není prázdné' }
```

## 7. Operátory nad polem

### Spojení a opakování

```powershell
$a = 1, 2
$b = 3, 4

($a + $b) -join ' '    # 1 2 3 4
($a + 5) -join ' '     # 1 2 5
($a * 3) -join ' '     # 1 2 1 2 1 2
```

Sčítání čísel v poli operátor `+` nedělá, na to je `Measure-Object` (kapitola 8).

### `-join`: pole na text

```powershell
$casti = 'C:', 'Data', 'Export'

$casti -join '\'            # C:\Data\Export
$casti -join ', '           # C:, Data, Export
-join ('a', 'b', 'c')       # abc   (unární -join spojí bez oddělovače)
```

### `-split`: text na pole

```powershell
('2026-10-08' -split '-') -join ' | '          # 2026 | 10 | 08
('a, b;c' -split '[,;]\s*') -join ' | '        # a | b | c   (oddělovač je regulární výraz)
('klíč=hodnota=další' -split '=', 2) -join ' | '   # klíč | hodnota=další   (nejvýše 2 části)
```

### Textové operátory na všech prvcích

`-replace` a formátovací operace se provedou na každém prvku zvlášť:

```powershell
$nazvy = 'report.txt', 'data.txt'

($nazvy -replace '\.txt$', '.bak') -join ', '   # report.bak, data.bak
```

### Formátovací operátor `-f`

Pole vpravo od `-f` dodá hodnoty pro zástupné znaky `{0}`, `{1}` a další:

```powershell
$udaje = 'Jan', 30, 'Praha'
'{0} ({1} let), {2}' -f $udaje   # Jan (30 let), Praha
```

### Volání metody a vlastnosti na všech prvcích

Když na poli zavoláte vlastnost nebo metodu, kterou pole samo nemá, PowerShell ji zavolá na každém prvku a vrátí pole výsledků:

```powershell
$jmena = 'jan', 'eva', 'petr'

$jmena.ToUpper() -join ' '          # JAN EVA PETR
$jmena.Substring(0, 1) -join ' '    # j e p
```

Výjimkou jsou vlastnosti, které pole má samo, hlavně `Length` a `Count`. `$jmena.Length` vrátí počet prvků, ne délky jednotlivých jmen:

```powershell
$jmena.Length                          # 3
$jmena.ForEach('Length') -join ' '     # 3 3 4
```

### Přiřazení do více proměnných

Pole lze při přiřazení rozdělit do několika proměnných. Poslední proměnná dostane zbytek:

```powershell
$prvni, $druhy, $ostatni = 'a', 'b', 'c', 'd', 'e'
$prvni              # a
$druhy              # b
$ostatni -join ' '  # c d e
```

Hodí se pro oddělení hlavičky od dat a pro prohození hodnot bez pomocné proměnné:

```powershell
$hlavicka, $radky = 'Jmeno;Vek', 'Jan;30', 'Eva;28'
$hlavicka       # Jmeno;Vek
$radky.Count    # 2

$x = 1; $y = 2
$x, $y = $y, $x
"$x $y"         # 2 1
```

## 8. Řazení, unikátní hodnoty a statistika

### Řazení

`Sort-Object` vrací nové seřazené pole, původní nemění:

```powershell
$body = 72, 95, 58, 88, 95

($body | Sort-Object) -join ' '               # 58 72 88 95 95
($body | Sort-Object -Descending) -join ' '   # 95 95 88 72 58
$body -join ' '                               # 72 95 58 88 95   (beze změny)
```

Řadit lze podle vlastnosti nebo vypočítané hodnoty:

```powershell
$slova = 'švestka', 'fík', 'jablko', 'kiwi'

($slova | Sort-Object Length) -join ' '                     # fík kiwi jablko švestka
($slova | Sort-Object { $_[-1] }) -join ' '                 # švestka kiwi fík jablko   (podle posledního písmena)
```

Past: čísla uložená jako text se řadí abecedně. Typické u dat z CSV nebo z názvů souborů:

```powershell
$textovaCisla = '10', '9', '100', '1'

($textovaCisla | Sort-Object) -join ' '              # 1 10 100 9
($textovaCisla | Sort-Object { [int]$_ }) -join ' '  # 1 9 10 100
```

Řazení na místě, bez vytváření nového pole, umí statické metody třídy `[array]`:

```powershell
$naMiste = 3, 1, 2
[array]::Sort($naMiste)
$naMiste -join ' '      # 1 2 3

[array]::Reverse($naMiste)
$naMiste -join ' '      # 3 2 1
```

### Unikátní hodnoty

Tři způsoby se liší v pořadí výsledku a v zacházení s velikostí písmen:

```powershell
$duplicity = 'b', 'a', 'B', 'c', 'a'

($duplicity | Select-Object -Unique) -join ' '   # b a B c   (zachová pořadí, rozlišuje velikost písmen)
($duplicity | Sort-Object -Unique) -join ' '     # a b c     (seřadí, velikost písmen nerozlišuje)
```

Pro velká pole je výrazně rychlejší `HashSet` (kapitola 15).

### Statistika

```powershell
$m = $body | Measure-Object -Sum -Average -Minimum -Maximum

$m.Count      # 5
$m.Sum        # 408
$m.Average    # 81.6
$m.Minimum    # 58
$m.Maximum    # 95
```

### Počty výskytů

```powershell
$body | Group-Object | Sort-Object Count -Descending | Select-Object -First 1 | ForEach-Object { "Nejčastější: $($_.Name) ($($_.Count)x)" }
```

### Náhodný výběr a zamíchání

```powershell
$balicek = 1..10

$jedna    = $balicek | Get-Random             # jeden náhodný prvek
$tri      = $balicek | Get-Random -Count 3    # tři různé prvky
$zamichano = $balicek | Get-Random -Count $balicek.Count   # celé pole v náhodném pořadí

$tri.Count         # 3
$zamichano.Count   # 10
```

### Množinové operace

```powershell
$tymA = 'Jan', 'Eva', 'Petr', 'Jana'
$tymB = 'Eva', 'Karel', 'Jana', 'Ota'

# sjednocení
($tymA + $tymB | Select-Object -Unique) -join ', '       # Jan, Eva, Petr, Jana, Karel, Ota

# průnik: kdo je v obou
($tymA | Where-Object { $_ -in $tymB }) -join ', '       # Eva, Jana

# rozdíl: kdo je jen v A
($tymA | Where-Object { $_ -notin $tymB }) -join ', '    # Jan, Petr
```

`Compare-Object` ukáže rozdíly oběma směry najednou. `<=` znamená „jen v prvním", `=>` „jen ve druhém":

```powershell
Compare-Object -ReferenceObject $tymA -DifferenceObject $tymB |
    Sort-Object SideIndicator, InputObject |
    ForEach-Object { "$($_.SideIndicator) $($_.InputObject)" }
```

```text
<= Jan
<= Petr
=> Karel
=> Ota
```

## 9. Pole a pipeline: rozbalování

Tohle je část, kde se PowerShell nejvíc liší od jiných jazyků, a zdroj většiny záhadných chyb s poli.

### Pipeline posílá prvky, ne pole

Když pole pošlete do pipeline nebo ho funkce vrátí, PowerShell ho rozbalí a posílá prvky jeden po druhém. Příjemce o původním poli neví. Pokud výsledek uložíte do proměnné, PowerShell z přijatých prvků složí pole znovu, ale jen pokud jsou aspoň dva:

| Počet výsledků | Co je v proměnné |
|---|---|
| 0 | `$null` |
| 1 | přímo ta jedna hodnota |
| 2 a více | pole |

```powershell
function Get-Suda {
    param([int[]]$Cisla)
    $Cisla | Where-Object { $_ % 2 -eq 0 }
}

$vice  = Get-Suda 1, 2, 3, 4
$jedno = Get-Suda 1, 2, 3
$zadne = Get-Suda 1, 3

$vice.GetType().Name    # Object[]
$jedno.GetType().Name   # Int32
$null -eq $zadne        # True
```

V praxi to znamená, že skript funguje se dvěma soubory, ale selže s jedním: `$vysledek[0]` u textu vrátí první znak, `$vysledek.Length` počet znaků, a metoda, kterou má jen pole, neexistuje.

```powershell
$nalezene = 'jediny-soubor.txt', 'jiny.log' | Where-Object { $_ -like '*.txt' }
$nalezene[0]   # j   (první znak, ne první soubor)
```

### Řešení: obalit do `@()`

`@( )` zaručí pole při libovolném počtu výsledků. Pole nechá být, jednu hodnotu zabalí, z ničeho udělá prázdné pole:

```powershell
$nalezene = @('jediny-soubor.txt', 'jiny.log' | Where-Object { $_ -like '*.txt' })
$nalezene[0]       # jediny-soubor.txt
$nalezene.Count    # 1

@(Get-Suda 1, 3).Count   # 0
```

Pravidlo: kdykoli ukládáte výsledek příkazu a budete s ním pracovat jako s polem (index, `Count`, `+=`), obalte ho do `@()`.

Druhá možnost je typ u proměnné, který převod na pole vynutí při každém přiřazení:

```powershell
[array]$vzdyPole = Get-Suda 1, 2, 3
$vzdyPole.GetType().Name   # Object[]
$vzdyPole.Count            # 1
```

### Sběr výstupu smyčky

Stejný mechanismus se dá využít. Vše, co smyčka nebo podmínka vypíše, lze přiřadit do proměnné. Je to nejjednodušší a zároveň rychlý způsob, jak postavit pole:

```powershell
$druheMocniny = foreach ($n in 1..5) { $n * $n }
$druheMocniny -join ' '   # 1 4 9 16 25

$jenVelke = foreach ($n in 1..10) {
    if ($n -gt 7) { "číslo $n" }
}
$jenVelke -join ', '   # číslo 8, číslo 9, číslo 10
```

I tady platí pravidlo o jednom výsledku, takže při nejistotě `@(foreach ...)`.

### Vrácení pole jako celku

Někdy rozbalení nechcete, například když funkce má vrátit pole i s jedním prvkem nebo pole polí. Unární čárka zabalí pole do dalšího, jednoprvkového pole. Pipeline rozbalí jen ten vnější obal:

```powershell
function Get-Seznam {
    $seznam = @('jediný')
    , $seznam
}

(Get-Seznam).GetType().Name   # Object[]
(Get-Seznam).Count            # 1
```

Totéž čitelněji: `Write-Output -NoEnumerate $seznam`.

Má to svou cenu. Kdo takovou funkci použije v pipeline, dostane celé pole jako jeden objekt:

```powershell
(Get-Seznam | Measure-Object).Count   # 1  (jedno pole, ne jeden řetězec)
```

Pro funkce určené do pipeline je proto lepší nechat výchozí chování a `@()` použít u volajícího.

### Nechtěný výstup ve funkci

Protože funkce vrací všechno, co se uvnitř vypíše, dostane se do výsledku i to, co jste vracet nechtěli. Typicky návratová hodnota metody:

```powershell
function Get-Polozky {
    $seznam = [System.Collections.ArrayList]::new()
    $seznam.Add('první')     # Add vrací index nového prvku, a ten se dostane do výstupu
    $seznam.Add('druhý')
    $seznam
}

(Get-Polozky) -join ', '   # 0, 1, první, druhý
```

Nechtěný výstup potlačte přiřazením do `$null`:

```powershell
function Get-Polozky {
    $seznam = [System.Collections.ArrayList]::new()
    $null = $seznam.Add('první')
    $null = $seznam.Add('druhý')
    $seznam
}

(Get-Polozky) -join ', '   # první, druhý
```

### Předání pole funkci

Argumenty funkce se oddělují mezerou. Čárka vytvoří pole, které se předá jako jeden argument. Zápis se závorkami, obvyklý v jiných jazycích, dělá totéž:

```powershell
function Show-Argumenty {
    param($A, $B)
    "A=[$A] B=[$B]"
}

Show-Argumenty 1 2      # A=[1] B=[2]
Show-Argumenty 1, 2     # A=[1 2] B=[]
Show-Argumenty(1, 2)    # A=[1 2] B=[]
```

Parametr, který má přijmout více hodnot, deklarujte jako pole. Volající pak může předat jednu hodnotu i několik:

```powershell
function Get-Delky {
    param([string[]]$Texty)
    foreach ($text in $Texty) { "${text}: $($text.Length)" }
}

Get-Delky -Texty 'jedna'
Get-Delky -Texty 'dvě', 'tři'
```

## 10. Pole objektů

V praxi pole většinou nedrží čísla nebo texty, ale objekty: soubory, procesy, řádky z CSV. Vlastní záznamy se vytvářejí přes `[pscustomobject]`:

```powershell
$lide = @(
    [pscustomobject]@{ Jmeno = 'Jan';   Oddeleni = 'IT';      Plat = 52000 }
    [pscustomobject]@{ Jmeno = 'Eva';   Oddeleni = 'IT';      Plat = 61000 }
    [pscustomobject]@{ Jmeno = 'Karel'; Oddeleni = 'Obchod';  Plat = 48000 }
    [pscustomobject]@{ Jmeno = 'Jana';  Oddeleni = 'Obchod';  Plat = 55000 }
    [pscustomobject]@{ Jmeno = 'Petr';  Oddeleni = 'Finance'; Plat = 58000 }
)
```

### Jedna vlastnost ze všech prvků

```powershell
$lide.Jmeno -join ', '      # Jan, Eva, Karel, Jana, Petr
$lide[0].Jmeno              # Jan
$lide[-1].Oddeleni          # Finance
```

`$lide.Jmeno` je zkratka za `$lide | ForEach-Object { $_.Jmeno }` nebo `$lide | Select-Object -ExpandProperty Jmeno`.

### Filtrování, řazení, výběr sloupců

```powershell
$lide |
    Where-Object { $_.Plat -gt 50000 } |
    Sort-Object Plat -Descending |
    Select-Object Jmeno, Plat |
    Format-Table
```

```text
Jmeno  Plat
-----  ----
Eva   61000
Petr  58000
Jana  55000
Jan   52000
```

Pro jednoduché podmínky existuje zkrácený zápis bez složených závorek:

```powershell
($lide | Where-Object Oddeleni -eq 'IT').Jmeno -join ', '   # Jan, Eva
```

### Souhrny

```powershell
($lide | Measure-Object Plat -Sum).Sum           # 274000
($lide | Measure-Object Plat -Average).Average   # 54800

($lide | Sort-Object Plat -Descending | Select-Object -First 1).Jmeno   # Eva
```

Souhrn po skupinách:

```powershell
$lide | Group-Object Oddeleni | ForEach-Object {
    [pscustomobject]@{
        Oddeleni = $_.Name
        Pocet    = $_.Count
        Prumer   = ($_.Group | Measure-Object Plat -Average).Average
    }
} | Sort-Object Oddeleni | Format-Table
```

```text
Oddeleni Pocet   Prumer
-------- -----   ------
Finance      1 58000.00
IT           2 56500.00
Obchod       2 51500.00
```

### Změna objektů v poli

Objekty jsou odkazové typy. Proměnná ve `foreach` je sice kopie, ale kopie odkazu na tentýž objekt, takže změna vlastnosti se v poli projeví. Rozdíl proti číslům z kapitoly 5:

```powershell
foreach ($clovek in $lide) {
    if ($clovek.Oddeleni -eq 'Obchod') { $clovek.Plat += 2000 }
}
($lide | Where-Object Oddeleni -eq 'Obchod').Plat -join ', '   # 50000, 57000
```

### Přidání záznamu a hledání

```powershell
$lide += [pscustomobject]@{ Jmeno = 'Ota'; Oddeleni = 'IT'; Plat = 49000 }
$lide.Count   # 6

$nalezeny = $lide.Where({ $_.Jmeno -eq 'Petr' }, 'First')[0]
$nalezeny.Oddeleni   # Finance
```

Pokud podle jména nebo ID hledáte opakovaně, sestavte si z pole hashtable jako index. Postup je v průvodci hashtables v kapitole o vyhledávacích tabulkách.

## 11. Vnořená a vícerozměrná pole

### Pole polí

Prvkem pole může být další pole. Vnitřní pole mohou mít různou délku:

```powershell
$mrizka = @(
    @(1, 2, 3),
    @(4, 5, 6),
    @(7, 8, 9)
)

$mrizka.Count        # 3
$mrizka[1] -join ' ' # 4 5 6
$mrizka[1][2]        # 6
$mrizka[-1][0]       # 7
```

V tomto zápisu jsou čárky mezi řádky nutné. Bez nich by `@( )` vnitřní pole rozbalil do jednoho plochého pole s devíti prvky.

Procházení:

```powershell
foreach ($radek in $mrizka) {
    $radek -join ' '
}
```

### Past: jedno vnořené pole

Stejné rozbalení postihne i vnější pole s jediným vnitřním polem:

```powershell
$spatne = @(@(1, 2))
$spatne.Count    # 2   (ploché pole 1, 2)

$spravne = @(, @(1, 2))
$spravne.Count   # 1   (pole obsahující jedno pole)
$spravne[0] -join ' '   # 1 2
```

Unární čárka před vnitřním polem ho ochrání. Stejně je potřeba postupovat při přidávání pole jako jednoho prvku:

```powershell
$seznamDvojic = @()
$seznamDvojic += , @('a', 1)
$seznamDvojic += , @('b', 2)
$seznamDvojic.Count    # 2
$seznamDvojic[1][0]    # b
```

Bez čárky by `+=` obě pole spojil a vznikly by čtyři samostatné prvky.

### Zploštění

```powershell
$plocha = $mrizka | ForEach-Object { $_ }
$plocha -join ' '   # 1 2 3 4 5 6 7 8 9
```

### Skutečná vícerozměrná pole

.NET zná i pravá vícerozměrná pole s pevným počtem řádků a sloupců. Indexy se píší do jedněch závorek oddělené čárkou:

```powershell
$matice = [int[,]]::new(2, 3)   # 2 řádky, 3 sloupce, samé nuly

$matice[0, 0] = 1
$matice[1, 2] = 9

$matice[1, 2]          # 9
$matice.Rank           # 2   (počet rozměrů)
$matice.GetLength(0)   # 2   (řádků)
$matice.GetLength(1)   # 3   (sloupců)
$matice.Length         # 6   (prvků celkem)
```

V běžném skriptování je potkáte málokdy, hlavně při volání .NET knihoven. Pro tabulková data je v PowerShellu přirozenější pole objektů z kapitoly 10.

## 12. Typovaná pole

Běžné pole přijme cokoli. Typované pole dovolí jen jeden typ a hodnoty při vložení převádí:

```powershell
[int[]]$kody = 1, 2, '3'      # text '3' se převede na číslo
$kody.GetType().Name          # Int32[]
$kody[2] + 1                  # 4

try {
    $kody[0] = 'abc'
} catch {
    'Chyba: hodnotu nelze převést na int'
}
```

### Typ na proměnné a typ na hodnotě

Záleží na tom, kde typ stojí. Typ před názvem proměnné platí pro všechna další přiřazení, takže pole zůstane typované i po `+=`:

```powershell
[int[]]$pevne = 1, 2
$pevne += 3
$pevne.GetType().Name   # Int32[]
```

Typ jen u hodnoty platí pro tu jednu hodnotu. `+=` pak vytvoří obyčejné pole objektů:

```powershell
$volne = [int[]](1, 2)
$volne += 3
$volne.GetType().Name   # Object[]
```

### Parametry funkcí

Nejčastější místo, kde typované pole potkáte, je deklarace parametru:

```powershell
function Get-Soucet {
    param([int[]]$Cisla)
    ($Cisla | Measure-Object -Sum).Sum
}

Get-Soucet -Cisla 1, 2, 3     # 6
Get-Soucet -Cisla '10', 20    # 30
Get-Soucet -Cisla 5           # 5   (jedna hodnota se sama zabalí do pole)
```

### Převod celého pole

```powershell
$texty = '10', '20', '30'
$jakoCisla = [int[]]$texty
($jakoCisla | Measure-Object -Sum).Sum   # 60

$jakoTexty = [string[]](1, 2, 3)
$jakoTexty[0].GetType().Name             # String
```

### Pole bajtů a znaků

Binární data jsou v .NET pole bajtů, text lze rozložit na pole znaků:

```powershell
$bajty = [System.Text.Encoding]::UTF8.GetBytes('Ahoj')
$bajty.GetType().Name    # Byte[]
$bajty -join ' '         # 65 104 111 106

$znaky = 'Praha'.ToCharArray()
$znaky.Count             # 5
[array]::Reverse($znaky)
-join $znaky             # aharP
```

Řetězec lze indexovat stejně jako pole, i když polem není:

```powershell
'Praha'[0]              # P
'Praha'[-1]             # a
-join 'Praha'[0..2]     # Pra
```

## 13. Kopírování a porovnávání

### Přiřazení nekopíruje

Pole je odkazový typ. Přiřazením do druhé proměnné vznikne druhý odkaz na totéž pole:

```powershell
$original = 1, 2, 3
$druhe    = $original
$druhe[0] = 99

$original -join ' '   # 99 2 3
```

Tady je důležitý rozdíl mezi změnou prvku a operátorem `+=`. Změna prvku se projeví v obou proměnných. `+=` vytvoří nové pole, a tím se proměnné rozdělí:

```powershell
$druhe += 4

$original -join ' '   # 99 2 3
$druhe -join ' '      # 99 2 3 4
```

Stejně je to u funkcí. Funkce může měnit prvky pole, které dostala, ale `+=` uvnitř funkce na původní pole nedosáhne:

```powershell
function Set-PrvniPrvek {
    param($Pole)
    $Pole[0] = 'změněno'
    $Pole += 'přidáno'      # mění jen místní proměnnou
}

$vstup = 'a', 'b'
Set-PrvniPrvek -Pole $vstup
$vstup -join ', '   # změněno, b
```

### Kopie: `Clone()`

```powershell
$original = 1, 2, 3
$kopie = $original.Clone()
$kopie[0] = 99

$original -join ' '   # 1 2 3
$kopie -join ' '      # 99 2 3
```

Kopie je mělká. Pokud pole obsahuje objekty, hashtables nebo další pole, kopie i originál ukazují na tytéž vnitřní objekty:

```powershell
$puvodni = @(
    [pscustomobject]@{ Jmeno = 'Jan' }
    [pscustomobject]@{ Jmeno = 'Eva' }
)
$melka = $puvodni.Clone()
$melka[0].Jmeno = 'ZMĚNA'

$puvodni[0].Jmeno   # ZMĚNA
```

### Porovnání obsahu

Operátor `-eq` mezi dvěma poli neporovnává obsah. Podle kapitoly 6 filtruje, a výsledek proto nedává smysl:

```powershell
$p1 = 1, 2, 3
$p2 = 1, 2, 3

($p1 -eq $p2).Count   # 0   (žádný prvek z $p1 se nerovná poli $p2)
```

Porovnání včetně pořadí:

```powershell
[System.Linq.Enumerable]::SequenceEqual([object[]]$p1, [object[]]$p2)          # True
[System.Linq.Enumerable]::SequenceEqual([object[]]$p1, [object[]](3, 2, 1))    # False
```

Porovnání bez ohledu na pořadí. `Compare-Object` nevrátí nic, pokud se pole neliší:

```powershell
$null -eq (Compare-Object $p1 (3, 2, 1))   # True   (stejné prvky)
$null -eq (Compare-Object $p1 (1, 2, 4))   # False  (liší se)
```

## 14. Výkon: proč je `+=` pomalé

Každé `+=` vytvoří nové pole a zkopíruje do něj všechny dosavadní prvky. Přidání tisícího prvku znamená zkopírovat 999 předchozích, desetitisícího 9 999. Celková práce tak roste s druhou mocninou počtu prvků. U stovek položek si toho nevšimnete, u desítek tisíc skript viditelně zpomalí.

```powershell
$pocet = 20000

# 1. += ve smyčce
$casPlus = Measure-Command {
    $vysledek = @()
    foreach ($n in 1..$pocet) { $vysledek += $n * 2 }
}

# 2. sběr výstupu smyčky
$casSber = Measure-Command {
    $vysledek = foreach ($n in 1..$pocet) { $n * 2 }
}

# 3. List[T]
$casList = Measure-Command {
    $seznam = [System.Collections.Generic.List[int]]::new()
    foreach ($n in 1..$pocet) { $seznam.Add($n * 2) }
}

'+=: {0:N0} ms, sběr výstupu: {1:N0} ms, List: {2:N0} ms' -f $casPlus.TotalMilliseconds, $casSber.TotalMilliseconds, $casList.TotalMilliseconds
```

Při ověřování tohoto příkladu (PowerShell 7.4, 20 000 prvků) trvala varianta s `+=` zhruba 7 sekund, sběr výstupu kolem 15 milisekund a `List` kolem 50 milisekund. Konkrétní čísla závisí na počítači, řádový rozdíl ne. PowerShell 7.5 operátor `+=` výrazně zrychlil, princip kopírování ale zůstává a ve Windows PowerShellu 5.1 je rozdíl plný.

Co z toho plyne:

- **Stavíte pole ve smyčce:** přiřaďte výstup smyčky (`$vysledek = foreach ...`). Je to nejkratší i nejrychlejší.
- **Přidáváte na více místech, podmíněně nebo průběžně odebíráte:** použijte `List[T]`.
- **Přidáváte pár prvků:** `+=` je v pořádku a čitelné.

Podobně je na tom opakované hledání přes `-contains` nebo `-in` ve velkém poli, které pokaždé prochází pole od začátku. Pro opakované testy použijte `HashSet` nebo hashtable.

## 15. Příbuzné typy: List, HashSet, Queue, Stack

### `List[T]`: pole, které umí růst

`List` je nejčastější náhrada pole. Indexuje se stejně, ale přidávání a odebírání je rychlé a mění seznam na místě:

```powershell
$seznam = [System.Collections.Generic.List[string]]::new()

$seznam.Add('první')
$seznam.Add('druhý')
$seznam.AddRange([string[]]('třetí', 'čtvrtý'))
$seznam.Insert(1, 'vložený')

$seznam -join ', '    # první, vložený, druhý, třetí, čtvrtý
$seznam[0]            # první
$seznam[-1]           # čtvrtý
$seznam.Count         # 5
```

Odebírání:

```powershell
$null = $seznam.Remove('druhý')    # podle hodnoty; vrací True/False, proto $null =
$seznam.RemoveAt(0)                # podle pozice
$null = $seznam.RemoveAll({ param($s) $s -like 'č*' })   # podle podmínky; vrací počet odebraných

$seznam -join ', '    # vložený, třetí
```

Vytvoření z existujícího pole a převod zpět:

```powershell
$cisla = [System.Collections.Generic.List[int]](5, 3, 8)
$cisla.Add(1)
$cisla.Sort()
$cisla -join ' '               # 1 3 5 8
$cisla.Contains(8)             # True

$zpetPole = $cisla.ToArray()
$zpetPole.GetType().Name       # Int32[]
```

`List` funguje v pipeline, ve `foreach` i s operátory `-join` a `-contains` stejně jako pole. Pro prvky různých typů použijte `List[object]`.

Ve starších skriptech uvidíte `System.Collections.ArrayList`. Dělá totéž bez typové kontroly a jeho metoda `Add` vrací index, který znečišťuje výstup (viz kapitola 9). V novém kódu dejte přednost `List[T]`.

### `HashSet[T]`: sada unikátních hodnot

`HashSet` drží každou hodnotu nejvýš jednou a umí velmi rychle odpovědět, jestli v něm hodnota je. Pořadí prvků negarantuje.

```powershell
$videno = [System.Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase)

$videno.Add('jablko')    # True   (přidáno)
$videno.Add('hruška')    # True
$videno.Add('JABLKO')    # False  (už tam je)

$videno.Count                # 2
$videno.Contains('Hruška')   # True
```

Návratová hodnota `Add` se hodí při odstraňování duplicit, protože rovnou říká, jestli jde o první výskyt:

```powershell
$vstup = 'b', 'a', 'B', 'c', 'a'
$uz = [System.Collections.Generic.HashSet[string]]::new([StringComparer]::OrdinalIgnoreCase)
$unikatni = foreach ($prvek in $vstup) {
    if ($uz.Add($prvek)) { $prvek }
}
$unikatni -join ' '   # b a c
```

Množinové operace mění sadu na místě:

```powershell
$sadaA = [System.Collections.Generic.HashSet[int]](1, 2, 3, 4)
$sadaB = [System.Collections.Generic.HashSet[int]](3, 4, 5)

$prunik = [System.Collections.Generic.HashSet[int]]::new($sadaA)
$prunik.IntersectWith($sadaB)
($prunik | Sort-Object) -join ' '     # 3 4

$rozdil = [System.Collections.Generic.HashSet[int]]::new($sadaA)
$rozdil.ExceptWith($sadaB)
($rozdil | Sort-Object) -join ' '     # 1 2

$sjednoceni = [System.Collections.Generic.HashSet[int]]::new($sadaA)
$sjednoceni.UnionWith($sadaB)
($sjednoceni | Sort-Object) -join ' ' # 1 2 3 4 5
```

### `Queue[T]` a `Stack[T]`

Fronta vydává prvky v pořadí, v jakém přišly. Zásobník vydává naposledy vložený prvek jako první:

```powershell
$fronta = [System.Collections.Generic.Queue[string]]::new()
$fronta.Enqueue('úkol 1')
$fronta.Enqueue('úkol 2')
$fronta.Enqueue('úkol 3')

$fronta.Dequeue()   # úkol 1
$fronta.Peek()      # úkol 2   (podívá se, ale neodebere)
$fronta.Count       # 2
```

```powershell
$zasobnik = [System.Collections.Generic.Stack[string]]::new()
$zasobnik.Push('první')
$zasobnik.Push('druhý')
$zasobnik.Push('třetí')

$zasobnik.Pop()     # třetí
$zasobnik.Pop()     # druhý
$zasobnik.Count     # 1
```

Fronta se hodí pro zpracování úkolů, které během práce přibývají (například procházení složek nebo odkazů do šířky), zásobník pro návrat o krok zpět a procházení do hloubky.

### Přehled

| Typ | Pořadí | Duplicity | Přidání a odebrání | Test „obsahuje" | Přístup přes index |
|---|---|---|---|---|---|
| pole `@()` | ano | ano | vytváří nové pole | pomalý | ano |
| `List[T]` | ano | ano | rychlé | pomalý | ano |
| `HashSet[T]` | ne | ne | rychlé | rychlý | ne |
| `Queue[T]` | podle příchodu | ano | jen na koncích | pomalý | ne |
| `Stack[T]` | obrácené | ano | jen na vrcholu | pomalý | ne |
| hashtable `@{}` | ne | klíče ne | rychlé | rychlý (klíče) | přes klíč |

## 16. Praktické vzory

### Rozdělení na dávky

Častá potřeba při volání API s limitem na počet položek nebo při hromadném zpracování:

```powershell
$polozky = 1..11
$velikostDavky = 4

$davky = for ($i = 0; $i -lt $polozky.Count; $i += $velikostDavky) {
    $konec = [math]::Min($i + $velikostDavky, $polozky.Count) - 1
    , $polozky[$i..$konec]
}

$davky.Count                       # 3
$davky[0] -join ' '                # 1 2 3 4
$davky[-1] -join ' '               # 9 10 11
```

Unární čárka zajistí, že každá dávka zůstane samostatným polem a výstup smyčky se nespojí do jednoho plochého pole.

### Souběžný průchod dvěma poli

```powershell
$nazvy  = 'CPU', 'RAM', 'Disk'
$vyuziti = 35, 72, 91

for ($i = 0; $i -lt $nazvy.Count; $i++) {
    '{0,-5} {1,3} %' -f $nazvy[$i], $vyuziti[$i]
}
```

Pokud dvě pole patří k sobě, je lepší je spojit do pole objektů a dál pracovat s jedním:

```powershell
$mereni = for ($i = 0; $i -lt $nazvy.Count; $i++) {
    [pscustomobject]@{ Nazev = $nazvy[$i]; Vyuziti = $vyuziti[$i] }
}

($mereni | Where-Object Vyuziti -gt 70).Nazev -join ', '   # RAM, Disk
```

### Zpracování textu po řádcích

```powershell
$log = @'
2026-10-08 09:15 INFO  Start
2026-10-08 09:16 ERROR Spojení selhalo
2026-10-08 09:17 INFO  Opakuji
2026-10-08 09:18 ERROR Časový limit
'@ -split '\r?\n'

$chyby = $log | Where-Object { $_ -match 'ERROR' }
$chyby.Count   # 2

$chyby | ForEach-Object {
    $datum, $cas, $uroven, $zprava = $_ -split '\s+', 4
    "$cas -> $zprava"
}
```

Přiřazení do více proměnných spolu s omezeným `-split` (nejvýše 4 části) rozloží řádek na pole a zprávu s mezerami ponechá vcelku.

### Kontrola vstupu proti seznamu

```powershell
function Test-Pripona {
    param([string]$Soubor)
    $povolene = '.csv', '.json', '.xml'
    [IO.Path]::GetExtension($Soubor) -in $povolene
}

Test-Pripona 'data.CSV'     # True
Test-Pripona 'skript.exe'   # False
```

U parametrů funkcí totéž zajistí atribut `[ValidateSet('csv', 'json', 'xml')]`, který navíc nabízí hodnoty při doplňování tabulátorem.

### Bezpečný první a poslední prvek

```powershell
$mozna = @()

$prvni = $mozna | Select-Object -First 1
$null -eq $prvni   # True   (bez chyby i v přísném režimu)
```

`Select-Object -First 1` navíc zastaví pipeline hned po prvním výsledku, takže předchozí příkazy nemusí doběhnout celé.

### Posun a rotace

```powershell
$rada = 1..5

($rada[1..($rada.Count - 1)] + $rada[0]) -join ' '    # 2 3 4 5 1   (rotace vlevo)
```

U rotace vpravo nestačí operandy prohodit. O tom, co operátor `+` udělá, rozhoduje levý operand. Pokud je vlevo číslo, PowerShell se pokusí o číselný součet a s polem vpravo skončí chybou:

```powershell
try {
    $rada[-1] + $rada[0..($rada.Count - 2)]
} catch {
    'Chyba: číslo a pole nelze sečíst'
}
```

Aby šlo o spojení polí, musí být levý operand pole:

```powershell
(@($rada[-1]) + $rada[0..($rada.Count - 2)]) -join ' '   # 5 1 2 3 4   (rotace vpravo)
```

### Stránkování

```powershell
$vsechno = 1..23
$naStranku = 10
$strana = 3

$zacatek = ($strana - 1) * $naStranku
($vsechno | Select-Object -Skip $zacatek -First $naStranku) -join ' '   # 21 22 23

[math]::Ceiling($vsechno.Count / $naStranku)   # 3   (počet stran)
```

### Pole jako poziční argumenty

Splatting znáte z hashtables pro pojmenované parametry. Pole se dá rozbalit stejně, jen jako poziční argumenty:

```powershell
$argumenty = 'jedna', 'dva'
Show-Argumenty @argumenty   # A=[jedna] B=[dva]
```

Často se to používá pro argumenty externích programů, například `& git @gitArgumenty`.

## 17. Pole a soubory: text, CSV, JSON

Pracovní složka pro příklady:

```powershell
$slozka = Join-Path ([IO.Path]::GetTempPath()) 'pole-demo'
New-Item -ItemType Directory -Path $slozka -Force | Out-Null
```

### Textové soubory

`Set-Content` zapíše každý prvek pole jako jeden řádek. `Get-Content` vrátí pole řádků:

```powershell
$souborTxt = Join-Path $slozka 'mesta.txt'
'Praha', 'Brno', 'Ostrava', 'Plzeň' | Set-Content -Path $souborTxt -Encoding utf8

$radky = Get-Content -Path $souborTxt
$radky.Count    # 4
$radky[1]       # Brno
$radky[-1]      # Plzeň
```

Užitečné přepínače:

```powershell
(Get-Content -Path $souborTxt -TotalCount 2) -join ', '   # Praha, Brno      (první dva řádky)
(Get-Content -Path $souborTxt -Tail 1)                    # Plzeň            (poslední řádek)
(Get-Content -Path $souborTxt -Raw).GetType().Name        # String           (celý soubor jako jeden text)
```

Rozbalování z kapitoly 9 platí i tady. Soubor s jedním řádkem vrátí řetězec, ne pole, a `[0]` pak dá první znak:

```powershell
$jedenRadek = Join-Path $slozka 'jeden.txt'
'pouze jeden řádek' | Set-Content -Path $jedenRadek -Encoding utf8

(Get-Content -Path $jedenRadek)[0]       # p
@(Get-Content -Path $jedenRadek)[0]      # pouze jeden řádek
```

Přidání řádků na konec:

```powershell
'Liberec', 'Olomouc' | Add-Content -Path $souborTxt -Encoding utf8
@(Get-Content -Path $souborTxt).Count   # 6
```

### CSV

`Import-Csv` vrací pole objektů, kde sloupce jsou vlastnosti. Všechny hodnoty jsou text:

```powershell
$souborCsv = Join-Path $slozka 'lide.csv'
$lide | Export-Csv -Path $souborCsv -NoTypeInformation -Encoding utf8

$zCsv = @(Import-Csv -Path $souborCsv)
$zCsv.Count                       # 6
$zCsv[0].Jmeno                    # Jan
$zCsv[0].Plat.GetType().Name      # String

($zCsv | Sort-Object { [int]$_.Plat } -Descending | Select-Object -First 1).Jmeno   # Eva
```

### JSON

Pole se na JSON převede jako seznam v hranatých závorkách:

```powershell
ConvertTo-Json -InputObject @(1, 2, 3) -Compress     # [1,2,3]
```

Pozor na předání přes pipeline. Pipeline pole rozbalí, takže jednoprvkové pole dorazí jako jedna hodnota a hranaté závorky z výstupu zmizí. Příjemce JSON, který čeká seznam, pak dostane něco jiného:

```powershell
@('jediný') | ConvertTo-Json -Compress                 # "jediný"
ConvertTo-Json -InputObject @('jediný') -Compress      # ["jediný"]
@('jediný') | ConvertTo-Json -Compress -AsArray        # ["jediný"]   (PowerShell 7)
```

U pole uvnitř objektu tento problém není. Vnořené pole zůstává polem i s jedním prvkem:

```powershell
@{ Servery = @('web01') } | ConvertTo-Json -Compress   # {"Servery":["web01"]}
```

Opačný směr. PowerShell 7 při načítání pole z JSON prvky rozbalí do pipeline, takže jednoprvkový seznam skončí jako jedna hodnota. Windows PowerShell 5.1 se chová opačně a pole pošle jako jeden objekt. V obou verzích dostanete pole spolehlivě přes `@( )`:

```powershell
$seznam = '[10, 20, 30]' | ConvertFrom-Json
$seznam.Count   # 3

$jedenPrvek = @('[10]' | ConvertFrom-Json)
$jedenPrvek.Count   # 1
```

Pole objektů do souboru a zpět:

```powershell
$souborJson = Join-Path $slozka 'lide.json'
ConvertTo-Json -InputObject $lide -Depth 5 | Set-Content -Path $souborJson -Encoding utf8

$zJson = @(Get-Content -Path $souborJson -Raw | ConvertFrom-Json)
$zJson.Count                    # 6
$zJson[1].Jmeno                 # Eva
$zJson[1].Plat.GetType().Name   # Int64   (JSON na rozdíl od CSV zachovává čísla)
```

## 18. Nejčastější chyby

| Chyba | Co se stane | Správně |
|---|---|---|
| `$v = Příkaz` a pak `$v[0]`, `$v.Count` | při jednom výsledku není `$v` pole | `$v = @(Příkaz)` |
| `$pole += $x` ve velké smyčce | každé přidání kopíruje celé pole | `$pole = foreach ...` nebo `List[T]` |
| `$pole.Add($x)` | chyba, pole má pevnou velikost | `+=` nebo `List[T]` |
| `if ($pole -eq $null)` | filtruje prvky místo testu proměnné | `if ($null -eq $pole)` |
| `if ($pole)` jako test neprázdnosti | `@(0)` a `@($false)` jsou nepravdivé | `$pole.Count -gt 0` |
| `$pole[2..-1]` pro „od indexu 2 do konce" | rozsah jde přes nulu dozadu | `$pole[2..($pole.Count - 1)]` |
| `0..($pole.Count - 1)` u prázdného pole | rozsah `0..-1` má dva prvky | `for` s podmínkou nebo `foreach` |
| `$a -eq $b` pro porovnání dvou polí | filtruje, obsah neporovná | `Compare-Object` nebo `SequenceEqual` |
| `$kopie = $pole` | obě proměnné sdílejí jedno pole | `$pole.Clone()` |
| `foreach ($x in $pole) { $x = ... }` | pole se nezmění | `for` s indexem nebo nové pole |
| `@(@(1, 2))` | vnitřní pole se rozbalí | `@(, @(1, 2))` |
| `$pole += @(1, 2)` pro přidání pole jako prvku | přidají se dva prvky | `$pole += , @(1, 2)` |
| `Funkce(1, 2)` | předá jedno pole prvnímu parametru | `Funkce 1 2` |
| `$pole -contains 'část'` | hledá celý prvek, ne část textu | `$pole -like '*část*'` |
| `$texty.Length` pro délky řetězců | vrátí počet prvků pole | `$texty.ForEach('Length')` |
| `"Počet: $pole.Count"` | vypíše pole a text `.Count` | `"Počet: $($pole.Count)"` |
| `Sort-Object` nad čísly v textu | řadí abecedně (`10` před `9`) | `Sort-Object { [int]$_ }` |
| metoda vracející hodnotu uvnitř funkce | hodnota se přimíchá do výstupu | `$null = $seznam.Add(...)` |
| `@('x') \| ConvertTo-Json` | výsledkem není JSON seznam | `ConvertTo-Json -InputObject @('x')` |
| `$hodnota + $pole` | o operaci rozhoduje levý operand: u čísla chyba, u textu spojený řetězec | `@($hodnota) + $pole` |

## 19. Tahák

| Operace | Zápis |
|---|---|
| Vytvoření | `$p = 1, 2, 3` nebo `$p = @(1, 2, 3)` |
| Prázdné pole | `$p = @()` |
| Jednoprvkové pole | `$p = @('x')` nebo `$p = , 'x'` |
| Rozsah | `1..10` |
| Zaručené pole z výsledku příkazu | `$p = @(Příkaz)` |
| Typované pole | `[int[]]$p = 1, 2, 3` |
| Počet prvků | `$p.Count` |
| První, poslední | `$p[0]`, `$p[-1]` |
| Řez | `$p[1..3]`, `$p[-3..-1]` |
| První a poslední N | `$p \| Select-Object -First 3`, `-Last 3`, `-Skip 3` |
| Změna prvku | `$p[0] = 'x'` |
| Přidání (vytvoří nové pole) | `$p += 'x'` |
| Spojení polí | `$a + $b` |
| Odebrání podle hodnoty | `$p = @($p \| Where-Object { $_ -ne 'x' })` |
| Obsahuje hodnotu | `$p -contains 'x'`, `'x' -in $p` |
| Pozice hodnoty | `$p.IndexOf('x')` |
| Filtr | `$p \| Where-Object { ... }`, `$p.Where({ ... })` |
| První vyhovující | `$p.Where({ ... }, 'First')` |
| Transformace | `$p \| ForEach-Object { ... }`, `$p.ForEach({ ... })` |
| Nové pole ze smyčky | `$n = foreach ($x in $p) { ... }` |
| Řazení | `$p \| Sort-Object`, `-Descending`, `-Property` |
| Unikátní hodnoty | `$p \| Select-Object -Unique`, `$p \| Sort-Object -Unique` |
| Obrácení na místě | `[array]::Reverse($p)` |
| Součet, průměr, min, max | `$p \| Measure-Object -Sum -Average -Minimum -Maximum` |
| Pole na text | `$p -join ', '` |
| Text na pole | `$text -split ','` |
| Rozdělení do proměnných | `$prvni, $zbytek = $p` |
| Kopie | `$p.Clone()` |
| Porovnání dvou polí | `Compare-Object $a $b` |
| Vrácení pole jako celku z funkce | `, $p` nebo `Write-Output -NoEnumerate $p` |
| Seznam s rychlým přidáváním | `[System.Collections.Generic.List[string]]::new()` |
| Sada unikátních hodnot | `[System.Collections.Generic.HashSet[string]]::new()` |
| Řádky souboru jako pole | `@(Get-Content -Path soubor.txt)` |
| Pole jako JSON seznam | `ConvertTo-Json -InputObject $p` |
