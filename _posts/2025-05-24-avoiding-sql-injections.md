---
tags: IBMi RPGLE SQL SQLRPGLE embeddedSQL
---
## Avoiding SQL injections in ILE-RPG with embedded SQL (SQLRPGLE)

Dynamic embedded SQL is a great tool, if you have to create your SQL statements depending on user input. But "inserting" those user inputs directly into your SQL statement string, can be very dangerous. This kind of attack is called SQL injection.

Even if many IBM i appications run in a very closed envorinment, [SQL injections can be real threat](https://xkcd.com/327/). So in this post, we explore a clever combination of SQL parameter markers and SQL indicators, that can help you to create injection-safe dynamic SQL statements.

### The problem: user-input in dynamic SQL statements

With dynamic statements in SQLRPGLE you always have to be careful with user-inputs. Users can enter malicious data like `' or 1=1--` - and if this code is "embedded" into a dynamic SQL statement, it can be harmful.

So since long, every SQL evangelist ist preaching to use SQL parameter markers - e.g. `?` - in dynamic SQL, and strictly avoid ebedding user entered strings into the SQL statements. 

So with this kind of code, you obviously can get into trouble:
```rpgle
exec sql prepare stmDelete using 'DELETE FROM MYTABLE WHERE COL1 = '''+%trim(myVar)+'''';
exec sql execute stmDelete;
```

If `myVar` contains `' or true --` the SQL statement will evaluate to:
```sql
DELETE FROM MYTABLE WHERE COL1 = '' or true --'
```
And as everything behind the SQL comment `--` is ignored, all rows in `MYTABLE` will get deleted. 

One solution might be, to search for "*forbidden*" characters like `'` oder strings like `--`. But then your users will be  unable to search for exactly these characters or strings. So thats an "sub-optimal" solution.

The real solution is, to avoid embedding strings and simply use `?` parameter markers, instead of appending strings into the statement - like this:
```rpgle
exec sql prepare stmDelete using 'DELETE FROM MYTABLE WHERE COL1 = ?';
exec sql execute stmDelete using :hostVar;
```

The `?` markers in the statement get their values during the `EXECUTE` or `OPEN` (for cursors) statement. During the execution the values are "insertet" in a typesafe way, and the are **never** "embedded" into the statement. So even a value like `' or 1=1--` will be entered as a **value** into the statement - and therefore treated as literal data, not as SQL code.


#### Really dynamic SQL statements

But what do you do, if your statement is "really dynamic"? I mean, if e.g. your `WHERE` conditions are created at runtime depending on user input. In this case, you probably don't know, how many `?` markers you will have in the end? 

But there is also a solution for cases like this - SQL indicator variables!

Let's look at a dynamic `SELECT` statement with parameter markers like this: 
```sql
selectStmt = 'SELECT MYTABLE.* +
              FROM MYTABLE  +
              WHERE MYTABLE.COL1 = ?';
if searchVal <> *blank;
  selectStmt += ' AND MYTABLE.COL2 LIKE ''%'' CONCAT TRIM(CAST(? AS VARCHAR(100))) CONCAT ''%''';
endif;
```

The problem is, that you would have to code different `OPEN` statements for the cursor - depending on whether the searchVal is `*BLANK` or not.

But SQL has a solution for that problem - SQL indicators! So here our new code:
```sql
dcl-s col2ind int(5) inz;

selectStmt = 'SELECT MYTABLE.* +
              FROM MYTABLE  +
              WHERE MYTABLE.COL1 = ?';

if col2val <> *blank;
  selectStmt += ' AND MYTABLE.COL2 LIKE ''%'' CONCAT TRIM(CAST(? AS VARCHAR(100))) CONCAT ''%''';
  col2ind = *zero;
else;
  col2ind = -7;
endif;

exec sql prepare stmSelect from :selectStmt;
if sqlcode = *zero;
  exec sql declare csrSelect cursor for stmSelect;
  exec sql open csrSelect using subset :col1val, :col2val :col2ind;
endif;
```

As you can see, the value of `col1val` and `col2val` are only used with the `OPEN` statement.

The magic is done by the clause `USING SUBSET`. Depending on the indicator `col2ind` for host-variable `col2val`, the `OPEN` statement gets executed in two different ways:

- `col2ind` = 0 -> `open csrSelect using :col1val, :col2val`
- `col2ind` = -7 -> `open csrSelect using :col1val`

In the second case, the host-variable `col2val` in the `USING SUBSET` clause is completely ignored. Therefore also the second `?` marker shouldn't be present in the prepared statement.

And now the "funny" part - you can mix host-variables with and without indicators in the `USING SUBSET` clause as you like. You only have to make sure, that in the end:

- You have exactly the number of host-variables without `-7` indicator as you have `?` parameter markers
- The sequence of the host-variables in the `USING SUBSET` clause is exactly matched to the sequence of the `?` parameter markers


#### But why of all things are you using `trim(cast(? as varchar(100)))`?

I'm using the somehow unnecessarily complex `trim(cast(? as varchar(100)))` to insert the parameter marker. And you may ask why?

And the reason for this is simple - our variable `col2val` is supposed to be of type `CHAR(..)`, and we want to trim leading and trailing blanks from the value, to make our `LIKE` search.

But there is a problem with that - you cannot code `TRIM(?)` in dynamic SQL directly. Due to an unresolved error in dynamic SQL, some scalar functions don't accept `?` parameter markers directly - and sadly `TRIM` is one of them. So we are wrapping the `?` marker in a `CAST` and this avoids the problem. 

Of course you can avoid that, if you simply use a host-variable of type `VARCHAR` with and already trimmed value in it. Instead of `trim(cast(? as varchar(100)))` you would then simply code `?`.

Maybe IBM is fixing this in the future - but even when they do, it won't affect your statements, that still use `CAST`.


#### Is this only for `OPEN` statements and dynamic SQL?
 
Short answer: No!

Somehow longer answer: You can also code `UPDATE` and `INSERT` statements with a variable number of `?` parameter markers, as `EXECUTE` also supports the `USING SUBSET` clause. And you can even use the `-7` indicator in static SQL - here an example:

```rpgle
exec sql update mytable
         set mytable.col1 = :col1val,
             mytable.col2 = :col2val :col2ind
         where mytable.id = :id
```

If the indicator `col2ind` is set to `-7`, the statement will be executed like this:
```rpgle
exec sql update mytable
         set mytable.col1 = :col1val
         where mytable.id = :id
```


#### Conclusion

Avoiding SQL injections is not so hard using indicators in dynamic embedded SQL. You should really use this trick at least with all character/string inputs. 

Thanks for reading.

If you have questions or ideas - I have opened the [Discussions](https://github.com/qpgmr-de/qpgmr-de.github.io/discussions)
page of this repository - if you like, leave a comment.

#### More information / Links

- [Info Center - SQL Indicator variable special values](https://www.ibm.com/docs/en/i/7.6.0?topic=sql-indicator-variables-used-assign-special-values)
- [Info Center - `OPEN` Statement](https://www.ibm.com/docs/en/i/7.6.0?topic=statements-open)
- [Info Center - `EXECUTE` Statement](https://www.ibm.com/docs/en/i/7.6.0?topic=statements-execute)

