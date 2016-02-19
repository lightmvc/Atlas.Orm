# Atlas.Orm

> No annotations. No migrations. No lazy loading. No data-type abstractions.

Atlas is a [data mapper](http://martinfowler.com/eaaCatalog/dataMapper.html)
implementation for your **persistence model** (*not* your domain model).

As such, Atlas uses the term "record" to indicate that its objects are *not*
domain entities. Note that an Atlas record is a *passive* record, not an [active
record](http://martinfowler.com/eaaCatalog/activeRecord.html); it is
disconnected from the database. Use Atlas records indirectly to populate your
domain entities, or directly for simple data source interactions.

**ATLAS IS A WORK IN PROGRESS. FOR ENTERTAINMENT PURPOSES ONLY. DO NOT USE IN
PRODUCTION. BREAKING CHANGES ARE GUARANTEED.** Having said that, Atlas is ready
for experimental and side-project use. Please send bug reports!

## Rationale

I wanted an alternative to Active Record that would allow you to get started
about as easily as Active Record for your *persistence* model, and then refactor
more easily towards a richer *domain* model as needed.

Using a data-mapper for the underlying table Rows, then composing them into
Records and RecordSets, does the trick. As you begin to need simple behaviors,
you can add them to the Record and RecordSet persistence model objects. (Rows
do not have behavior.) Your domain logic layer (e.g. a Service Layer) can then
use them as needed.

However, per [this article from Mehdi Khalili][mkap], the target end-state for
your modeling should eventually move toward "Domain Model composed of
Persistence Model". That is, the domain Entity and Aggregate classes might use
data source Records and RecordSets internally, but will not expose them. They
can manipulate the persistence model objects internally as much as they wish.
E.g., an Entity might have "getAddress()" and read from the internal Record
(which in turn reads from its internal Row or Related objects).  Alternatively,
the end state might be "DDD on top of ORM" where Repositories map the
persistence model objects to domain Entities, Value Objects, and Aggregates.

A persistence model alone should get you a long way, especially at the beginning
of a project. Even so, the data-mapped Row, Record, and RecordSet objects are
disconnected from the database, which should make the refactoring process a lot
cleaner than with Active Record.

[mkap]: http://www.mehdi-khalili.com/orm-anti-patterns-part-4-persistence-domain-model/

Other rationalizations, essentially based around things I *do not* want in an
ORM:

- No annotations. I want the code to be in code, not in comments.

- No migrations or other table-modification logic. Many ORMs read the PHP objects
and then create or modify tables from them. I want the persistence system to be
a *model* of the schema, not a *creator* of it. If I need a migration, I'll use
a tool specifically for migrations.

- No lazy-loading. Lazy-loading is seductive but eventually is more trouble than
it's worth; I don't want it to be available at all, so that it cannot accidently
be invoked.

- No data-type abstractions. I used to think data-type abstraction was great,
but it turns out to be another thing that's just not worth the cost. I want the
actual underlying database types to be exposed and available as much as
possible.

Possible deal-breakers for potential users:

- Atlas uses code generation, though only in a very limited way. I'm not a fan
of code generation myself, but it turns out to be useful for building the SQL
table classes. Each table is described as a PHP class, one that just returns
things like the table name, the column names, etc. That's the only class that
really gets generated by Atlas; the others are just empty extensions of parent
classes.

- Atlas uses base Row and Record classes, instead of plain-old PHP objects. If
this were a domain modelling system, a base class would be unacceptable. Because
Atlas is a *persistence* modelling system, I think a base class is less
objectionable, but for some people that's going to be a real problem.

Finally, Atlas supports **composite primary keys** and **composite foreign keys.**
Performance in these cases is sure to be slower, but it is in fact supported.


## Installation

This package is installable and autoloadable via [Composer](https://getcomposer.org/)
as [atlas/orm](https://packagist.org/packages/atlas/orm).

Make sure your project it set up to [autoload Composer-installed packages](https://getcomposer.org/doc/00-intro.md#autoloading).


## Basic Usage

> This section is sorely incomplete.

### Creating Classes

You can create your data source classes by hand, but it's going to be tedious to
do so. Instead, use the `atlas-skeleton` command to read the table information
from the database.

Create a PHP file to return an array of connection parameters suitable for PDO:

```php
<?php
// /path/to/conn.php
return ['mysql:dbname=testdb;host=localhost', 'username', 'password'];
```

You can then invoke `atlas-skeleton` using that connection and a table name.
Specify a target directory for the skeleton files if you like, and pass the
namespace name for the data source classes.

```bash
./bin/atlas-skeleton.php \
    --dir=./src/App/DataSource \
    --conn=/path/to/conn.php \
    --table=threads \
    App\\DataSource\\Thread
```

> N.b.: Calling `atlas-skeleton` with `--conn` and `--table` will overwrite any
> existing Table class; this makes sense only because the Table class represents
> the table description at the database. No other existing files will ever be
> overwritten.

That will create this subdirectory and these classes in `./src/App/DataSource/`:

    └── Thread
        ├── ThreadMapper.php
        └── ThreadTable.php

The Mapper class will be essentially empty, and the Table class will contain a
description of the database table.

Do that once for each SQL table in your database.

> N.b.: By default, Atlas uses generic Record and RecordSet classes for
> table data. You can create custom Record and RecordSet classes passing
> `--full` to `atlas-skeleton`; the Mapper will use the custom classes if
> available, and fall back to the generic ones if not. (Custom Row classes are
> not available, and probably not desirable.)

### Relationships

You can add relationships by editing the _Mapper_ class:

```php
<?php
namespace Atlas\DataSource\Thread;

use App\DataSource\Author\AuthorMapper;
use App\DataSource\Summary\SummaryMapper;
use App\DataSource\Reply\ReplyMapper;
use App\DataSource\Tagging\TaggingMapper;
use App\DataSource\Tag\TagMapper;
use Atlas\Orm\Mapper\Mapper;

class ThreadMapper extends AbstractMapper
{
    protected function setRelated()
    {
        $this->manyToOne('author', AuthorMapper::CLASS);
        $this->oneToOne('summary', SummaryMapper::CLASS);
        $this->oneToMany('replies', ReplyMapper::CLASS);
        $this->oneToMany('taggings', TaggingMapper::CLASS);
        $this->manyToMany('tags', TagMapper::CLASS, 'taggings');
    }
}
```

By default, in all relationships except many-to-one, the relationship will take
the primary key column(s) in the native table, and map to those same column
names in the foreign table. In the case of many-to-one, it is the reverse; that
is, the relationship will take the primary key column(s) in the foreign table,
and map to those same column names in the native table.

If you want to change the mappings, use the `on()` method on the relationship.
For example, if the threads table uses `author_id`, but the authors table uses
just `id`, you can do this:

```php
<?php
$this->oneToOne('author', AuthorMapper::CLASS)
    ->on([
        // native (threads) column => foreign (authors) column
        'author_id' => 'id',
    ]);
```

Likewise, if a table uses a composite key, you can re-map the relationship on
multiple columns. If table `foo` has composite primary key columns of `acol` and
`bcol`, and it maps to table `bar` on `foo_acol` and `foo_bcol`, you would do
this:

```php
<?php
class FooMapper
{
    protected function setRelated()
    {
        $this->oneToMany('bars', BarMapper::CLASS)
            ->on([
                // native (foo) column => foreign (bar) column
                'acol' => 'foo_acol',
                'bcol' => 'foo_bcol',
            ]);
    }
}
```

### Reading

Create an _Atlas_ instance using the _AtlasContainer_, and provide the default
_ExtendedPdo_ connection parameters:

```php
<?php
$atlasContainer = new AtlasContainer(
    'mysql:host=localhost;dbname=testdb',
    'username',
    'password'
);
```

Next, set the available mapper classes, and get back an _Atlas_ instance:

```php
<?php
$atlasContainer->setMappers([
    AuthorMapper::CLASS,
    ReplyMapper::CLASS,
    SummaryMapper::CLASS,
    TagMapper::CLASS,
    ThreadMapper::CLASS,
    TaggingMapper::CLASS,
]);

$atlas = $atlasContainer->getAtlas();
```

You can then use Atlas to select a Record or a RecordSet:

```php
<?php
// fetch thread_id 1; with related replies, including each reply author
$threadRecord = $atlas->fetchRecord(ThreadMapper::CLASS, '1', [
    'author',
    'summary',
    'replies' => function ($select) {
        $select->with(['author']);
    },
    'taggings',
    'tags',
]);

// fetch thread_id 1, 2, and 3; with related replies, including each reply author
$threadRecordSet = $atlas->fetchRecordSet(ThreadMapper::CLASS, [1, 2, 3], [
    'author',
    'summary',
    'replies' => function ($select) {
        $select->with(['author']);
    },
    'taggings',
    'tags',
]);

// a more complex select of only the last 10 threads, with only some relateds
$threadRecordSet = $atlas
    ->select(ThreadMapper::CLASS)
    ->orderBy('thread_id DESC')
    ->limit(10)
    ->with([
        'author',
        'summary'
    ])
    ->fetchRecordSet();
```

If you do not load a Record "with" a related, it will be `null` in the
Record, and it will not be lazy-loaded for you later. This means you need to
think ahead as to exactly what you will need from the database.

You can then address the Record's underlying Row columns and the related
fields as properties.

```php
<?php
echo $thread->title;
echo $thread->body;
foreach ($thread->replies as $reply) {
    echo $reply->author->name;
    echo $reply->body;
}
```

Incidentally, you can also use the mapper to select non-Record values directly
from the database; the mapper selection tool exposes the underlying
`ExtendedPdo::fetch*()` methods for your convenience.

```php
<?php
// an array of IDs
$threadIds = $atlas
    ->select(ThreadMapper::CLASS)
    ->cols(['thread_id'])
    ->limit(10)
    ->orderBy('thread_id DESC')
    ->fetchCol();

// key-value pairs of IDs and titles
$threadIdsAndTitles = $atlas
    ->select(ThreadMapper::CLASS)
    ->cols(['thread_id', 'tite'])
    ->limit(10)
    ->orderBy('thread_id DESC')
    ->fetchPairs();

// etc.
```

See [the list of `ExtendedPdo::fetch*()` methods][fetch] for more.

[fetch]: https://github.com/auraphp/Aura.Sql#new-fetch-methods


### Changing

Make changes to the Record by setting new property values.

```php
<?php
$thread = $atlas->newRecord(ThreadMapper::CLASS);
$thread->title = "Thread title";
$thread->body = "Body text for the thread";
```

Note that the Row supporting each Record is identity-mapped, so a change to
a Row used by more than one Record will be reflected immediately in each
Record using that Row.

 ```php
<?php
// $reply1 and $reply2 are two different replies by the same author. the reply
// rows are different, but the underlying author row is the same.
$reply1->author->name = "New name"; // $reply2->author->name is now also "New name"
```

### Writing

After you make changes to a Record, you can write it back to the database
using a unit-of-work _Transaction_. You can plan for Records to be inserted,
updated, and deleted, in whatever order you like, and then execute the entire
transaction plan at once. Exceptions will cause a rollback.

```php
<?php
// create a transaction
$transaction = $atlas->newTransaction();

// plan work for the transaction
$transaction->insert($record1);
$transaction->update($record2);
$transaction->delete($record3);

// execute the transaction plan
$ok = $transaction->exec();
if ($ok) {
    echo "Transaction success.";
} else {
    // get the exception that was thrown in the transaction
    $exception = $transaction->getException();
    // get the work element that threw the exception
    $work = $transaction->getFailure();
    // some output
    echo "Transaction failure. ";
    echo $work->getLabel() . ' threw ' . $exception->getMessage();
}
```
