# Gameing Model Demo

## Overview

This demo demonstrates a modeling excercise.
The goal is to come up with a usable document structure in MongoDB or compatible database which fulfils the requirements in a document-oriented way.

## Scenario Description



## Implementation

Set some global variables.

```javascript
gameId = ObjectId("111122223333444455556666");

var now = new Date();
var later = new Date();
later.setDate(now.getDate() + 1);
```

Setup a game. Seed the database with some data.

```javascript
function createGame(){
  db.game.drop();

  db.gameSetup.replaceOne({ "_id": gameId },
    {
      "name": "forest quest",
      "start": ISODate(),
      "maxDuration": "10",
      "players": ["bob", "kim", "ogg"],
      "trophies":
        [
          { "Aye-aye": [-95.85518, 26.91995] },
          { "Xenops": [-6.06384, 37.13601] },
          { "Whydah": [-27.54989, 74.70806] },
          { "Quoll": [140.43106, 24.1615] },
          { "Pangolin": [45.12271, -71.49936] },
          { "Basilisk": [97.9106, -86.62228] },
          { "Agouti": [-161.59849, -7.89993] }
        ]
    })

  db.game.replaceOne({ "_id": gameId },
    {
      "name": "forest quest",
      "players": {
        "bob": { "trophies": [], points: 0 },
        "kim": { "trophies": [], points: 0 },
        "ogg": { "trophies": [], points: 0 },
      },
      "trophies": ["Aye-aye", "Xenops", "Whydah", "Quoll", "Pangolin", "Basilisk", "Agouti"],
      "deadline": dt
    }, { upsert: true });
}
```

Create and index for easy end-game query

```javascript
db.game.createIndex({ _id: 1, deadline: 1, trophies:1 })
```


Game Interaction 1: Clame a trophy

```javascript
// claim a trophy
function claimTrophy(id, gamer) {
  TROPHY = db.game.findOne({ "_id": id }).trophies.pop()
  print(`Claiming ${TROPHY}`)

  return db.game.updateOne(
    { "_id": id, "deadline": { "$gt": ISODate() }, "trophies": TROPHY },
    {
      "$inc": { [`players.${gamer}.points`]: 3 },
      "$push": { [`players.${gamer}.trophies`]: TROPHY },
      "$pull": { "trophies": TROPHY }
    }
  )
}
```

Game Interaction 2: Check if game-over

```javascript
// is game over?
function isGameOver(gameId, now= ISODate()) {
  result = db.game.aggregate([
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

Claiming a trophy: supply the game id and the gamer id.

```javascript
claimTrophy(gameId, 'bob');
claimTrophy(gameId, 'kim');
claimTrophy(gameId, 'kim');
claimTrophy(gameId, 'ogg');
claimTrophy(gameId, 'bob');
```

Checking for game-over condition:

More than one way to do this.
An explicit query can be issued periodically:

```javascript
isGameOver(gameId);
```

It can also be checked using change-streams. A listener on the change stream can project the `trophies` array and game `deadline`. similar to the game-over conditions in the explicity query. But by filtering on that event, a listener knows the game is over when the event fires. Then all that remains is to broadcast that to the interested parties (players, game app).

## Playing Past Deadline?

The method of trophy-claim ensures that players may not overrun the deadline.

```javascript
// attempt to claim past game ending deadline
db.game.findOne({ "_id": gameId }, { deadline: 1 })
// set deadline to the past
db.game.updateOne({ "_id": gameId }, { "$set": { deadline: ISODate('1970-01-01') } });
// ensure deadline is way in the past
print( db.game.findOne({ "_id": gameId }, { deadline: 1 }))
// attempt a claim. Should not match.
claimTrophy();
```
