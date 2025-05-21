---
tags: RPGLE SQL 
---
## Avoiding SQL injections in ILE-RPG with embedded SQL (SQLRPGLE)

### The problem: user-input in dynamic SQL statements

With dynamic statements in SQLRPGLE you always have to be careful with user-inputs. Users can enter malicious data like `' or 1=1--` - and if this code is "embedded" into a dynamic SQL statement, it can be harmful.

So since long, every SQL evangelist ist preaching to use SQL parameter markers - e.g. `?` - in dynamic SQL, and strictly avoid ebedding user entered strings into the SQL statements. The `?` markers later get their values, during execution of the SQL statements - and the values are **never** "embedded", but always entered typesafe. So the string `' or 1=1--` above will be entered as a value into the statement - and therefore treated as literal data, not as SQL code.

Using this is quite easy:

```rpgle
  exec sql prepare stmDelete using 'DELETE FROM MYTABLE WHERE COL1 = ?';
  exec sql execute stmDelete using :hostVar;
```

#### Really dynamic SQL statements

But what do you do, if your statement is "really dynamic"? I mean, if your `WHERE` conditions are created at runtime depending on user input. In this case, you probably don't know, how many `?` markers you will have in the end? 

But there is also a solution for cases like this - SQL indicator variables!

Let's look at a dynamic `SELECT` statement for a cursor - the typical solution might have been like that:
 
```rpgle
  ##sql01 = 'select mytable.* +
             from mytable +
             where mytable.col1 = '+%char(EntryFirma)+' +
               and mytable.col2 = '''+%trim(EntryNdl)+'''';
 
    // WHERE mit SQL Injection Protection
    if gibstyphf <> *blanks;
      ##sql01 += ' and ibsist.gibstyp like ''%'''+%trim(gibstyphf)+'''%''';
    endif;
    if hlevelhf <> *blanks;
      ##sql01 += ' and ibsist.hlevel like ''%'''+%trim(hlevelhf)+'''%''';
    endif;
```
 
Da die diversen Suchkriterien immer nur dann angewender werden sollen, wenn auch eine Suche gewünscht ist, ist es schwer dieses Anweisungen in statischen SQL zu schreiben.
 
Aber durch einschleusen von "bösartigen" Eingabewerten, wie z.B. `' or 1=1--` im Feld `GIBSTYPHF` könnte die SQL-Engine dazu gebracht werden, etwas falsches anzuzeigen.
 
### Lösung in Embedded SQL
 
Tatsächlich kann man das Problem umgehen, indem man (mindestens für alle Zeichenketten) SQL Parameter-Marker im dynamischen SQL nutzt.
 
Dafür benötigen wir zuerst für jeden "dynamischen" Suchwert (der also nur dann gesucht werden soll, wenn er auch angegeben wurde) eine SQL-Indikator Variable:
 
```sql
  dcl-s #GibsTypHF int(5) inz;
  dcl-s #HLevelHF int(5) inz;
```
 
Im Anschluss wird die dynamische SQL Anweisung zusammengebaut:
 
```sql
    // BasisSQL zusammenstellen mit SQL Injection Protection
    ##sql01 = 'select ibsist.* +
               from ibsist  +
               where ibsist.firma = ? +
                 and ibsist.ndl = ?';
 
    // WHERE mit SQL Injection Protection
    if gibstyphf <> *blanks;
      ##sql01 += ' and ibsist.gibstyp like ''%'' concat trim(cast(? as varchar(10))) concat ''%''';
      #GibsTypHF = *zero;
    else;
      #GibsTypHF = -7;
    endif;
    if hlevelhf <> *blanks;
      ##sql01 += ' and ibsist.hlevel like ''%'' concat trim(cast(? as varchar(10))) concat ''%''';
      #HLevelHF = *zero;
    else;
      #HLevelHF = -7;
    endif;            
```
 
Wenn ein Wert gesucht werden soll, wird die SQL-Anweisung um die entsprechende `WHERE`-Bedingung erweitert, und der jeweilige SQL-Indikator auf `0` gesetzt.
 
Wenn ein Wert NICHT gesucht werden soll, muss nur der jeweilige SQL-Indikator auf `-7` gesetzt werden.
 
Zuletzt muss die SQL Anweisung `OPEN` mit einer `USING SUBSET` Klausel erweitert werden:
 
```sql
      exec sql prepare sqlview01 from :##sql01;
      if sqlcode=*zeros;
        exec sql open sqlcurs01 using subset :EntryFirma, :EntryNdl, :gibstyphf:#GibsTypHF, :hlevelhf:#HLevelHF, ...;
      endif;
```
 
Normalerweise muss für jeden `?` SQL Parameter Marker eine `:`Host-Variable in der `USING` Klausel angegeben werden.
 
Für Firma und Niederlassung ist dies kein Problem - da wir aber nicht wissen, wieviele weitere Suchwerte im SQL-String angegeben wurden, wäre dies ein Problem.
 
Derhalb wird die `USING SUBSET` Klausel in Kombination mit den SQL-Indikatoren verwendet.
Steht nämlich der jeweilige Indikator (z.B. `:#GibsTypHF`) einer `:`Host-Variable (z.B. `:gibstyphf`) auf `-7`, dann wird die jeweilige `:`Host-Variable in der `USING` Klausel komplett ignoriert - ganz so, als wäre sie nicht angegeben worden.
 
Durch diese Technik enthält die dynamische SQL-Anweisung dann genauso so viele `?` Parameter Marker wie die `USING SUBSET` Klausel effektiv angibt. Wichtig ist aber, immer auf die korrekte Reihenfolge zu achten.
 
In der `USING SUBSET` Klausel können `:`Host-Variablen MIT und OHNE SQL-Indikator gemischt werden - solange am Ende immer die richtige Reihenfolge eingehalten wird. Es empfiehlt sich, alle "fixen" Such-Kriterien (hier z.B. Firma und Niederlassung) zuerst zu behandeln - dies ist aber nicht zwingend so.
 
#### Exkurs: Warum ausgerechnet `trim(cast(? as varchar(10)))`?
 
Im dynamischen SQL wird um den `?` Parameter Marker herum ein `trim(cast(? as varchar(10)))` konstruiert.
 
Hintergrund ist ein noch nicht behobener Fehler, der verhindert, dass nicht direkt `trim(?)` angegeben werden kann. Die `TRIM` Funktion (und noch einige andere) darf nicht direkt auf einen `?` Parameter Marker angewendet werden.
 
Durch den `CAST` auf den Datentyp `VARCHAR` wird dies effektiv verhindert, und `TRIM` bekommt einen SQL-Ausdruck als Eingabe - was dann zulässig ist.
 
Beim `CAST` ist jeweils auf die Länge des Eingabe-Feldes zu achten. Ein `CAST` in einen längeren `VARCHAR` Wert ist aber unschädlich.
 
### Weitere Anwendungen
 
Die Technik lässt sich so auch in dynamischen `UPDATE` oder `INSERT` Anweisungen einsetzen.
 
### SQL-Indikator-Werte
 
| Wert | Bedeutung |
|------|-----------|
| `0`  | der Wert der SQL Host-Variable soll verwendet werden |
| `-1`<br>`-2`<br>`-3`<br>`-4`<br>`-6` | <br><br>der Wert `NULL` soll verwendet werden |
| `-5` | der `DEFAULT` Wert der Spalte soll verwendet werden |
| `-7` | die `:`Host-Variable soll ignoriert werden ("not assigned") |
 
Weitere Infos: https://www.ibm.com/docs/en/i/7.5.0?topic=sql-indicator-variables-used-assign-special-values
 
## Problem: Dynamische SQL-Anweisungen in "Regelwerken" o.ä.
 
Werden SQL Anweisungen in einer Datenbank-Tabelle gespeichert - z.B. für ein Regelwerk o.ä. - könnte ein bösartiger Anwender dort ggf. auch eine `DELETE` oder eine `DROP TABLE` Anweisung einschleusen.
 
### Hilfe
 
Man kann das Problem nicht 100%ig ausschließen, aber durch eine Analyse der SQL-Anweisung, bevor diese ausgeführt wird, kann man das Problem zumindest größtenteils vermeiden.
 
Dazu bietet sich die SQL Tabellen-Funktion [`PARSE_STATEMENT`](https://www.ibm.com/docs/en/i/7.5.0?topic=services-parse-statement-table-function) an. Diese analysiert die Anweisung, OHNE sie auszuführen.
 
Beispiel:
```sql
select *
from table(
    parse_statement('select aaaa01, aaaa02, aac013, aac014 from gfaada')
)
```
|NAME_TYPE|NAME  |SCHEMA|RDB|COLUMN_NAME|ADDITIONAL_NAME|USAGE_TYPE|NAME_START_POSITION|SQL_STATEMENT_TYPE|
|---------|------|------|---|-----------|---------------|----------|-------------------|------------------|
|COLUMN   |-     |-     |-  |AAAA01     |               |QUERY     |8                  |QUERY             |
|COLUMN   |-     |-     |-  |AAAA02     |               |QUERY     |16                 |QUERY             |
|COLUMN   |-     |-     |-  |AAC013     |               |QUERY     |24                 |QUERY             |
|COLUMN   |-     |-     |-  |AAC014     |               |QUERY     |32                 |QUERY             |
|TABLE    |GFAADA|-     |-  |-          |               |QUERY     |44                 |QUERY             |
 
Wenn in dieser Tabelle in der Spalte `SQL_STATEMENT_TYPE` ein anderer Wert als `QUERY` steht, so handelt es sich nicht um eine reine `SELECT` Anweisung.
 
Der Code im Programm könnte also wie folgt aussehen:
```sql
exec sql select count(*), count(case sql_statement_type when 'QUERY' then 1 end)
         into :nCountAll, :nCountQry
         from table(parse_statement(:##sql));
if sqlcode < *zero;
  // richtiger SQL-Fehler - nicht in der dynamischen Anweisung
elseif sqlcode > *zero or nCountAll = *zero;
  // sehr wahrscheinlich Syntax-Fehler in der dynamischen SQL Anweisung
elseif nCountQry <> nCountAll;
  // die Anweisung ist kein reiner SELECT
end;
```
 
Aber auch in einem reinen `SELECT` ließe sich bösartiges einschleusen - z.B. könnten mit der Funktion `QCMDEXC` System-Befehle ausgeführt werden. Man müsste also zusätzlich noch eine Liste von Funktionen anlegen, die grundsätzlich nicht in diesen SQLs eingebaut werden dürfen.

