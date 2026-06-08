 # Příklady práce se slovníky (hash tables) v PowerShellu:

1. **Vytvoření slovníku:**
   ```powershell
   $mojeSlovnik = @{
       "jablko" = "ovoce"
       "pes" = "zvíře"
       "modrá" = "barva"
   }
   ```

2. **Přístup k hodnotám:**
   ```powershell
   $mojeSlovnik["jablko"]  # Výstup: ovoce
   ```

3. **Přidání nového páru klíč-hodnota:**
   ```powershell
   $mojeSlovnik["strom"] = "rostlina"
   ```

4. **Změna hodnoty:**
   ```powershell
   $mojeSlovnik["pes"] = "čtyřnohé zvíře"
   ```

5. **Odstranění položky:**
   ```powershell
   $mojeSlovnik.Remove("modrá")
   ```

6. **Získání všech klíčů:**
   ```powershell
   $mojeSlovnik.Keys
   ```

7. **Získání všech hodnot:**
   ```powershell
   $mojeSlovnik.Values
   ```

8. **Iterace přes slovník:**
   ```powershell
   foreach ($klic in $mojeSlovnik.Keys) {
       Write-Output "$klic: $($mojeSlovnik[$klic])"
   }
   ```

9. **Kontrola existence klíče:**
   ```powershell
   if ($mojeSlovnik.ContainsKey("jablko")) {
       Write-Output "Jablko je v slovníku"
   }
   ```

10. **Složitější příklad - použití slovníku jako parametru funkce:**
    ```powershell
    function VypisSlovnik {
        param (
            [hashtable]$slovnik
        )
        foreach ($klic in $slovnik.Keys) {
            Write-Host "$klic: $($slovnik[$klic])"
        }
    }

    $testSlovnik = @{
        "jmeno" = "Jan"
        "vek" = 30
        "mesto" = "Praha"
    }

    VypisSlovnik -slovnik $testSlovnik
    ```

Tyto příklady ukazují základní operace se slovníky v PowerShellu. Slovníky jsou velmi užitečné pro ukládání a manipulaci s daty, kde klíčem může být jakýkoli objekt (včetně dalších slovníků nebo pole).

