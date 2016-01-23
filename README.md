# GraphAware Reco4PHP

## Neo4j based Recommendation Engine Framework for PHP

GraphAware Reco4PHP is a library for building complex recommendation engines atop Neo4j.

Features:

* Clean and flexible design
* Built-in algorithms and functions
* Ability to measure recommendation quality
* Built-in Cypher transaction management

The library imposes a specific recommendation engine architecture, which has emerged from our experience building recommendation
engines and solves the architectural challenge to run recommendation engines remotely via Cypher.
In return it handles all the plumbing so that you only write the recommendation business logic specific to your use case.

### Recommendation Engine Architecture

#### Discovery Engines and Recommendations

The purpose of a recommendation engine is to `recommend` something, should be users you should follow, products you should buy,
articles you should read.

The first part in the recommendation process is to find items to recommend, it is called the `discovery` process.

In Reco4PHP, a `DiscoveryEngine` is responsible for discovering items to recommend in one possible way.

Generally, recommender systems will contains multiple discovery engines, if you would write the `who you should follow on github` recommendation engine,
you might end up with the non-exhaustive list of `Discovery Engines` :

* Find people that contributed on the same repositories than me
* Find people that `FOLLOWS` the same people I follow
* Find people that `WATCH` the same repositories I'm watching
* ...

Each `Discovery Engine` will produce a set of `Recommendations` which contains the discovered `Item` as well as the score for this item (more below).

#### Filters and BlackLists

The purpose of `Filters` is to compare the original `input` to the `discovered` item and decide whether or not this item should be recommended to the user.
A very straightforward filter could be `ExcludeSelf` which would exclude the item if it is the same node as the input, which can relatively happen in a densely connected graph.

`BlackLists` on the other hand are a set of predefined nodes that should not be recommended to the user. An example could be to create a `BlackList` with the already purchased items
by the user if you would recommend him products he should buy.

#### PostProcessors

`PostProcessors` are providing the ability to post process the recommendation after it has passed the filters and blacklisting process.

For example, if you would reward a recommended person if he/she lives in the same city than you, it wouldn't make sense to load all people from the database that live
in this city in the discovery phase (this could be millions if you take London as an example).

You would then create a `RewardSameCity` post processor that would adapt the score of the produced recommendation if the input node and the recommended item are living in the same city.

#### Summary

To summarize, a typical recommendation engine will be a set of :

* one or more `Discovery Engines`
* zero or more `Fitlers` and `BlackLists`
* zero or more `PostProcessors`

Let's start it !


#### Usage by example

We will use the small dataset available from MovieLens containing movies, users and ratings as well as genres.

The dataset is publicly available here : http://grouplens.org/datasets/movielens/

Once downloaded and extracted the archive, you can run the following Cypher statements for importing the dataset, just adapt the file urls to match your actual path to the files :

```
CREATE CONSTRAINT ON (m:Movie) ASSERT m.id IS UNIQUE;
CREATE CONSTRAINT ON (g:Genre) ASSERT g.name IS UNIQUE;
CREATE CONSTRAINT ON (u:User) ASSERT u.id IS UNIQUE;
```

```
LOAD CSV WITH HEADERS FROM "file:///Users/ikwattro/dev/movielens/movies.csv" AS row
WITH row
MERGE (movie:Movie {id: toInt(row.movieId)})
ON CREATE SET movie.title = row.title
WITH movie, row
UNWIND split(row.genres, '|') as genre
MERGE (g:Genre {name: genre})
MERGE (movie)-[:HAS_GENRE]->(g)
```


```
USING PERIODIC COMMIT 500
LOAD CSV WITH HEADERS FROM "file:///Users/ikwattro/dev/movielens/ratings.csv" AS row
WITH row
MATCH (movie:Movie {id: toInt(row.movieId)})
MERGE (user:User {id: toInt(row.userId)})
MERGE (user)-[r:RATED]->(movie)
ON CREATE SET r.rating = toInt(row.rating), r.timestamp = toInt(row.timestamp)
```

For the purpose of the example, we will assume we are recommending movies for the User with ID 460.


### Installation

Require the dependency with `composer` :

```bash
composer require graphaware/reco4php
```

### Usage

#### Discovery

In order to recommend movies people should watch, you have decided that we should find potential recommendations in the following way :

* Find movies rated by people who rated the same movies than me, but that I didn't rated yet

As told before, the `reco4php` recommendation engine framework makes all the plumbing so you only have to concentrate on the business logic, that's why it provides base class that you should extend and just implement
the methods of the upper interfaces, here is how you would create your first discovery engine :

```php
<?php

namespace GraphAware\Reco4PHP\Tests\Example;

use GraphAware\Reco4PHP\Engine\SingleDiscoveryEngine;

class RatedByOthers extends SingleDiscoveryEngine
{
    public function query()
    {
        $query = "MATCH (input)-[:RATED]->(m)<-[:RATED]-(other)
        WITH distinct other
        MATCH (other)-[r:RATED]->(reco)
        WITH distinct reco, sum(r.rating) as score
        ORDER BY score DESC
        RETURN reco, score LIMIT 500";

        return $query;
    }

    public function name()
    {
        return "rated_by_others";
    }

}
```

The `input` node is implicitly matched by the underlying query executor, so you don't have to write the query for matching the input node everytime. So basically it is doing for you `MATCH (input) WHERE id(input) = {idInput}`;

The `query` method should return a string containing the query for finding recommendations, the `name` method should return a string describing the name of your engine (this is mostly for logging purposes).

The query here has some logic, we don't want to return as candidates all the movies found, as in the initial dataset it would be 10k+, so imagine what it would be on a 100M dataset. So we are summing the score
of the ratings and returning the most rated ones, limit the results to 500 potential recommendations.



### License

This library is released under the Apache v2 License, please read the attached `LICENSE` file.