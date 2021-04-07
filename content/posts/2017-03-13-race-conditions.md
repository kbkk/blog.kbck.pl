---
title: "Race conditions in web applications"
date: 2017-03-13T15:02:12+02:00
summary: Race conditions are a common issue in web applications. This post presents the most effective techniques to prevent them.
---

## What is a race condition?

A race condition can happen when two or more threads try to work on the same data simultaneously. As a result the data might lose integrity.

### An example

Imagine that you want to ensure every user of your web application has a unique email. So, when the users register you issue a SELECT with the provided email to a database, and if it doesn't return any results you INSERT the user. What can go wrong? Well, a race condition can. Here's the code:

```js
db.get('SELECT * FROM users WHERE email = ?', email)
    .then(result => {
        if (result)
            res.send('User with this email already exists!');
        else
            db.run(`INSERT INTO users VALUES(?, ?)`, username, email)
                .then(() => res.send(`You've registered successfully!`));
    });
```
Now, if you register an account with the `mail@example.com` email you will see: `You've registered successfully!`. 
If you do it again you should see: `User with this email already exists!`. 
However! if you fire the register requests fast enough you will actually see:

```
You've registered successfully!
You've registered successfully!
```

What just happened is a race condition.
Before the user is inserted in the first request, the second request has already done the SELECT query, and then they will both insert a record into the database.

Code for this scenario is available on my [Github](https://github.com/kbkk/race-condition-example/blob/8048800b961c24de488d5c5e623e9eb701ab4e6c/server.js).

## Preventing race conditions

### Unique index

The way to fix the example would be adding a unique index on the email field:

```sql
CREATE UNIQUE INDEX EmailUniqueIndex ON users(email)
```

Now if a race condition happens the following error will be thrown and the record won't be inserted:

```
Error: SQLITE_CONSTRAINT: UNIQUE constraint failed: users_unique.email
at Error (native) errno: 19, code: 'SQLITE_CONSTRAINT'
```

With the unique index it would also be possible to completely drop the SELECT query:

```js
db
    .run(`INSERT INTO users_unique VALUES(?, ?)`, username, email)
    .then(() => res.send(`You've registered successfully!`))
    .catch(err => {
        if (err.errno === 19 && err.code === 'SQLITE_CONSTRAINT')
            res.send('User with this email already exists!');
        else
            res.status(500).send('Something went wrong');
        console.log(err);
    });
```
However, this solution has two cons:

- every database engine returns different error codes for the unique constraint violation
- not all ORMs support this approach (as in, there may be no meaningful way to recognize the error code)

### Transactions

Transaction themselves don't prevent race conditions, and they cannot fix the example provided below.
Let's say you want to update a user's account balance and you run the following pseudocode mixed with SQL:

```sql
SELECT balance FROM accounts WHERE id = 123

-- somewhere in app code:
updatedBalance = balance + 100

UPDATE accounts SET balance = updatedBalance WHERE id = 123
```

This is how it could be fixed with a transaction:

```sql
START TRANSACTION
SELECT balance FROM accounts WHERE id = 123 FOR UPDATE

-- somewhere in app code:
updatedBalance = balance + 100

UPDATE accounts SET balance = updatedBalance WHERE id = 123
COMMIT
```

Using a transaction allows us to use the `FOR UPDATE` clause (syntax might be different among databases).
If a row is selected with `FOR UPDATE` it will be locked for other queries issued with `FOR UPDATE` unless a `COMMIT` is sent.
This is useful if the logic behind calculating the updated values is complicated and cannot be done with SQL.

### Atomic increment or decrement update

If you just need to increment or decrement a value in the database you can issue the following SQL:

```
UPDATE accounts SET balance = balance + 100 WHERE id = 123
```

This will increment the balance by 100 and is race condition safe.

## More 
There are more places where race conditions can occur and more ways of fighting them.
If you want to get more knowledge in this area you should read about isolation levels, pessimistic locking and optimistic locking.

### Bonus content (real world scenarios)
- [Printing money (DigitalOcean)](https://josipfranjkovic.blogspot.com/2015/04/race-conditions-on-facebook.html)
- [Free T-shirts (DigitalOcean again)](https://github.com/digitalocean/hacktoberfest/issues/415)
- [Killing people (Therac-25 radiation therapy machine)](https://www.bugsnag.com/blog/bug-day-race-condition-therac-25)
