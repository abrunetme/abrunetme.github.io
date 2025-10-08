+++
title = "Calculer la date de Pâques en PL/SQL"
date = 2025-10-07
draft = false
+++

Le calcul de la date de Pâques permet de déterminer le **dimanche de Pâques** ainsi que les jours fériés associés comme le **lundi de Pâques**, l’**Ascension** et la **Pentecôte**. Il peut donc être utile de les calculer directement depuis une base de données Oracle.

<!--more-->

En PL/SQL, on peut implémenter **l'algorithme de Butcher-Meeus** pour déterminer le dimanche de Pâques pour une année donnée.

```sql
CREATE OR REPLACE FUNCTION dimanche_paques(p_year IN NUMBER)
   RETURN DATE
IS
   a NUMBER;
   b NUMBER;
   c NUMBER;
   d NUMBER;
   e NUMBER;
   f NUMBER;
   g NUMBER;
   h NUMBER;
   i NUMBER;
   k NUMBER;
   l NUMBER;
   m NUMBER;
   month NUMBER;
   day NUMBER;
BEGIN
   -- Algorithme de Butcher-Meeus
   a := MOD(p_year, 19);
   b := FLOOR(p_year / 100);
   c := MOD(p_year, 100);
   d := FLOOR(b / 4);
   e := MOD(b, 4);
   f := FLOOR((b + 8) / 25);
   g := FLOOR((b - f + 1) / 3);
   h := MOD(19 * a + b - d - g + 15, 30);
   i := FLOOR(c / 4);
   k := MOD(c, 4);
   l := MOD(32 + 2 * e + 2 * i - h - k, 7);
   m := FLOOR((a + 11 * h + 22 * l) / 451);
   month := FLOOR((h + l - 7 * m + 114) / 31); -- 3= mars, 4= avril
   day := MOD(h + l - 7 * m + 114, 31) + 1;

   RETURN TO_DATE(p_year || '-' || month || '-' || day, 'YYYY-MM-DD');
END;
```

On peut ensuite calculer les jours fériés associés.

Pour le lundi de Pâques :
```sql
CREATE OR REPLACE FUNCTION lundi_paques(p_year IN NUMBER)
   RETURN DATE
IS
	v_lundi_paques DATE;
BEGIN
	v_lundi_paques := dimanche_paques(p_year) + 1;
	RETURN v_lundi_paques;
END;
```


Pour le jeudi de l'Ascension :
```sql
CREATE OR REPLACE FUNCTION jeudi_ascension(p_year IN NUMBER)
   RETURN DATE
IS
	v_jeudi_ascension DATE;
BEGIN
	v_jeudi_ascension := dimanche_paques(p_year) + 39;
	RETURN v_jeudi_ascension;
END;
```

Pour le dimanche de la Pentecôte :
```sql
CREATE OR REPLACE FUNCTION dimanche_pentecote(p_year IN NUMBER)
   RETURN DATE
IS
	v_dimanche_pentecote DATE;
BEGIN
	v_dimanche_pentecote := dimanche_paques(p_year) + 49;
	RETURN v_dimanche_pentecote;
END;
```

Pour le lundi de la Pentecôte :
```sql
CREATE OR REPLACE FUNCTION lundi_pentecote(p_year IN NUMBER)
   RETURN DATE
IS
	v_lundi_pentecote DATE;
BEGIN
	v_lundi_pentecote := dimanche_pentecote(p_year) + 1;
	RETURN v_lundi_pentecote;
END;
```

Exemple d'utilisation :
```sql
BEGIN
	DBMS_OUTPUT.PUT_LINE('Dimanche de Pâques 2025 : ' || TO_CHAR(dimanche_paques(2025), 'DAY DD/MM/YYYY'));
	DBMS_OUTPUT.PUT_LINE('Lundi de Pâques 2025 : ' || TO_CHAR(lundi_paques(2025), 'DAY DD/MM/YYYY'));
	DBMS_OUTPUT.PUT_LINE('Jeudi Ascension 2025 : ' || TO_CHAR(jeudi_ascension(2025), 'DAY DD/MM/YYYY'));
   	DBMS_OUTPUT.PUT_LINE('Dimanche Pentecôte 2025 : ' || TO_CHAR(dimanche_pentecote(2025), 'DAY DD/MM/YYYY'));
	DBMS_OUTPUT.PUT_LINE('Lundi Pentecôte 2025 : ' || TO_CHAR(lundi_pentecote(2025), 'DAY DD/MM/YYYY'));
END;
```
affiche
```text
Dimanche de Pâques 2025 : DIMANCHE 20/04/2025
Lundi de Pâques 2025 : LUNDI    21/04/2025
Jeudi Ascension 2025 : JEUDI    29/05/2025
Dimanche Pentecôte 2025 : DIMANCHE 08/06/2025
Lundi Pentecôte 2025 : LUNDI    09/06/2025
```
