## Named columns for the SQL `INSERT` statement

As the anouncement is out some days, a lot was written about the next Technology Refreshes 
for IBM i version 7.6 (TR1) and version 7.5 (TR7).

- [IBM i 7.6 Technology Refresh 1](https://www.ibm.com/docs/en/announcements/adds-new-capabilities-enhancements-i-76-technology-refresh-1)
- [IBM i 7.5 Technology Refresh 7](https://www.ibm.com/docs/en/announcements/adds-new-capabilities-enhancements-i-75-technology-refresh-7)

As always the TRs are packed with new features, but I want to highlight one, that is my personal
favorite.

The SQL `INSERT` statement gets a new *named columns* syntax for the `VALUES` clause. But before we
have a look at this, let's look at the syntax of the `INSERT` statement as we know it.

### The *traditional* `INSERT` statement syntax

The SQL `INSERT` statements purpose is to create new rows in a table. The traditional and standard 
syntax is which looks familiar since 1992:

```sql
insert into mytable (column1, column2, column3)
values ('Value 1', 2, 'Value 3');
```

If you don't want to supply values for every column of the table, you list the table columns in
parentheses and the values in `VALUES` clause - exactly in the same order. 

If you have lot of columns and a lot of values, this is a common source for errors. And here
the new syntax, that is introduced with the new TRs, will be a huge improvement.

### The new `INSERT` *named columns* syntax

You can read about the new syntax here: [Inserting rows using the values clause](https://www.ibm.com/docs/en/i/7.6.0?topic=statement-inserting-rows-using-values-clause)

But let's look at a short example first:

```sql
insert into mytable values (
  column1 => 'Value 1',
  column2 => 2,
  column3 => 'Value 3'
);
```

This syntax looks a bit like a SQL procedure oder function call with named parameters. 
You don't list the columns after the table name. Instead you use the a construct like 
`column1 => value1` inside the `VALUES` clause to bind the value directly to a column name. 

And at first look, it doesn't add any new functionality to `INSERT`. You still have to 
write every column name - you still have to write every value. But from my point of view, 
it's much easier to read - especially when you look which value goes into which column.

And if you are embedding a SQL `INSERT` statement into RPG code, it really makes it also much 
easier to maintain, as you simply can add one line for a new column, instead of have to touch 
the column list and the value list. 

```rpgle
exec sql insert into mytable values (
           column1 => 'Value 1',
           column2 => 2,
           column3 => 'Value 3',
           column4 => 4            // added new column
         );
```

With this change the `INSERT` now looks more like an `UPDATE` statement, where you usually
have one line per column/value pair.

### Finally

So what do you think about the new syntax? Yes, I know - it's not SQL standard (at the moment)
but it's really so much better to read. And I really hope, that the SQL standard is adopting 
this syntax soon. 

Will you adopt it?
