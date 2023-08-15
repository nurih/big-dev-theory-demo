# Gameing Model Demo

## Overview

This demo demonstrates a modeling excercise.

The goal is to come up with a usable document structure in MongoDB or compatible database which fulfils the requirements in a document-oriented way.

## Scenario Description

A multi player game backend.

- Game has a set number of trophies
- Thousands of concurrent games
- Up to 10 players concurrent players
- Game has time limit
- Game ends when all trophies claimed
- Winner is player with most trophies claimed
- Tie declares all players with said number of trophies as winners

## Implementation

Set some global variables.

```javascript
db.game.drop();

db.gamePlay.drop();

gameId = ObjectId("111122223333444455556666");

var now = new Date();
var deadline = new Date();
deadline.setDate(now.getDate() + 1);
```

### Game Setup

Seed the database with some data.

```javascript
function createGame(name,gameId,start,deadline){

  const r1 = db.game.replaceOne({ "_id": name},
    {
      "start": start,
      "maxDuration": "10",
      "players": ["bob", "kim", "ogg"],
      "trophies":
        [
          { "trophy": "Aye-aye", "loc": {"type":"Point", "coordinates": [ -95.8551,  26.9199]}},
          { "trophy": "Xenops",  "loc": {"type":"Point", "coordinates": [  -6.0638,  37.1360]}},
          { "trophy": "Whydah",  "loc": {"type":"Point", "coordinates": [ -27.5498,  74.7080]}},
          { "trophy": "Quoll",   "loc": {"type":"Point", "coordinates": [ 140.4310,  24.1615]}},
          { "trophy": "Pangolin","loc": {"type":"Point", "coordinates": [  45.1227, -71.4993]}},
          { "trophy": "Basilisk","loc": {"type":"Point", "coordinates": [  97.9106, -86.6222]}},
          { "trophy": "Agouti",  "loc": {"type":"Point", "coordinates": [-161.5984,  -7.8999]} } 
        ]
    }, {upsert:true});

  const r2 = db.gamePlay.replaceOne({ "_id": gameId },
    {
      "name": "forest quest",
      "players": {
        "bob": { "trophies": [], points: 0 },
        "kim": { "trophies": [], points: 0 },
        "ogg": { "trophies": [], points: 0 },
      },
      "trophies": ["Aye-aye", "Xenops", "Whydah", "Quoll", "Pangolin", "Basilisk", "Agouti"],
      "start": start,
      "deadline": deadline
    }, { upsert: true });

    return [r1,r2];
}
```

With this, we can run: `createGame("forest quest", gameId, now, deadline)`

### Indexing

Create and index for efficient end-game query

```javascript
db.gamePlay.createIndex({ _id: 1, deadline: 1, trophies:1 })
```

### Play - Claim a Trophy

Claiming a trophy is an operation which assignes a trophy to a player, and awards points while doing it.
The operation is atomic, so no two players can claim the same trophy.

A _trophy_ is claimed against a certain _game id_, by a certain _gamer_. The claim time is implicitely "now".

```javascript

function claimTrophy(gameId, gamer, trophy) {
  return db.gamePlay.updateOne(
    { "_id": gameId, "deadline": { "$gt": ISODate() }, "trophies": trophy },
    {
      "$inc": { [`players.${gamer}.points`]: 3 },
      "$push": { [`players.${gamer}.trophies`]: trophy },
      "$pull": { "trophies": trophy }
    }
  )
}
```

> Claiming a trophy only succeeds if
>
> 1. The game by the _id_ exists
> 1. The trophy is yet to be claimed
> 1. The game deadline has not passed

Claiming a trophy: supply the game id and the gamer id, and a trophy.

> Gamers: [`bob`,`kim`,`ogg`]
> Trophies: [`Aye-aye`, `Xenops`, `Whydah`, `Quoll`, `Pangolin`, `Basilisk`, `Agouti`]

```javascript
claimTrophy(gameId, 'bob', 'Quoll');
```


### Play - Game Over?

Checking if the game is over is a read operation, and is processed by the game engine. That is - the game data does not require mutation.

```javascript
function isGameOver(gameId, now= ISODate()) {
  result = db.gamePlay.aggregate([
    { $match: { "_id": gameId }},
    {
      $project: {
        trophies_remaining: { $size: '$trophies' },
        all_trophies_claimed: { $cond: { if: { $eq: [{ $size: '$trophies' }, 0] }, then: true, else: false } },
        deadline_expired: { $cond: { if: { $gt: [now, '$deadline'] }, then: true, else: false } },
      }
    }
  ]);

  return result;
}
```

Checking for game-over condition:

```javascript
isGameOver(gameId);
```

There is more than one way to do this. An explicit query can be issued periodically. Game API probably exposes a "hub" method which is game-aware. It is not efficient for each gamer to poll repeatedly.

It can also be checked using change-streams. A listener on the change stream can project the `trophies` array and game `deadline`. Similar to the game-over conditions in the explicity query. But by filtering on that event, a listener knows the game is over when the event fires. Then all that remains is to broadcast that to the interested parties (players, game app).

### Cheater Beware

The trophy claiming method ensures that:

1. Player can't overrun the deadline.
1. Player can't claim a previously claimed trophy
1. Player can't claim a non-existant trophy

```javascript

whatId = ObjectId();

var now = new Date();
var deadline = new Date();
deadline.setDate(now.getDate() +1 );

print([now,deadline]);

createGame("roughshot", whatId, now, deadline);
```

With the _roughshot_ game, try to claim same trophy twice:

```javascript
// attempt to claim past game ending deadline
db.gamePlay.findOne({ "_id": whatId }, { deadline: 1 , trophies:1})

claimTrophy(whatId,'bob','Aye-aye');
// ^^^ should succeed: modifiedCount = 1

claimTrophy(whatId,'bob','Aye-aye');
// ^^^ should fail: modifiedCount = 0
```

With the _roughshot_ game, try to claim non-existing trophy:

```javascript
// attempt to claim past game ending deadline
db.gamePlay.findOne({ "_id": whatId }, { deadline: 1 , trophies:1})

// attempt a claim. Should not match.
claimTrophy(whatId,'bob','MadeUpTrophy');

```



With the _roughshot_ game, try to overrun the clock:

```javascript
// attempt to claim past game ending deadline
db.gamePlay.findOne({ "_id": whatId }, { deadline: 1 , trophies:1})
// set deadline to the past
db.gamePlay.updateOne({ "_id": whatId }, { "$set": { deadline: ISODate('1970-01-01') } });
// ensure deadline is way in the past
print( db.gamePlay.findOne({ "_id": whatId }, { deadline: 1 }))
// attempt a claim. Should not match.
claimTrophy(whatId,'bob','Basilisk');
```

## Scale-Out

The `game` collection contains prototype documents. These change at low frequency, and are not anticipated to grow too much.

`gamePlay` documents on the other hand, are expected to grow numerous, and experience high write loads. With tens of thousands of people playing concurrently, we want to distribute the load. This is acheivable by crafting a shard key that includes the `_id` of the `gamePlay` collection in it. Since _ObjectId_ is a monotonously increasing kind of critter, stating it as `hashed` is going to ensure even distribution of gamePlay documents across a sharded cluster. The `gamePlay`'s `_id` is known at runtime, and is omnipresent in every interaction. Therefore the access to that single document (per game) would be very efficient, and enroll exactly one shard. Thoug the occasional query might exist to join against the `game` collection or theoretically the gamer list (not part of this scenario), care should be taken to use a proper filter and indexing strategy to minimise impact this rarity may impose.
