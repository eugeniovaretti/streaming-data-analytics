# Bocce EPL problem
## Resources

- [espertech](https://www.espertech.com/)
- [EPL documentation](http://esper.espertech.com/release-8.7.0/reference-esper/html/index.html)
- [EPL playground](http://esper-epl-tryout.appspot.com/epltryout/mainform.html)

## The problem

> Let's suppose you want to monitor an Italian Bocce game through a data streaming device. The game of Bocce requires the presence of a "Boccino," a small sphere, some "Bocce," larger spheres compared to the Boccino, and two players (or two teams).

The Boccino is thrown by one of the two players at the beginning of each round. The goal for the two players is to throw their own Bocce, four each, trying to get them as close as possible to the Boccino. Players take turns, and in each turn, a player throws only one Boccia toward the Boccino. The thrown Boccia can come into contact with the Bocce already on the playing field, resulting in a change in the arrangement of the Bocce in play (the Bocce can also touch the Boccino).

At the end of each round, one point will be awarded to the player with the Boccia closest to the Boccino.

### Modeling
> Formalize in EPL the schema of the two streams.

Reading the description carefully it is possible to derive the following schemas:
```  
create schema Boccia(
    playerID int,
    numero int,
    distance double,
    status string
);

create schema Player(
    playerID int, // can be sostituted by teamID if #members for each team is > 1
    name string
    // other informations about each player
);

create schema Boccino(
    round int,
    distance double //represents the distance from the bowling line (not necessary)
);
```

An instance of the **Player** stream defines the players in the game (it is not mandatory but it could be useful if we want to store players' informations)
An instance of the **Boccia** stream is issued once a certain Boccia is thrown or is moving.
An instance of **Boccino** stream is issued when every round starts.

### Assumptions
Before delving into the problem it is better to define some assumptions to better model the data stream and the queries:

- An instance of the Boccia type will only be thrown if the same Boccia has moved (either thrown by the player or hit by other bocce). The status of the Boccia represents this: status = "thrown" if the boccia is thrown, status = "moving" if the boccia is hit.

- Between one throw of the boccia and another, 10 seconds pass, and the total duration of the game is 80 seconds

### Data stream generation
In order to test the queries that will be later presented we will use the following data stream. It is suggested to modify it using different configurations to test wether your queries will work in different settings.

``` 
/*
Player = {playerID = 1, name = 'Marco'}
Player = {playerID = 2, name = 'Giorgia'}
*/

Boccino = {round = 1, distance = 7}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero=1, distance=2, status = "throw"}

t=t.plus(10 seconds)

Boccia = {playerID = 2, numero=1, distance=3, status = "throw"}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero = 2, distance = 3, status = "throw"}
Boccia = {playerID = 1, numero = 1, distance = 1.5, status = "moving"}
Boccia = {playerID = 2, numero = 1, distance = 5, status = "moving"}

t=t.plus(10 seconds)

Boccia = {playerID = 2, numero=2, distance=0.5, status = "throw"}
Boccia = {playerID = 1, numero = 1, distance = 3.5, status = "moving"}


t=t.plus(10 seconds)
Boccia = {playerID = 1, numero=3, distance=2, status = "throw"}

t=t.plus(10 seconds)
Boccia = {playerID = 2, numero=3, distance=5, status = "throw"}

t=t.plus(10 seconds)
Boccia = {playerID = 1, numero=4, distance=7, status = "throw"}

t=t.plus(10 seconds)
Boccia = {playerID = 2, numero=4, distance=8, status = "throw"}

t=t.plus(10 seconds)

```

### Assignment

> Q1) Determine the total number of Bocce that have been thrown since the beginning of the **match**
> Q1 bis) Determine how many Bocce each player has thrown

#### Solution
Under the assumption of one round in the data stream, we can answer to the above requests as follows:
```  
@Name('Q1')
SELECT COUNT(*)
FROM pattern[
every b = Boccino() 
-> 
every B = Boccia(B.status="throw")
];
```

```  
@Name('Q1bis')
SELECT B.playerID as playerID, COUNT(*)
FROM pattern[
every b = Boccino() 
-> 
every B = Boccia(B.status="throw")
]
GROUP BY B.playerID
;
```

But what happens if the data stream admits multiple rounds? 

``` 
/*
Player = {playerID = 1, name = 'Marco'}
Player = {playerID = 2, name = 'Giorgia'}
*/

Boccino = {round = 1, distance = 7}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero=1, distance=2, status = "throw"}

t=t.plus(10 seconds)

Boccia = {playerID = 2, numero=1, distance=3, status = "throw"}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero = 2, distance = 3, status = "throw"}
Boccia = {playerID = 1, numero = 1, distance = 1.5, status = "moving"}
Boccia = {playerID = 2, numero = 1, distance = 5, status = "moving"}

t=t.plus(10 seconds)

Boccia = {playerID = 2, numero=2, distance=0.5, status = "throw"}
Boccia = {playerID = 1, numero = 1, distance = 3.5, status = "moving"}


t=t.plus(10 seconds)
Boccia = {playerID = 1, numero=3, distance=2, status = "throw"}

t=t.plus(10 seconds)
Boccia = {playerID = 2, numero=3, distance=5, status = "throw"}

t=t.plus(10 seconds)
Boccia = {playerID = 1, numero=4, distance=7, status = "throw"}

t=t.plus(10 seconds)
Boccia = {playerID = 2, numero=4, distance=8, status = "throw"}

t=t.plus(10 seconds)

Boccino = {round = 2, distance = 5}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero=1, distance=3, status = "throw"}

t=t.plus(10 seconds)

//the stream goes on...

```

The clause _every b -> every B_ matches an event _everytime_ we have an event Boccino followed by an event Boccia (thrown). Hence, the event Boccino(round=1) will match also the event Boccia(playerID = 1) of the second round. The solution to this problem can be modelled in different ways, e.g.:

```  
@Name('Q1')
SELECT COUNT(*)
FROM pattern[
every b = Boccino() 
-> 
every B = Boccia(B.status="throw")
and not b2 = Boccino()
];
```
or, knowing that each round lasts 80 seconds:

```  
@Name('Q1_time')
SELECT COUNT(*)
FROM pattern[
every b = Boccino() 
-> 
(every B = Boccia(B.status="throw")
where timer:within(80 seconds))
];
```


/* Da fare a casa
```  
@Name('Q1bis_time')
SELECT B.playerID as playerID, COUNT(*)
FROM pattern[
every b = Boccino() 
-> 
(every B = Boccia(B.status="throw")
where timer:within(80 seconds))
]
GROUP BY B.playerID
;
```
/*


### Assignment

An alternative request for this type of problem can be:

> Q2) Determine the total number of Bocce that have been thrown since the beginning of the **round**
> Q2-bis) Determine how many Bocce each player has thrown
> Q2-extra) Determine the total number of Bocce that have been thrown since the beginning of the i-th **round**

### Solution

The Q1 solution is not an option anymore, because it counts the entire amount of Bocce that has been thrown since the start of the datastream.
We can then solve the problem in the following ways:

```  
@Name('Q2')
SELECT b.round as round, COUNT(*)
FROM pattern[
every b = Boccino() 
-> 
every B = Boccia(B.status="throw")
and not b2 = Boccino()
];
GROUP BY b.round
```

Alternatively, you can use sliding logical windows using a support scheme: (PERCHE' NON FUNZIONA??)

/*
```
create schema BoccePerRound(
round int
);
```

```  
@Name('Q2_support')
INSERT INTO BoccePerRound
SELECT b.round as round
FROM pattern[
every b = Boccino() 
-> 
(every B = Boccia(B.status="throw")
where timer:within(80 seconds)) 
];

@Name('Q2')
SELECT round, COUNT(*)
FROM BoccePerRound.win:time(80 seconds);

```
*/

Try to solve Q2-Bis by yourself...

Moreover, we can answer to the question Q2-extra considering, e.g, i=2, using the HAVING clause:
```
@Name('Q2-extra')
SELECT b.round as round, COUNT(*)
FROM pattern[
every b = Boccino() 
-> 
every B = Boccia(B.status="throw")
and not b2 = Boccino()
]
GROUP BY b.round
HAVING b.round = 2;
```


### Assignment
//> Q3-easy) State the average distance from the Boccino for the last two Bocce:
> Q3) State the average distance from the Boccino for the last two  _thrown_ Bocce:


### Solution

```
create schema ThrownBoccia(
    playerID int,
    numero int,
    distance double
);

@Name('Q3_support')
INSERT INTO ThrownBoccia
SELECT playerID, numero, distance
FROM Boccia
WHERE status="throw"; // funziona anche HAVING 

@Name('Q3')
SELECT AVG(distance)
FROM ThrownBoccia.win:length(2)
```
**Note**: Since the text does not specify "in the current round", the above is the correct solution. An alternative rational solution could have been:
```
create schema ThrownBocciaAlt(
    playerID int,
    numero int,
    distance double,
    round int
);

@Name('Q3_support_alt')
INSERT INTO ThrownBocciaAlt
SELECT B.playerID as playerID, B.numero as numero, B.distance as distance, b.round as round
FROM pattern[
    every b = Boccino()
    ->
    every B = Boccia()
    and not b2 = Boccino
]
WHERE B.status="throw"; 

@Name('Q3_alt')
SELECT round, AVG(distance)
FROM ThrownBocciaAlt.win:length(2)
GROUP BY round
```

### Assignment

> Q4) Identify the player leading the game, i.e., the player whose Boccia is closest to the Boccino

### Solution

In order to meet this requirement, we need to associate a time stamp with the events, as the actual position of the Bocce is that given by the last event generated for the individual Boccia. All events in the game window must therefore be taken into account:

```  
create schema Boccia(
playerID int,
numero int,
distance double,
status string,
timestamp int
);

create schema Boccino(
    round int,
    distance double, //represents the distance from the bowling line (not necessary)
    timestamp int
)

/*
create schema Player(
playerID int,
name string
);
*/

```
/*
Player = {playerID = 1, name = 'Marco'}
Player = {playerID = 2, name = 'Giorgia'}
*/
Boccino = {round = 1, distance = 7, timestamp = 0}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero=1, distance=2, status = "throw", timestamp = 10}

t=t.plus(10 seconds)

Boccia = {playerID = 2, numero=1, distance=3, status = "throw", timestamp = 20}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero = 2, distance = 3, status = "throw", timestamp = 30}
Boccia = {playerID = 1, numero = 1, distance = 1.5, status = "moving", timestamp = 30}
Boccia = {playerID = 2, numero = 1, distance = 5, status = "moving", timestamp = 30}

t=t.plus(10 seconds)

Boccia = {playerID = 2, numero=2, distance=0.5, status = "throw", timestamp = 40}
Boccia = {playerID = 1, numero = 1, distance = 3.5, status = "moving", timestamp = 40}


t=t.plus(10 seconds)
Boccia = {playerID = 1, numero=3, distance=2, status = "throw", timestamp = 50}

t=t.plus(10 seconds)
Boccia = {playerID = 2, numero=3, distance=5, status = "throw", timestamp = 60}
Boccia = {playerID = 2, numero = 1, distance = 1, status = "moving", timestamp = 60}


t=t.plus(10 seconds)
Boccia = {playerID = 1, numero=4, distance=7, status = "throw", timestamp = 70}

t=t.plus(10 seconds)
Boccia = {playerID = 2, numero=4, distance=4, status = "throw", timestamp = 80}
Boccia = {playerID = 1, numero = 3, distance = 0.5, status = "moving", timestamp = 80}


t=t.plus(10 seconds)

Boccino = {round = 2, distance = 5, timestamp = 90}

t=t.plus(10 seconds)

Boccia = {playerID = 1, numero=1, distance=3, status = "throw", timestamp = 100}

t=t.plus(10 seconds)

```
Respecting the assumption of a round duration of 10 seconds (thus, 80 seconds per game), we can verify which boccia is closest to the Boccino.

```
create schema FinalPos(
    playerID int,
    numero int,
    distance double
);

@Name('Q4support')
INSERT INTO FinalPos
SELECT playerID, numero, distance
FROM Boccia.win:time_batch(80 seconds)
GROUP BY playerID, numero
HAVING timestamp = max(timestamp)
OUTPUT EVERY 10 seconds
;

@Name('Q4')
SELECT playerID, numero
FROM FinalPos.win:length_batch(8) //questa finestra è necessaria, altrimenti viene outputtato il minimo "a cascata" (output last non funziona).
HAVING distance = min(distance)
;
```

Eventually, you can insert this last event into another schema that records the points of the rounds and selects the winner at the end of the game.




/// DA QUA NON FEASIBLE SECONDO ME
We can also know the player who is currently leading the game

```
create schema Pos(
    playerID int,
    numero int,
    distance double,
timestamp int
);

create schema LastPos(
    playerID int,
    numero int,
    distance double
);

@Name('Q4support')
INSERT INTO LastPos
SELECT B.playerID as playerID, B.numero as numero, B.distance as distance, B.timestep as timestep
FROM pattern[
    every b = Boccino()
    ->
    every B = Boccia()
    and not b2 = Boccino()
]
;

@Name('Q4-BIS)
SELECT
FROM LastPos.win:time_batch(80 seconds)
OUTPUT every(10 seconds)

```
// OPPURE
@Name('Q4support')
INSERT INTO FinalPos
SELECT playerID, numero, distance
FROM Boccia.win:time(80 seconds)
GROUP BY playerID, numero
HAVING timestamp = max(timestamp)
;

@Name('Q4')
SELECT playerID, numero, distance
FROM FinalPos
HAVING distance = min(distance)
;

viene outputttato un evento se il minimo cambia!