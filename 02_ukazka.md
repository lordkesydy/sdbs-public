# RMAN: Ukázka — scénář k přečtení (15 min)

> **Cíl:** Živě předvést nastavení RMAN, plnou zálohu, inkrementální zálohu úrovně 1 bez BCT a s BCT a porovnat počet přečtených bloků, velikost a dobu trvání.

**Jak číst tenhle soubor:**
- **🎤** = text k přečtení nahlas (první osoba).
- *(kurzívou v závorce)* = co v tu chvíli udělat, neříkat.
- **📝 Pro sebe** = poznámky, které neříkáš nahlas.
- 👥 = globální krok na adminu (CDB), dělá jeden, ostatní sledují. 👤 = dělá každý ve svém připojení.

> ⚠️ **Zálohy trvají.** Celou ukázku vyzkoušej nanečisto a zdlouhavé zálohy (plná, L0) **předpřiprav**. Živě pouštěj jen to, co trvá desítky sekund (L1).

---

## Před ukázkou (nanečisto, neříkat)

```sql
-- SYS_CDB / sqlplus / as sysdba
ARCHIVE LOG LIST                                           -- musí být Archive Mode
SHOW PARAMETER db_recovery_file_dest                       -- kam půjdou zálohy (FRA)
SELECT status, filename FROM v$block_change_tracking;      -- pokud už je ENABLED, krok 4 přeskoč
```

```sql
-- připojení VOJTECHVOLF (FREEPDB1): ověř, kdo a kde jsi, a jestli tabulka existuje
SELECT SYS_CONTEXT('USERENV', 'SESSION_USER') AS uzivatel, SYS_CONTEXT('USERENV', 'CON_NAME') AS pdb FROM dual;
SELECT table_name FROM user_tables WHERE table_name = 'ZBOZI';
```

Pokud `ZBOZI` chybí, vytvoř ji ve svém schématu:

```sql
CREATE TABLE zbozi AS
SELECT level AS id, 'Zboží ' || level AS nazev, MOD(level, 50) + 1 AS kategorie_id,
       ROUND(DBMS_RANDOM.VALUE(10, 50000), 2) AS cena
FROM dual CONNECT BY level <= 100000;
ALTER TABLE zbozi ADD CONSTRAINT pk_zbozi PRIMARY KEY (id);
```

Pokud `CREATE TABLE` skončí chybou oprávnění nebo kvóty, dej si jako SYS ve `FREEPDB1` (připojení SYS_PDB1) `GRANT CREATE TABLE TO vojtechvolf;` a `ALTER USER vojtechvolf QUOTA UNLIMITED ON <tvůj_default_tablespace>;`.

Pro krok 6 (skript) vytvoř předem `mkdir C:\backup\skripty` a `mkdir C:\backup\logy`, soubor `level1.rman` a ověř, že `SHOW ALL` ukazuje `RECOVERY WINDOW OF 7 DAYS`.

**📝 Pro sebe — ověř nanečisto:**
- že `FREEPDB1` je `READ WRITE` a že jsi ve správném účtu (`VOJTECHVOLF`),
- do jakého tablespace padá tabulka: `SELECT default_tablespace FROM user_users;` (Tomášova otázka),
- chování `PLUS ARCHIVELOG` (vynucená archivace, záloha logů, záloha DB, znovu archivace) a to, že po chybě uvnitř `RUN { ... }` se zbytek bloku neprovede. Obojí říkám z dokumentace, ne z vlastní zkušenosti.

---

## Časový plán

| Krok | Co | Čas | Živě / předpřipraveno |
|---|---|---|---|
| 1 | Připojení a CONFIGURE | 2 min | živě |
| 2 | Plná záloha | 2 min | předpřipravit, ukázat `LIST` |
| 3 | L0, změna, L1 **bez BCT** | 4 min | L0 předpřipravit, L1 živě |
| 4 | Zapnout BCT, nový L0, změna, L1 **s BCT** | 4 min | L0 předpřipravit, L1 živě |
| 5 | Porovnání (dotazy) | 3 min | živě |
| 6 (bonus) | RMAN skript | 2 min | soubor předpřipravit, živě jen spustit |

---

## Krok 1 👥: Připojení a nastavení (2 min)

```
rman target /
RMAN> SHOW ALL;
RMAN> CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 7 DAYS;
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON;
RMAN> CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO COMPRESSED BACKUPSET;
RMAN> CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO DISK;
RMAN> SHOW ALL;
```

**🎤 Říkám:**

Začneme nastavením. Do RMAN se připojím ke kořeni CDB přes operační systém, takže nezadávám žádné heslo. Heslo napsané v příkazu by zůstalo v historii příkazů a bylo by vidět ve výpisu procesů. A protože jsem připojený ke kořeni, vidím celou CDB včetně všech PDB.

*(spustit `SHOW ALL`)* SHOW ALL ukáže trvalé nastavení, které si RMAN drží v řídicím souboru. Spustím ho dvakrát, před změnami a po nich, abyste viděli rozdíl. Výchozí politika uchovávání je jedna záloha každého souboru a autozáloha řídicího souboru je od verze 12.2 už zapnutá.

*(spustit čtyři příkazy `CONFIGURE`)* Teď čtyři změny. První je politika uchovávání, okno obnovy sedm dní. Tím říkám RMAN: musím se umět vrátit k libovolnému okamžiku za posledních sedm dní. Co už k tomu není potřeba, je zastaralé. Vypíše to REPORT OBSOLETE a smaže DELETE OBSOLETE.

Druhá je autozáloha řídicího souboru. V řídicím souboru je evidence všech záloh. Kdybychom ho ztratili a neměli jeho zálohu, RMAN by o zálohách nevěděl, i když fyzicky leží na disku. Je to už výchozí, ale nastavuju to výslovně, ať víme, proč tam je.

Třetí je jeden kanál a komprimované zálohy. Více kanálů je funkce Enterprise Edition, ve Free necháváme jedna. Komprese šetří místo za cenu větší zátěže procesoru, uvidíme to ve sloupci compression ratio.

Čtvrtá je politika mazání archivních logů: log se smí smazat, až když je alespoň jednou zazálohovaný. Chrání to logy, které ještě v žádné záloze nejsou.

Formát záloh nenastavuju, takže všechno půjde do Fast Recovery Area, kterou Oracle spravuje sám. A celé nastavení platí pro celou CDB.

*(spustit druhý `SHOW ALL` a ukázat změny)*

**📝 Pro sebe:** výchozí `CONTROL_FILE_RECORD_KEEP_TIME` je 7 dní, tedy stejně jako okno. Přednáška říká, že má být delší. Pro ukázku to nevadí, jen kdyby se někdo zeptal.

---

## Krok 2 👥: Plná záloha celé CDB (2 min)

```
RMAN> BACKUP DATABASE PLUS ARCHIVELOG TAG 'plna';
RMAN> LIST BACKUP SUMMARY;
```

**🎤 Říkám:**

*(ukázat předpřipravený výstup, případně spustit)* Teď plná záloha celé CDB. Příkaz BACKUP DATABASE zálohuje všechny datové soubory kořene, PDB seed i všech PDB. PLUS ARCHIVELOG přidá archivní logy. RMAN nejdřív vynutí archivaci aktuálního logu, zazálohuje logy, potom databázi a nakonec ještě zbytek logů. Díky tomu se záloha dá později obnovit až do konce. TAG je jen jméno, podle kterého ji najdu v přehledech.

*(spustit `LIST BACKUP SUMMARY`)* V přehledu se dívejte na čtyři sloupce. TY je typ, B znamená sada záloh. LV je úroveň: F je plná záloha, nula a jednička jsou inkrementální a A jsou archivní logy. S je stav, A znamená dostupná. A TAG, to je naše jméno. Všimněte si, že vznikla i autozáloha řídicího souboru, ta se udělala sama.

Důležité: tohle je plná záloha, ne úroveň nula. Na plné záloze se nedají stavět inkrementální zálohy. Zálohy leží ve Fast Recovery Area. *(ukázat umístění z `SHOW PARAMETER db_recovery_file_dest`)*

---

## Krok 3 👥: Úroveň 0, změna dat, úroveň 1 bez BCT (4 min)

```
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE TAG 'lvl0';
```

```sql
-- VOJTECHVOLF, FREEPDB1
UPDATE zbozi SET cena = cena * 1.1 WHERE MOD(id, 10) = 0;
COMMIT;
```

```
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE TAG 'lvl1_bez_bct';
```

**🎤 Říkám:**

Teď inkrementální zálohy. Nejdřív úroveň nula. *(předpřipravená)* Obsahem je stejná jako plná záloha, ale je označená jako základ inkrementální řady. Od ní se budou počítat všechny další změny.

*(přepnout na připojení VOJTECHVOLF, spustit `UPDATE` a `COMMIT`)* Teď změním data. Zvýším cenu každého desátého řádku, to je deset tisíc řádků. Jsou rozptýlené po celé tabulce, takže se změní skoro všechny její bloky. Ale tabulka je jen malá část všech souborů databáze. COMMIT změny potvrdí, bez něj by transakce zůstala otevřená. Změna vytváří undo a redo, takže přibývají archivní logy.

*(spustit L1 bez BCT)* A teď inkrementální záloha úrovně jedna. Je rozdílová, protože nemá slovo CUMULATIVE, a zálohuje jen bloky změněné od úrovně nula. Teď pozor. Bez Block Change Tracking neví RMAN, které bloky se změnily. Musí tedy přečíst všechny bloky všech souborů a porovnat jejich SCN se SCN základní zálohy. Výsledná záloha je malá, ale čtení trvá skoro stejně dlouho jako u úrovně nula. Přesně to si za chvíli ukážeme v číslech.

---

## Krok 4 👥: Zapnout BCT a zopakovat (4 min)

```sql
-- SYS_CDB (jen pokud BCT ještě není ENABLED)
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING USING FILE 'C:\app\oradata\FREE\bct_free.chg';
SELECT status, filename FROM v$block_change_tracking;
```

```
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE TAG 'lvl0_bct';
```

```sql
-- VOJTECHVOLF, FREEPDB1
UPDATE zbozi SET cena = cena * 1.1 WHERE MOD(id, 10) = 0;
COMMIT;
```

```
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE TAG 'lvl1_bct';
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE TAG 'lvl1_kum';
```

**🎤 Říkám:**

Teď zapneme Block Change Tracking. *(spustit `ENABLE`)* Databáze od teď zapisuje do speciálního souboru, které bloky se změnily, a dělá to proces CTWR. Ten soubor je jeden pro celou CDB. *(spustit `SELECT`)* Ověřím, že je stav ENABLED.

Hned po zapnutí ale BCT ještě nemá od čeho počítat. Potřebuju nový základ. *(spustit nebo ukázat předpřipravenou `lvl0_bct`)* Je to další záloha úrovně nula. Kdybych ji vynechal, první záloha úrovně jedna by stejně četla všechno.

*(přepnout na VOJTECHVOLF, spustit `UPDATE` a `COMMIT`)* Znovu změním stejná data, abychom porovnávali srovnatelné věci.

*(spustit `lvl1_bct`)* A teď záloha úrovně jedna se zapnutým BCT. RMAN nečte všechno, čte jen bloky, které jsou v souboru sledování označené jako změněné.

*(spustit `lvl1_kum`)* A ještě jedna, kumulativní. Obsahuje změny od poslední úrovně nula. Protože jsme udělali jen jednu změnu, má stejný obsah jako ta rozdílová. Rozdíl by byl vidět až po několika dnech záloh.

**📝 Pro sebe:** tag `LVL1_KUM` není ve filtru dotazu v kroku 5. Chceš-li ji ukázat, přidej ho do seznamu v `IN (...)`.

---

## Krok 5 👤: Porovnání (3 min)

Každý si může dotazy pustit ve svém SQL Developeru (připojení SYS_CDB).

```sql
-- velikost a doba trvání
SELECT session_key, input_type, status, TO_CHAR(start_time, 'HH24:MI:SS') AS start_, time_taken_display,
       input_bytes_display, output_bytes_display, compression_ratio
FROM   v$rman_backup_job_details ORDER BY start_time;

-- kolik bloků záloha přečetla a kolik jich uložila
SELECT DISTINCT bp.tag, bd.file#, bd.incremental_level, bd.used_change_tracking,
       bd.datafile_blocks, bd.blocks_read, bd.blocks AS bloku_v_zaloze
FROM   v$backup_datafile bd
JOIN   v$backup_piece bp ON bp.set_stamp = bd.set_stamp AND bp.set_count = bd.set_count
WHERE  bp.tag IN ('LVL0', 'LVL1_BEZ_BCT', 'LVL0_BCT', 'LVL1_BCT')
ORDER BY bd.file#, bp.tag;
```

**🎤 Říkám:**

*(pustit první dotaz)* Teď porovnání. První dotaz ukazuje jeden řádek za každý spuštěný RMAN job. Vidíme, jak dlouho trval, kolik dat přečetl, kolik zapsal a jakou měl kompresi. Nemá tag, takže řádky poznáme podle pořadí: úroveň jedna bez BCT je před tou s BCT.

*(pustit druhý dotaz)* Druhý dotaz je zajímavější. Pro každý datový soubor a každou zálohu ukazuje tři čísla. DATAFILE_BLOCKS je počet bloků celého souboru. BLOCKS_READ je, kolik z nich záloha přečetla. A BLOCKS je, kolik jich uložila do zálohy.

*(ukázat řádky `LVL1_BEZ_BCT`)* U zálohy bez BCT je BLOCKS_READ skoro stejné jako DATAFILE_BLOCKS. RMAN prošel celý soubor, přestože uložil jen malou část. USED_CHANGE_TRACKING je NO.

*(ukázat řádky `LVL1_BCT`)* A u zálohy s BCT je BLOCKS_READ výrazně menší, protože RMAN četl jen změněné bloky, a USED_CHANGE_TRACKING je YES. Největší čísla čekejte u souboru s naší tabulkou a u undo. Ostatní soubory se skoro nezměnily.

U tak malé databáze může být rozdíl v čase malý. Důležité je, kolik bloků se četlo. U databáze s terabajty to rozhoduje o hodinách.

**📝 Pro sebe:** hodnoty si ověř na svém výstupu, nepřebírej je odsud.

---

## Krok 6 (jen pokud zbude čas) 👥: RMAN skript (2 min)

Soubor `C:\backup\skripty\level1.rman` (při ukládání v Poznámkovém bloku vyber „Všechny soubory“, jinak se přidá `.txt`):

```
RUN {
  BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG TAG 'tyden_l1';
  DELETE NOPROMPT OBSOLETE;
}
```

Spuštění z příkazového řádku Windows (ne uvnitř RMAN):

```
rman target / cmdfile=C:\backup\skripty\level1.rman log=C:\backup\logy\level1.log
type C:\backup\logy\level1.log
```

**🎤 Říkám:**

*(ukázat soubor ve Poznámkovém bloku)* Poslední věc je skript. Příkazy můžu uložit do souboru a spustit je bez klikání, a takhle je spouští plánovač třeba v noci. Skript je jednoduchý: blok RUN a v něm záloha úrovně jedna a mazání zastaralých záloh. RUN provede příkazy jako celek. Když záloha uvnitř selže, zbytek se neprovede, takže se po neúspěšné záloze nic nemaže.

DELETE NOPROMPT OBSOLETE maže bez potvrzení, protože skript běží bez člověka. Je to nevratné, proto před ním vždycky kontroluju politiku uchovávání.

*(spustit z příkazového řádku)* Spouštím RMAN s parametrem cmdfile, to je soubor se skriptem, a log, to je soubor, kam se uloží výstup. Připojení je opět přes operační systém, takže ve skriptu ani v příkazu není heslo. Ráno se pak výsledek kontroluje z logu, ne z obrazovky. *(spustit `type` a ukázat log)*

**📝 Pro sebe:** skript spustí skutečnou zálohu L1, takže trvá stejně dlouho jako v kroku 3. Ukaž hlavně soubor a log.

---

## Krok 7 (jen pokud zbude čas) 👥: LIST a REPORT

```
RMAN> LIST BACKUP SUMMARY;
RMAN> REPORT NEED BACKUP;
RMAN> REPORT OBSOLETE;
RMAN> BACKUP PLUGGABLE DATABASE freepdb1 TAG 'pdb_demo';
RMAN> LIST BACKUP OF PLUGGABLE DATABASE freepdb1 SUMMARY;
```

**🎤 Říkám:**

A ještě kontrola. REPORT NEED BACKUP vypíše soubory, které nemají dost záloh. Prázdný výstup je dobře. REPORT OBSOLETE vypíše zastaralé zálohy. Při sedmidenním okně by měl být prázdný a něco vypíše, jen když někdo mezitím změnil politiku, třeba na redundanci dva.

Nakonec BACKUP PLUGGABLE DATABASE, ten zálohuje jen jednu PDB. Funguje, ale k obnově samotná nestačí. Potřebuje i archivní logy a řídicí soubor z kořene, a ty už máme z předchozích kroků.

---

## Závěr ukázky (20 s)

**🎤 Říkám:**

Shrnu to. Plná záloha a úroveň nula mají stejný obsah, ale jen úroveň nula je základ pro inkrementální zálohy. Záloha úrovně jedna je malá vždycky, ale bez BCT přečte skoro všechno. Nastavení RMAN je trvalé a platí pro celou CDB. A teď se dostaneme k úkolu.

---

## Po ukázce (neříkat)

BCT a nastavení RMAN jsou společné pro celou CDB. Nic nemaž ani nevypínej, dokud neproběhnou navazující témata (obnova, Flashback). Úklid dělá jeden člen na konci cvičení (kapitola 5).

## Otázky ke studentům a odpovědi

**🎤 Ptám se:**

- „Proč používáme úroveň nula a ne obyčejný BACKUP DATABASE, když mají stejný obsah?“
- „Co by se stalo, kdybychom zapomněli zapnout autozálohu řídicího souboru?“
- „Proč byla záloha úrovně jedna bez BCT malá, ale trvala skoro jako úroveň nula?“

**🎤 Kdyby nikdo nereagoval, odpovídám:**

- Obsah je stejný, ale jen úroveň nula je základ pro úroveň jedna. Plná záloha tuto roli nemůže plnit.
- Po ztrátě řídicího souboru by RMAN o zálohách nevěděl, protože evidence je právě v něm, i když zálohy leží na disku. Od verze 12.2 je autozáloha výchozí, ale nastavujeme ji výslovně.
- Výsledek je malý, protože obsahuje jen změněné bloky. Aby ale RMAN zjistil které, musel přečíst všechny bloky a porovnat jejich SCN.
