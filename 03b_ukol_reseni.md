# RMAN: Úkol — řešení (komentované)

> Řešení k zadání `03a_ukol_zadani.md`. Symboly: 👥 společně na adminu, 👤 každý zvlášť, 📝 na papíře.

---

## Úkol 1 — Politika uchovávání

### 👥 SPOLEČNĚ

```
rman target /
RMAN> SHOW ALL;                                                 -- výchozí je REDUNDANCY 1
RMAN> CONFIGURE RETENTION POLICY TO REDUNDANCY 2;
RMAN> REPORT OBSOLETE;                                          -- jen vypsat, NEMAZAT
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
RMAN> REPORT OBSOLETE;                                          -- pro srovnání
RMAN> SHOW ALL;                                                 -- ověřit, že je okno obnovy zpět
```

**Komentář:**
- `REDUNDANCY 2` = zachovat **2 zálohy každého souboru**. Zastaralé je to, co přesahuje tento počet, **bez ohledu na stáří**. Proto mohou být za zastaralé označeny i hodinu staré zálohy. Co přesně se zobrazí, závisí na tom, kolik základních záloh (plná a úroveň 0) existuje; výstup si ověřte na vlastní databázi.
- `RECOVERY WINDOW OF 7 DAYS` = zachovat zálohy potřebné k obnově k **libovolnému okamžiku za posledních 7 dní**. Neurčuje počet, ale časový rozsah.
- ⚠️ `DELETE OBSOLETE` při redundanci 2 nespouštějte: smazali byste zálohy, které potřebují navazující témata.

---

## Úkol 2 — Týdenní strategie

### 📝 Doplněná tabulka

| Den | Záloha | Proč |
|---|---|---|
| Neděle | `BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;` | nový základ týdne; jen L0 slouží jako základ pro L1 |
| Pondělí | `BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;` | rozdílová, bloky změněné od neděle; nejmenší a nejrychlejší |
| Úterý | totéž (L1 rozdílová) | bloky změněné od pondělí |
| Středa | `BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE PLUS ARCHIVELOG;` | všechny změny od neděle; zkrátí obnovu ve druhé polovině týdne (stačí L0 + poslední kumulativní) |
| Čtvrtek–sobota | L1 rozdílová | rychlé denní zálohy |

**Kdy se maže:** každý den po záloze `DELETE NOPROMPT OBSOLETE;` (úklid podle okna obnovy 7 dní).

**Kdy se ověřuje:** každý den `REPORT NEED BACKUP;` (mají všechny soubory dost záloh?) a **jednou týdně** `RESTORE DATABASE VALIDATE;` (lze ze záloh skutečně obnovit?).

**RPO / RTO:** přísnější RPO = častější zálohy a archivace logů; přísnější RTO = kratší řetěz záloh k aplikaci (častější L0 nebo kumulativní, případně inkrementálně aktualizované kopie obrazů, které lze použít okamžitě).

### 👥 Skripty

```
mkdir C:\backup\skripty
mkdir C:\backup\logy
```

```
-- C:\backup\skripty\level0.rman
RUN {
  BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG TAG 'tyden_l0';
  DELETE NOPROMPT OBSOLETE;
}
```

```
-- C:\backup\skripty\level1.rman
RUN {
  BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG TAG 'tyden_l1';
  DELETE NOPROMPT OBSOLETE;
}
```

```
rman target / cmdfile=C:\backup\skripty\level0.rman log=C:\backup\logy\level0.log
rman target / cmdfile=C:\backup\skripty\level1.rman log=C:\backup\logy\level1.log
type C:\backup\logy\level1.log
```

**Komentář:** `rman target /` používá OS autentizaci, takže ve skriptu ani příkazu není heslo. Skripty se pak spouštějí plánovačem (Plánovač úloh Windows, cron, nebo `DBMS_SCHEDULER`). `TAG` usnadní pozdější dohledání záloh.

---

## Úkol 3 — Vyzkoušení a ověření

### 👤 KAŽDÝ — změny dat

```sql
-- CV4, ve své PDB
UPDATE zbozi SET cena = cena * 1.1 WHERE MOD(id, 10) = 0;
COMMIT;
```

Změna je nutná, aby měla L1 co zálohovat. Bez ní by byla skoro prázdná.

### 👥 SPOLEČNĚ — zálohy

```
-- 1. základ
rman target / cmdfile=C:\backup\skripty\level0.rman log=C:\backup\logy\level0.log
-- 2. po změně dat obou členů
rman target / cmdfile=C:\backup\skripty\level1.rman log=C:\backup\logy\level1.log
-- 3. kumulativní záchytný bod
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE PLUS ARCHIVELOG TAG 'tyden_l1_kum';
```

### 👥 SPOLEČNĚ — LIST a REPORT

```
RMAN> LIST BACKUP SUMMARY;
RMAN> REPORT NEED BACKUP;
RMAN> REPORT OBSOLETE;
```

| Příkaz | Co z výstupu čtete |
|---|---|
| `LIST BACKUP SUMMARY` | sloupec `LV` = úroveň (0, 1, `A` = archivní logy), `S` = stav (`A` available, `X` expired), `TAG` = vaše značky |
| `REPORT NEED BACKUP` | prázdný výstup = všechny soubory mají dostatečnou zálohu podle politiky |
| `REPORT OBSOLETE` | zálohy, které politika už nepotřebuje; smažou se `DELETE NOPROMPT OBSOLETE` |

> **Odpověď na zamyšlení:** Za týden bez záloh by `REPORT NEED BACKUP` začal vypisovat soubory, protože by okno obnovy 7 dní nebylo pokryté. **L1 kumulativní** obsahuje změny od poslední L0, takže při obnově stačí L0 a poslední kumulativní. **L1 rozdílová** obsahuje změny od poslední zálohy jakékoli úrovně, takže se musí aplikovat všechny rozdílové od L0.

### 👤 KAŽDÝ — kontrola v SQL

```sql
-- SYS_CDB
SELECT session_key, input_type, status, TO_CHAR(start_time, 'HH24:MI:SS') AS start_, time_taken_display,
       input_bytes_display, output_bytes_display, compression_ratio
FROM   v$rman_backup_job_details ORDER BY start_time;
```

| Sloupec | Co říká |
|---|---|
| `input_type` | typ zálohy (např. `DB INCR`, `ARCHIVELOG`) |
| `input_bytes_display` | kolik dat RMAN přečetl |
| `output_bytes_display` | velikost zálohy na disku |
| `time_taken_display` | jak dlouho trvala |
| `compression_ratio` | poměr komprese (zálohy jsou komprimované přes `CONFIGURE`) |

---

## Bonus — řešení

`input_bytes_display` ukazuje, kolik dat RMAN přečetl, `output_bytes_display` kolik zapsal. U **L0** je výstup velký (všechny použité bloky), u **L1** malý (jen změněné bloky). Vstup může být u L1 bez BCT velký, protože RMAN musí projít všechny bloky a porovnat SCN. Se zapnutým BCT čte jen změněné.

---

## Časté potíže

| Potíž | Příčina | Řešení |
|---|---|---|
| Po změně retention policy zmizely zálohy, které jste potřebovali | spuštěno `DELETE OBSOLETE` při redundanci 2 | před mazáním vždy `REPORT OBSOLETE` a ověřit politiku přes `SHOW ALL` |
| Skript nenajde soubor nebo log | neexistují adresáře `C:\backup\skripty`, `C:\backup\logy` | vytvořit je předem |
| `LIST BACKUP OF PLUGGABLE DATABASE` nic nevrací | zálohovala se jen jiná PDB, nebo záloha ještě neběžela | zkontrolovat `LIST BACKUP SUMMARY` |
| Heslo ve skriptu nebo příkazu | použito `rman target sys/heslo@FREE` | používat `rman target /` |
