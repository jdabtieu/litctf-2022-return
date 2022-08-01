# Guess The Pokemon
![](https://img.shields.io/badge/category-web-blue)
![](https://img.shields.io/badge/solves-187-orange)

## Description
Have you heard the new trending game? [GUESS THE POKEMON](http://litctf.live:31772/)!!! Please come try out our vast database of pokemons.

[guess-the-pokemon.zip](https://drive.google.com/uc?export=download&id=1_NkoqdEGrYelVcKjVOVOJ0GmlBMxyXUs)

## Solution
In the zipfile, in `main.py`, a database is initialized with the flag being stored in the `pokemon` table as its only entry.

Then, upon the user making a POST request to `/`, it attempts to run the following query against the database:
```sql
SELECT * FROM pokemon WHERE names=[USER INPUT]
```

If this query matches at least one entry, the page says that we are correct and returns the flag.

Looking at the query, there is no escaping of user input, so we can just inject an expression such that the `WHERE` clause will always be true, matching the flag.

To do this, we can send `1 or 1=1` as the input, resulting in this SQL query:
```sql
SELECT * FROM pokemon WHERE names=1 or 1=1
```

While `names` will never equal 1, 1=1 is always true, so the query will succeed.

Flag: `LITCTF{flagr3l4t3dt0pok3m0n0rsom3th1ng1dk}`
