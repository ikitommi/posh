# Posh

Posh is a ClojureScript / React library that lets you use a single
[DataScript](https://github.com/tonsky/datascript/) database to store
your app state. Components access the
data they need to render by calling DataScript queries with `q` or
`pull` and are only updated when the query changes. `transact!` is
used within components to change the global state. If you are familiar
with Datomic, you will find Posh incredibly easy to use. If not, it's
worth learning because of the power and versatility it will give your components.

Posh is now self-contained and can be used with multiple front-ends
(see `posh.core`), such as Reagent, Rum, or Quiescent. Only
Reagent is currently well-supported by Posh, and is the focus of this documentation.

`posh.reagent` uses [Reagent](https://github.com/reagent-project/reagent) and can be integrated with your current Reagent
project. Because it uses a single database to store app state, like [Om](https://github.com/omcljs/om) or [re-frame](https://github.com/Day8/re-frame), it is fitting to write
large, extensible apps and reusable components, with the added
benefit of being much simpler to use and having a more expressive data
retrieval and state updating syntax.

Posh is also very fast because the in-component data queries only run when the
database is updated with relevant data (found by pattern matching on the
tx report).

For example, below is a component that displays a list of a person's age, name,
and weight. The component will only re-render when something in the
database changed an attribute of the `person-id` entity:

```
(defn person [conn person-id]
  (let [p @(pull conn '[*] person-id)]
    [:ul
     [:li (:person/name p)]
     [:li (:person/age p)]
     [:li (:person/weight p)]]))
```
## Resources

Posh chat room on Gitter: https://gitter.im/mpdairy/posh

I am also currently looking for contract work or employment on a
project that uses Posh.

### Examples:

[Posh Todo List](https://github.com/mpdairy/posh-todo) - A todo list
with categories, edit boxes, checkboxes, and multi-stage delete
buttons ([trashy live demo](http://otherway.org/posh-todo/)).

### Projects Using Posh:
* [Zetawar](http://www.zetawar.com/) "A highly customizable, open
  source, turn-based, tactical strategy web game written in
  ClojureScript."
* [Datsys](https://github.com/metasoarous/datsys) A web framework.


## Usage

Start a Reagent project and include these dependencies:

```clj
[posh "0.5.2"]
```

Require in Reagent app files:
```clj
(ns example
  (:require [reagent.core :as r]
            [posh.reagent :refer [pull q posh!]]
            [datascript.core :as d]))
```

###Important changes

####0.5.1
* `get-else` now works with `q`, but still no `pull` in q.
* `q` with no `:in` args now works properly

####0.5
* You must require `posh.reagent` in your ns's instead of `posh.core`.
  This is because Posh 0.5's core is now front-end agnostic and
  Reagent is just one of the front-ends it will work with (Rum and
  Quiescent soon to come!)
* Previously, Posh's `q` took the `conn` as the first argument. Now,
  the `conn` is placed behind the query, in the args, as in DataScript
  or Datomic's `q`.
* db-tx, pull-tx, and q-tx are now deprecated. The tx-patterns generated
  for `pull` are exact and for `q` are pretty thorough.
* `q` with `get-else` and `pull` do not currently work in 0.5, though
  they sort-of worked in the older version. If you need to use those,
  just keep using the older version until those expressions are
  supported.
  
## Overview

Posh gives you two functions to retrieve data from the database from
within Reagent components: `pull` and `q`. They watch the
database's transaction report and only update (re-render) the hosting
component when one of the transacted datoms affects the requested data.

### posh!

`(posh! [DataScript conn1] ...)`

Sets up the tx-report listener for a conn.

```clj
(def conn (d/create-conn))

(posh! conn)
```

New in Posh 0.5, you can `posh!` multiple conns together if you intend
to ever use them together in a `q` query:

```clj
(posh! conn users-conn styles-conn)
```

### pull

`(pull [conn] [pull pattern] [entity id])`

`pull` retrieves the data specified in `pull-pattern` for the entity
with `entity-id`.  `pull` can be called from within any Reagent
component and will re-render the component only when the pulled
information has changed.

Posh's `pull` operates just like Datomic / Datascript's `pull` except it takes a
`conn` instead of a `db`. (See
[Datomic's pull](http://docs.datomic.com/pull.html))

Posh's `pull` only attempts to pull any new data if there has been a
transaction of any datoms that have changed the data it is
looking at. For example:

```clj
(pull conn '[:person/name :person/age] 1234)
```
Would only do a pull into Datascript if there has been a transaction
changing `:person/name` or `:person/age` for entity `1234`.

Below is an example that pulls all of the info from the entity with `id`
whenever `id` is updated and increases its age whenever clicked:

```clj
(defn pull-person [id]
  (let [p @(pull conn '[*] id)]
    (println "Person: " (:person/name p))
    [:div
     {:on-click #(transact! conn [[:db/add id :person/age (inc (:person/age p))]])}
     (:person/name p) ": " (:person/age p)]))
```
### q

`(q [query] & args)`

`q` queries for data from the database according to the datalog rules
specified in the query. It must be called within a Reagent component
and will only update the component whenever the data it is querying
has changed. See
[Datomic's Queries and Rules](http://docs.datomic.com/query.html) for
how to do datalog queries. `args` are extra variables, including the
conn or conns from which you will be querying, that DataScript's `q` looks for
after the `[:find ...]` query if the query has an `:in` specification.
Note that Posh's `q` takes conns rather than dbs.

Whenever the database has changed, `q` will check the transacted
datoms to see if anything relevant to its query has occured. If so,
`q` runs Datascript's `q` and compares the new query to the old. If it
is different, the hosting component will update with the new data.

Below is an example of a component that shows a list of people's names
who are younger than a certain age. It only attempts to re-query when
someone's age changes or a young person's name changes:

```clj
(q '[:find [?name ...]
     :in $ ?old
     :where
     [?p :person/age ?age]
     [(< ?age ?old)]
     [?p :person/name ?name]]
   conn
   old-age)
```

Currently, `pull` is not supported inside `q`. It is recommended to
query for the eids, manually send them to components with a separate
pull for each eid.

### transact!

`posh.reagent`'s `transact!` currently just calls DataScript's `transact!`:

```clj
(transact! conn [[:db/add 123 :person/name "Jim"]])
```

### posh-atom

`(get-posh-atom conn)`

The cache of all the queries is stored inside the posh-atom, which is
pointed to by the conn. If you want to see or edit things "under the
hood", this is where to go. The dereffed posh-atom can be used as the
`posh-tree` in the functions in `posh.core`. A wiki will
one day explain further.

## Advanced Examples

### Editable Label

This component will show the text value
for any entity and attrib combo. There is an "edit" button that, when clicked, 
creates an `:edit` entity that keeps track of the
temporary text typed in the edit box. The "done" button resets the original
value of the entity and attrib and deletes the `:edit` entity. The
"cancel" button just deletes the `:edit` entity.

The state is stored entirely in the database for this solution, so if
you were to save the db during the middle of an edit, if you restored
it later, you would be in the middle of the edit still.

```clj
(defn edit-box [conn edit-id id attr]
  (let [edit @(p/pull conn [:edit/val] edit-id)]
    [:span
     [:input
      {:type "text"
       :value (:edit/val edit)
       :onChange #(p/transact! conn [[:db/add edit-id :edit/val (-> % .-target .-value)]])}]
     [:button
      {:onClick #(p/transact! conn [[:db/add id attr (:edit/val edit)]
                                    [:db.fn/retractEntity edit-id]])}
      "Done"]
     [:button
      {:onClick #(p/transact! conn [[:db.fn/retractEntity edit-id]])}
      "Cancel"]]))

(defn editable-label [conn id attr]
  (let [val  (attr @(p/pull conn [attr] id))
        edit @(p/q conn '[:find ?edit .
                          :in $ ?id ?attr
                          :where
                          [?edit :edit/id ?id]
                          [?edit :edit/attr ?attr]]
                   id attr)]
    (if-not edit
      [:span val
       [:button
        {:onClick #(new-entity! conn {:edit/id id :edit/val val :edit/attr attr})}
        "Edit"]]
      [edit-box conn edit id attr])))

```

This can be called with any entity and its text attrib, like
`[editable-label conn 123 :person/name]` or
`[editable-label conn 432 :category/title]`.

## Back-end

As of version 0.5, `posh.core` should be able to run on Datomic
databases and keep track of all queries. It can also generate, for any
`q` or `pull`, the "necessary datoms" needed in a bare database to get
the same result for that `q` or `pull`, which means that the front-end
can send its graph of queries to the backend and get back any datoms
needed to update its db whenever anything relevant changes.

[Datsync](https://github.com/metasoarous/datsync) is a utility that
eventually will do this, though currently it just copies the entire
Datomic db over to DataScript.

See our Gitter room for updates: https://gitter.im/mpdairy/posh

## License

Copyright © 2015 Matt Parker

If somebody needs to BSD then sure, it's under that too.
Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
