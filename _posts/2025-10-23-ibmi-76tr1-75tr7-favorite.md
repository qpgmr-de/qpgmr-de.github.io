## My personal favorite feature from IBM i 7.6 TR1 and 7.5 TR7

As the anouncement is out some days, a lot was written about the next Technology Refreshes 
for IBM i version 7.6 (TR1) and version 7.5 (TR7).

- [IBM i 7.6 Technology Refresh 1](https://www.ibm.com/docs/en/announcements/adds-new-capabilities-enhancements-i-76-technology-refresh-1)
- [IBM i 7.5 Technology Refresh 7](https://www.ibm.com/docs/en/announcements/adds-new-capabilities-enhancements-i-75-technology-refresh-7)

As always the TRs are packed with new features, but I want to highlight one, that is my personal
favorite. 

### Named columns for the SQL `INSERT` statement

We all know the SQL `INSERT` statement to create new rows in a table. The traditional and 
standard syntax is which looks familiar since 1992:

  ```sql
  INSERT INTO mytable (column1, column2, column3)
  VALUES ('Value 1', 2, 'Value 3');
  ```

But the new TRs are adding a new - and AFAIK exclusive to Db2 for i - syntax, 
using *[named columns](https://www.ibm.com/docs/en/i/7.6.0?topic=statement-inserting-rows-using-values-clause)*:

  ```sql
  INSERT INTO mytable VALUES (
    column1 => 'Value 1',
    column2 => 2,
    column3 => 'Value 3'
  );
  ```

This syntax looks a bit like a SQL procedure oder function call with named parameters. And at the 
first look, it doesn't add any new functionality to `INSERT`. You still have to write every column
name - you still have to write every value.

But from my point of view, it's much easier to read - especially when you look which value goes 
into which column.

And if you are embedding a SQL `INSERT` statement into RPG code, it really makes it also much 
easier to maintain, as you simply can add one line for a new column, instead of have to touch 
the column list and the value list. 

  ```rpgle
  EXEC SQL INSERT INTO mytable VALUES (
             column1 => 'Value 1',
             column2 => 2,
             column3 => 'Value 3',
             column4 => 4            // added new column
           );
  ```

With this change the `INSERT` is now also more like an `UPDATE` statement, where you usually
have one line per column/value pair.

### Finally

So what do you think about the new syntax? Yes, I know, it's not SQL standard (at the moment)
but it's really so much better. And I really hope, that the SQL standard is adopting this 
syntax soon.
