---

title: Many-to-many relationships
description: An example that models a shopping cart system with drift.

---



Since drift is a relational database library and not an ORM, it doesn't automatically
fetch relationships between entities for you. Instead, it gives you the tool to
manually write the joins needed to express more complex queries efficiently.

This example shows how to do that with a complex many-to-many relationship by
implementing a database for an online shop. In particular, we'll focus on
how to model shopping carts in SQL.
Here, there is a many-to-many relationship between shopping carts and products:
A product can be in many shopping carts at the same time, and carts can of course
contain more than one product too.

In sqlite3, there are two good ways to model many-to-many relationships
between tables:

1. The traditional way of using a third table storing every combination of
   products and carts.
2. A more modern way might be to store product IDs in a shopping cart as a JSON
   array.

The two approaches have different upsides and downsides. With the traditional
relational way, it's easier to ensure data integrity (by, for instance, deleting
product references out of shopping carts when a product is deleted).
On the other hand, queries are easier to write with JSON structures. Especially
when the order of products in the shopping cart is important as well, a JSON
list is very helpful since rows in a table are unordered.

Picking the right approach is a design decision you'll have to make. This page
describes both approaches and highlights some differences between them.

## Common setup

In both approaches, we'll implement a repository for shopping cart entries that
will adhere to the following interface:

{{ load_snippet('interface','lib/snippets/modular/many_to_many/shared.dart.excerpt.json') }}

We also need a table for products that can be bought:

{{ load_snippet('buyable_items','lib/snippets/modular/many_to_many/shared.dart.excerpt.json') }}

## In a relational structure



### Defining the model

We're going to define two tables for shopping carts: One for the cart
itself, and another one to store the entries in the cart.
The latter uses [references](../dart_api/tables.md#references)
to express the foreign key constraints of referencing existing shopping
carts or product items.

{{ load_snippet('cart_tables','lib/snippets/modular/many_to_many/relational.dart.excerpt.json') }}

### Inserts
We want to write a `CartWithItems` instance into the database. We assume that
all the `BuyableItem`s included already exist in the database (we could store
them via `into(buyableItems).insert(BuyableItemsCompanion(...))`). Then,
we can replace a full cart with

{{ load_snippet('updateCart','lib/snippets/modular/many_to_many/relational.dart.excerpt.json') }}

We could also define a helpful method to create a new, empty shopping cart:

{{ load_snippet('createEmptyCart','lib/snippets/modular/many_to_many/relational.dart.excerpt.json') }}

### Selecting a cart
As our `CartWithItems` class consists of multiple components that are separated in the
database (information about the cart, and information about the added items), we'll have
to merge two streams together. The `rxdart` library helps here by providing the
`combineLatest2` method, allowing us to write

{{ load_snippet('watchCart','lib/snippets/modular/many_to_many/relational.dart.excerpt.json') }}

### Selecting all carts
Instead of watching a single cart and all associated entries, we
now watch all carts and load all entries for each cart. For this
type of transformation, RxDart's `switchMap` comes in handy:

{{ load_snippet('watchAllCarts','lib/snippets/modular/many_to_many/relational.dart.excerpt.json') }}

## With JSON functions

This time, we can store items directly in the shopping cart table. Multiple
entries are stored in a single row by encoding them into a JSON array, which
happens with help of the `json_serializable` package:



{{ load_snippet('tables','lib/snippets/modular/many_to_many/json.dart.excerpt.json') }}

Creating shopping carts looks just like in the relational example:

{{ load_snippet('createEmptyCart','lib/snippets/modular/many_to_many/json.dart.excerpt.json') }}

However, updating a shopping cart doesn't require a transaction anymore since it can all happen
in a single table:

{{ load_snippet('updateCart','lib/snippets/modular/many_to_many/json.dart.excerpt.json') }}

To select a single cart, we can use the [`json_each`](https://sqlite.org/json1.html#jeach)
function from sqlite3 to "join" each item stored in the JSON array as if it were a separate
row. That way, we can efficiently look up all items in a cart:

{{ load_snippet('watchCart','lib/snippets/modular/many_to_many/json.dart.excerpt.json') }}

Watching all carts isn't that much harder, we just remove the `where` clause and
combine all rows into a map from carts to their items:

{{ load_snippet('watchAllCarts','lib/snippets/modular/many_to_many/json.dart.excerpt.json') }}
