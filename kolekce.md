**Kolekce** je obecný pojem pro jakýkoli objekt, který drží víc hodnot a dá se procházet. Pole, hashtable i seznam jsou konkrétní druhy kolekcí a liší se tím, jak se k hodnotám dostanete a jestli se dají měnit.

| | Pole `@()` | Seznam `List[T]` | Hashtable `@{}` |
|---|---|---|---|
| Přístup k hodnotě | přes pozici `$p[0]` | přes pozici `$s[0]` | přes klíč `$h['jmeno']` |
| Pořadí | zaručené | zaručené | nezaručené (kromě `[ordered]`) |
| Velikost | pevná | proměnná | proměnná |
| Přidání prvku | `+=` vytvoří nové pole, pomalé | `.Add()`, rychlé | `$h[$k] = $v`, rychlé |
| Odebrání prvku | jen filtrem do nového pole | `.Remove()`, `.RemoveAt()` | `.Remove($k)` |
| Hledání hodnoty | projde vše, pomalé | projde vše, pomalé | podle klíče okamžité |
| Duplicity | ano | ano | klíče ne, hodnoty ano |

- **Pole** je výchozí volba PowerShellu: vznikne samo, když příkaz vrátí víc výsledků. Hodí se pro data, která po vytvoření už moc neměníte.
- **Seznam** se chová jako pole, ale umí růst a zmenšovat se. Použijte ho, když ve smyčce průběžně přidáváte nebo odebíráte.
- **Hashtable** není řada, ale slovník dvojic klíč → hodnota. Použijte ji, když hledáte podle názvu nebo ID, nebo když předáváte sadu pojmenovaných hodnot (konfigurace, splatting).

```powershell
$pole    = 'Jan', 'Eva'                                       # pevná řada
$seznam  = [System.Collections.Generic.List[string]]::new()   # řada, která roste
$seznam.Add('Jan')
$slovnik = @{ Jan = 30; Eva = 28 }                            # hledání podle klíče

$pole[0]          # Jan
$seznam[0]        # Jan
$slovnik['Jan']   # 30
```

V pipeline se pole a seznam rozpadnou na jednotlivé prvky, kdežto hashtable projde jako jeden objekt. Pro průchod po položkách proto potřebuje `.GetEnumerator()`.

Mezi kolekce patří i další typy, například `HashSet` (jen unikátní hodnoty), `Queue` (fronta) a `Stack` (zásobník). Podrobnosti jsou v kapitole 15 průvodce poli a v kapitole 17 průvodce hashtables.
