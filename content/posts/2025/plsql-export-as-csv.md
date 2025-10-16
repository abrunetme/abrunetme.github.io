+++
title = "Exporter une requête Oracle en CSV avec une fonction PL/SQL"
date = 2025-10-15
draft = true
tags= ["oracle", "plsql", "csv", "sql", "export"]
+++

Dans certains contextes, il est utile d’exporter le résultat d’une requête Oracle directement au format **CSV**, que ce soit pour le partager, l’analyser dans Excel ou l’intégrer à un autre système.

Plutôt que d’utiliser des outils externes ou SQL Developer, voici une **fonction PL/SQL autonome** permettant d’obtenir un export CSV à partir d’une requête SQL dynamique.

<!--more-->

## Function `export_as_csv`

Cette fonction PL/SQL exécute une requête passée en paramètre, parcourt le jeu de résultats et construit un texte au format **CSV**.  
Le tout est retourné sous forme de **CLOB**, idéal pour les gros volumes.

### Signature

```sql
CREATE OR REPLACE FUNCTION export_as_csv(
    p_query         IN VARCHAR2,
    p_with_header   IN VARCHAR2  DEFAULT 'TRUE',
    p_separator     IN VARCHAR2  DEFAULT ';'
) RETURN CLOB
```
* `p_query` — la requête SQL à exécuter
* `p_with_header` — indique si les noms de colonnes doivent être inclus dans la première ligne (TRUE / FALSE)
* `p_separator` — séparateur de colonnes (par défaut ;)

### Code de la fonction

```sql
CREATE OR REPLACE FUNCTION export_as_csv(
    p_query         IN VARCHAR2,
    p_with_header   IN VARCHAR2  DEFAULT 'TRUE',
    p_separator     IN VARCHAR2  DEFAULT ';'
) RETURN CLOB
AS
    v_cursor_id    INTEGER;
    v_col_cnt      INTEGER;
    v_desc_tab     DBMS_SQL.DESC_TAB;
    v_col_value    VARCHAR2(4000);
    v_row_csv      VARCHAR2(32767);
    v_result       CLOB := EMPTY_CLOB();
    v_status       INTEGER;
    v_first_col    BOOLEAN;
BEGIN
    v_cursor_id := DBMS_SQL.OPEN_CURSOR;

    BEGIN
        DBMS_SQL.PARSE(v_cursor_id, p_query, DBMS_SQL.NATIVE);
        DBMS_SQL.DESCRIBE_COLUMNS(v_cursor_id, v_col_cnt, v_desc_tab);

        FOR i IN 1 .. v_col_cnt LOOP
            DBMS_SQL.DEFINE_COLUMN(v_cursor_id, i, v_col_value, 4000);
        END LOOP;

        v_status := DBMS_SQL.EXECUTE(v_cursor_id);

        IF UPPER(p_with_header) = 'TRUE' THEN
            v_row_csv := '';
            FOR i IN 1 .. v_col_cnt LOOP
                IF i > 1 THEN
                    v_row_csv := v_row_csv || p_separator;
                END IF;
                v_row_csv := v_row_csv || v_desc_tab(i).col_name;
            END LOOP;
            v_result := v_result || v_row_csv || CHR(10);
        END IF;

        WHILE DBMS_SQL.FETCH_ROWS(v_cursor_id) > 0 LOOP
            v_row_csv := '';
            v_first_col := TRUE;
            FOR i IN 1 .. v_col_cnt LOOP
                DBMS_SQL.COLUMN_VALUE(v_cursor_id, i, v_col_value);
                IF NOT v_first_col THEN
                    v_row_csv := v_row_csv || p_separator;
                END IF;
                v_row_csv := v_row_csv || REPLACE(NVL(v_col_value, ''), p_separator, ' ');
                v_first_col := FALSE;
            END LOOP;
            v_result := v_result || v_row_csv || CHR(10);
        END LOOP;

    EXCEPTION
        WHEN OTHERS THEN
            DBMS_SQL.CLOSE_CURSOR(v_cursor_id);
            RAISE;
    END;

    DBMS_SQL.CLOSE_CURSOR(v_cursor_id);
    RETURN v_result;
END;
/
```

### Explications détaillées

```sql
v_cursor_id := DBMS_SQL.OPEN_CURSOR;
DBMS_SQL.PARSE(v_cursor_id, p_query, DBMS_SQL.NATIVE);
```
permet d'assigner un nouveau curseur et de l'associer à la requête. L’option `DBMS_SQL.NATIVE` indique que la requête doit être analysée comme une requête SQL Oracle standard.

```sql
DBMS_SQL.DESCRIBE_COLUMNS(v_cursor_id, v_col_cnt, v_desc_tab);
```
demande à Oracle d'extraire les méta-données de la curseur (cad. de la requête).
`v_desc_tab` va contenir la description de chaque colonne (nom, type, longeur, etc.) et `v_col_cnt` le nombre de colonnes (utile pour parcourir `v_desc_tab`).






