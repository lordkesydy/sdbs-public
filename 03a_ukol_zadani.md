# RMAN: Úkol — zadání (15 min)

> **Téma:** Politika uchovávání a týdenní strategie záloh.
> **Cíl:** Nastavit politiku uchovávání, navrhnout strategii záloh na týden, připravit skripty a ověřit je příkazy LIST a REPORT.

---

## Legenda

| Symbol | Kdo to dělá |
|---|---|
| 👥 **SPOLEČNĚ** | Na adminu v kořeni CDB. Provádí jeden člen skupiny, druhý sleduje. Nastavení i zálohy jsou společné pro celou CDB. |
| 👤 **KAŽDÝ** | Každý samostatně ve svém připojení (SQL Developer). |
| 📝 **NA PAPÍRE** | Navrhujete a zapisujete, nespouštíte. |

## Předpoklady

- Databáze běží v režimu **ARCHIVELOG** (`ARCHIVE LOG LIST`).
- Máte hotové Cvičení 4: ve své PDB existuje uživatel **CV4** a tabulka **ZBOZI**.
- RMAN je nastavený a existuje alespoň jedna záloha (např. z ukázky).

## Scénář

Jste správci databáze skupiny. Vedení chce mít možnost obnovy k libovolnému okamžiku za posledních 7 dní a jednou týdně ověřit, že se ze záloh dá obnovit.

---

## Úkol 1 — Politika uchovávání (4 min)

### 👥 SPOLEČNĚ

1. Zjistěte aktuální politiku uchovávání.
2. Změňte ji na **redundanci 2** a zjistěte, které zálohy jsou nyní zastaralé (**jen vypsat, nemazat**).
3. Vraťte **okno obnovy 7 dní**.

> ⚠️ **Nemažte zastaralé zálohy**, dokud je nastavená redundance 2. Zálohy potřebují i navazující témata (obnova).

> 💡 **Nápověda:** `SHOW ALL`, `CONFIGURE RETENTION POLICY ...`, `REPORT OBSOLETE`.

> ❓ **Zamyslete se:** Proč se po změně na redundanci 2 mohou některé zálohy označit jako zastaralé, i když jsou staré jen hodiny? Čím se redundance liší od okna obnovy?

---

## Úkol 2 — Týdenní strategie (5 min)

### 📝 NA PAPÍRE (oba)

Navrhněte týdenní strategii pro databázi vaší skupiny. Doplňte tabulku, **sloupec „Proč“ vyplňte sami**.

| Den | Záloha (úroveň, typ) | Proč |
|---|---|---|
| Neděle | | |
| Pondělí | | |
| Úterý | | |
| Středa | | |
| Čtvrtek | | |
| Pátek | | |
| Sobota | | |

A zodpovězte:
- Kdy a čím se **maže** (zastaralé zálohy)?
- Kdy a čím se **ověřuje**, že zálohy jsou použitelné?
- Co by bylo jiné, kdyby RPO i RTO byly mnohem přísnější?

> 💡 **Nápověda:** Úrovně `0` a `1`, `CUMULATIVE`, `PLUS ARCHIVELOG`, `DELETE ... OBSOLETE`, `REPORT NEED BACKUP`, `RESTORE DATABASE VALIDATE`.

### 👥 SPOLEČNĚ — skripty

Připravte dva skripty (adresáře `C:\backup\skripty` a `C:\backup\logy` vytvořte předem): `level0.rman` a `level1.rman`. Každý provede zálohu příslušné úrovně včetně archivních logů a smaže zastaralé zálohy. Spusťte je z příkazového řádku Windows.

> 💡 **Nápověda:** blok `RUN { ... }`, spuštění `rman target / cmdfile=... log=...`.

---

## Úkol 3 — Vyzkoušení a ověření (6 min)

### 👤 KAŽDÝ — vygenerujte změny

Jako **CV4** ve své PDB (autocommit vypnutý):

```sql
UPDATE zbozi SET cena = cena * 1.1 WHERE MOD(id, 10) = 0;
COMMIT;
```

### 👥 SPOLEČNĚ — zálohy

1. Spusťte `level0.rman`.
2. Počkejte, až oba členové provedou změnu dat, a spusťte `level1.rman`.
3. Vyzkoušejte i kumulativní záchytný bod ze středy z vaší strategie (jedna záloha úrovně 1 kumulativní).

### 👥 SPOLEČNĚ — kontrola v RMAN

Ověřte strategii příkazy **LIST** a **REPORT**: přehled záloh, soubory bez dostatečné zálohy, zastaralé zálohy.

### 👤 KAŽDÝ — kontrola v SQL

Ve svém připojení k CDB (SYS_CDB) zjistěte z pohledů velikost, dobu trvání a typ vašich záloh.

> 💡 **Nápověda:** `LIST BACKUP SUMMARY`, `REPORT NEED BACKUP`, `REPORT OBSOLETE`, pohled `v$rman_backup_job_details`.

> ❓ **Zamyslete se:** Který ze souborů v hlášení `REPORT NEED BACKUP` by se objevil, kdybyste týden nezálohovali? Čím se liší záloha **L1 kumulativní** od **L1 rozdílové** při obnově?

---

## Bonus (pro rychlé) 👤

Z výstupu `v$rman_backup_job_details` porovnejte `input_bytes_display` a `output_bytes_display` u L0 a L1. Proč se liší?
