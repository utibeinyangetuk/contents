# Workshop: Morra Game

## Introduction
In this workshop, we'll be building a traditional game of morra using reach. Morra game is a game that involves 2 players.

The flow of the game goes as follows; the two players are allowed to pick a number of fingers i.e a number between 1 and 5 and then  predict the sum of their opponents finger and their's, the player that makes the right prediction goes ahead and wins the game.

But in this variation  we decided to spice things up with a wager where the winner wins the wager and in a case of a draw the two players get refunded and get to play again.


## Problem Analysis

This game was built and deployed on the blockchain using reach, the advantages of using smart contracts to emulate applications like this is that there  is no humanware in the middle, when humans are involved in transactions like this there's always a possibility of human created errors but with programs especially decentralized applications those factors are completely replaced and the errors depend on the efficency of the program which can be totally wiped out if the programmer takes his time and tests his smart contract and makes the expected changes to make the program perfect.

I did exactly that in my project and I did so by following some steps firstly I analyzed the problems by generating some important questions I would have to solve to attain that perfection I desire; below are the questions I had while brainstorming:

```
Who is involved in this application?

What is the logic of the game?

How do I ensure that the game is fair and cheating proof?

How do I ensure that the payment system of the game is perfect?
```

**Questions and Answers!**
```
This game involves 2 players: Alice and Bob (Attacher and Deployer).
```
```
The logic of the game goes as follows, Alice sets the wager, then deploys the contract, Bob then attaches to the contract accepts the wager set by Alice then the game begins, Alice starts first she picks the her desired number of fingers between 1 to 5 then she picks the her prediction which is the sum of her finger and the number of fingers she thinks Bob would pick, once she has successfully made those 2 inputs Bob is then allowed to go on with the same actions as  Alice i.e choosing his fingers and making his prediction. Once they have both played their inputs are been returned to the backend and the backend checks to see who made the right prediction and pays the player accordingly, if none of  them made the right guess they will be prompted to repeat the same steps above until there is a winner.
```
```
To ensure  there is not cheating and all the players are honest reach has various functions that are dedicated to making your program cheating proof and we will get back to that  duing the course of this  workshop.
```
```
The payment system is been perfected when testing the smart contract ensuring that the winner gets paid and not the other way around also  ensuring the funds dont get stuck in the contract.
```

## Application Logic

Now that we have an idea of the operations of the program lets just go ahead and break down how we would do all this using reach.

This workshop assumes that you've completed the rock paper scissors tutorial.

We assume that you’ll go through this workshop in a directory named ~/reach/morra:

```$ mkdir -p ~/reach/morra```

And that you have a copy of Reach installed in ~/reach so you can write

```$ ./reach version```

You should start by initializing your project this would create 2 files in your directory `index.mjs` and `index.rsh`

```$ ./reach init```

Now lets naviagate to our `index.rsh` file and start  writing our reach code, I would advice that you write every line of code yourself imbibe the habit of writing reach programs.

Lets start by creating a player object that includes all the common functions that Alice and  Bob can  perform.

```js
//Player abilities
const Player = {
  ...hasRandom,
  getFinger: Fun([], UInt),
  getPrediction: Fun([], UInt),
  seeOutcome: Fun([UInt], Null),
  informTimeout: Fun([], Null),
  seeOpponentMove: Fun([UInt, UInt], Null)
};
```
-  `...hasRandom` is a function that is used to generate random variables, it is important when trying to cryptographically encrypt data, we would see the need of this as we go on.
- `getFinger` is a function used to get the fingers of the players, and this action is done by the 2 players thus it is in this common object.
- `getPrediction` is used to get the predictions of the two players.
- `seeOutcome` is used by the players to see the winner or the outcome of the game.
- `informTimeout` is used to when there is a timeout.
- `seeOpponentMove` is used to show the players the move of their opponents at the end of every round

```js
export const main = Reach.App(() => {
//Alice interface
  const Alice = Participant('Alice', {
    ...Player,
    wager: UInt,
    deadline: UInt,

  });
//Bob interface
  const Bob   = Participant('Bob', {
    ...Player,
    acceptWager: Fun([UInt], Null),
  });
  init();
```
Next we would be creating the 2 participants in the game, passing the `Player` object into their interfaces and other functions that are not common. The function names pretty explain their functions already.

```js
//Alice wager and the deadline for the timeout
  Alice.only(() => {
    const wager = declassify(interact.wager) * 1000000;
    const deadline = declassify(interact.deadline);
  });
  Alice.publish(wager, deadline)
    .pay(wager);
  commit();

```
- Here Alice sets the wager and the deadline of the game.
- Then publishes it to the blockchain so everyone in the contract can see.
- The Alices pays the wager into the contract.

```js
//Bob accepting or rejecting the wager
  Bob.only(() => {
    interact.acceptWager(wager);
  });
  Bob.pay(wager)
    .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));

```
- Here Bob accepts the wager then pays the wager into the contract as well.
- The timeout function is added incase Bob decides to delay the flow of the game.

Next we would be starting the game i.e where Alice and Bob get to select their fingers and predictions.

```js

  var [hand1, hand2, hand3, hand4] = [0,0,0,0];
  invariant( balance() == 2 * wager);

  while ((hand1 + hand2 == hand3) && (hand1 + hand2 == hand4)) {
    commit();


    Alice.only(() => {
      const _handAlice = interact.getFinger();
      const _predictionAlice = interact.getPrediction();

      const [_handcommit, _handSalt] = makeCommitment(interact, _handAlice);
      const [_predictcommit, _predictsalt] = makeCommitment(interact, _predictionAlice);

      const commitAliceFinger = declassify(_handcommit);
      const commitAlicePredict = declassify(_predictcommit);


    });
    Alice.publish(commitAliceFinger, commitAlicePredict)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    commit();

    unknowable(Bob, Alice(_handAlice, _predictionAlice, _handSalt, _predictsalt));
```
- These features are put inside a while loop because we would want the actions to be repeated over and over again until at least one of the players get the right prediction.
- For the while loop syntax to be complete we need at least a mutable variable (`var`), then at least one invariant which is a condition that must be true before and after the while loop in this case we stated that the balance of the contract must be 2 times the wager and this is true because alice and bob paid in the wager into the contract.
- The condition of the while loop is that if Alice guess is not equal to the sum of Alice finger and Bobs fingers and  Bobs prediction is not equal to the sum of Bob and Alice fingers continue looping.
- In the loop Alice  interacts with the `getFinger` function which she uses to select the number of fingers of her choice but at this stage we do not want bob to know the fingers of Alice beacause if Bob does he can win the game everytime making the game unfair, so we use this reach command `makeCommitment` which encrypts the number of fingers selected by alice and publishes it to the contract, we'd do the same for Alice predictions then to verify that Alice fingers and prediction are unknown to Bob at this stage use this command `unknowable`.

```js
 Bob.only(() => {
      const handBob = declassify(interact.getFinger());
      const predictionBob = declassify(interact.getPrediction());

    });
    Bob.publish(handBob, predictionBob)
      .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));
    commit();
```
- Bob interacts with the `getFinger` function as well after Alice then interacts with the `getPrediction` function and publishes them.
- `.timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));` ensures Bob's prediction and fingers are been revealed same time Alice's fingers and predictions are been revealed.

```js
Alice.only(() => {
      const handAlice = declassify(_handAlice);
      const handSalt = declassify(_handSalt);
      const predictionAlice = declassify(_predictionAlice);
      const predictSalt = declassify(_predictsalt);
    });
    Alice.publish(handAlice, handSalt, predictionAlice, predictSalt)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    checkCommitment(commitAliceFinger, handSalt, handAlice);
    checkCommitment(commitAlicePredict, predictSalt, predictionAlice);

    Alice.interact.seeOpponentMove(handBob, predictionBob);
    Bob.interact.seeOpponentMove(handAlice, predictionAlice);
    [hand1, hand2, hand3, hand4] = [handAlice, handBob, predictionAlice, predictionBob];

    continue;

  }
```
- Remember Alice's fingers and predictions are still encrypted so to know the actual values of her prediction and fingers we need to declassify the salt and the other arbitary variable created by `makeCommitment`, once done we would go ahead and publish them to the blockchain.
- `checkCommitment` is used to check if the encrypted and actual values of Alices fingers and predictions actually correspond.
- Once Alice and Bob have interacted with the two functions successfully at the end of each loop Alice and Bob interact with the `seeOpponentMove` function.
- The mutable variables are then beeing updated and then the program flow goes back to the while loop conditions to check if theres a winner or not.

```js
  const outcome = winner(hand1, hand2, hand3, hand4);
//Uses the outcome to pay the winner
  payWinner(outcome,wager, Alice, Bob);
  commit();
  ```
- A function `winner` is used to get the winner of the game.
- Depending on th output of the function `winner` it is been passed into a function `payWinner` that pays the winner of the game the wager or refunds the players in case of a draw.
At the stage the game is deemed complete and the index.rsh file is complete, now you can build a functional  UI around this reach code.

Below is the complete `index.rsh` code:

```js
"reach 0.1";
//Outcome array
const [ isOutcome, B_WINS, DRAW, A_WINS ] = makeEnum(3);

//This computes the winner of the game
const winner = (hand1, hand2, hand3, hand4) => {

  if (hand1 + hand2 == hand3  && hand1 + hand2 != hand4){
    return A_WINS;
  }
  else if (hand1 + hand2 != hand3 && hand1 + hand2 == hand4){
    return B_WINS;
  }
  else  return DRAW;

};

// Makes the required payment to the winner
const payWinner = (outcome, wager, Alice, Bob) => {
  if(outcome == A_WINS) {
    transfer(2*wager).to(Alice);
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome);
    });
  }
  else if (outcome == B_WINS){
    transfer(2*wager).to(Bob);
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome);
    });

  }
  else {
    each([Alice, Bob], () => {
      interact.seeOutcome(outcome)
    });
    transfer(wager).to(Alice);
    transfer(wager).to(Bob);

  };
}


//Player abilities
const Player = {
  ...hasRandom,
  getFinger: Fun([], UInt),
  getPrediction : Fun([], UInt),
  seeOutcome: Fun([UInt], Null),
  informTimeout: Fun([], Null),
  seeOpponentMove: Fun([UInt, UInt], Null)
};


export const main = Reach.App(() => {
//Alice interface
  const Alice = Participant('Alice', {
    ...Player,
    wager: UInt,
    deadline: UInt,

  });
//Bob interface
  const Bob   = Participant('Bob', {
    ...Player,
    acceptWager: Fun([UInt], Null),
  });
  init();

  const informTimeout = () => {
    each([Alice, Bob], () => {
      interact.informTimeout();
    });
  };
//Alice wager and the deadline for the timeout
  Alice.only(() => {
    const wager = declassify(interact.wager) * 1000000;
    const deadline = declassify(interact.deadline);
  });
  Alice.publish(wager, deadline)
    .pay(wager);
  commit();



//Bob accepting or rejecting the wager
  Bob.only(() => {
    interact.acceptWager(wager);
  });
  Bob.pay(wager)
    .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));




  var [hand1, hand2, hand3, hand4] = [0,0,0,0];
  invariant( balance() == 2 * wager);

  while ((hand1 + hand2 == hand3) && (hand1 + hand2 == hand4)) {
    commit();


    Alice.only(() => {
      const _handAlice = interact.getFinger();
      const _predictionAlice = interact.getPrediction();

      const [_handcommit, _handSalt] = makeCommitment(interact, _handAlice);
      const [_predictcommit, _predictsalt] = makeCommitment(interact, _predictionAlice);

      const commitAliceFinger = declassify(_handcommit);
      const commitAlicePredict = declassify(_predictcommit);


    });
    Alice.publish(commitAliceFinger, commitAlicePredict)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    commit();

    unknowable(Bob, Alice(_handAlice, _predictionAlice, _handSalt, _predictsalt));
    Bob.only(() => {
      const handBob = declassify(interact.getFinger());
      const predictionBob = declassify(interact.getPrediction());

    });
    Bob.publish(handBob, predictionBob)
      .timeout(relativeTime(deadline), () => closeTo(Alice, informTimeout));
    commit();

    Alice.only(() => {
      const handAlice = declassify(_handAlice);
      const handSalt = declassify(_handSalt);
      const predictionAlice = declassify(_predictionAlice);
      const predictSalt = declassify(_predictsalt);
    });
    Alice.publish(handAlice, handSalt, predictionAlice, predictSalt)
      .timeout(relativeTime(deadline), () => closeTo(Bob, informTimeout));
    checkCommitment(commitAliceFinger, handSalt, handAlice);
    checkCommitment(commitAlicePredict, predictSalt, predictionAlice);

    Alice.interact.seeOpponentMove(handBob, predictionBob);
    Bob.interact.seeOpponentMove(handAlice, predictionAlice);
    [hand1, hand2, hand3, hand4] = [handAlice, handBob, predictionAlice, predictionBob];

    continue;


  }
  const outcome = winner(hand1, hand2, hand3, hand4);
  //Using the winner function with arguments of the users inputs and the random number to get the winner

//Uses the outcome to pay the winner
  payWinner(outcome,wager, Alice, Bob);




  commit();


});
```
For a link to my repo click [here](https://github.com/utibeinyangetuk/Umoja3-Hack).
## Conclusion

You have just implemented the Morra Game application that runs on the algorand blockchain yourself.

If you found this workshop rewarding please let us know on the [Discord Community](https://discord.gg/AZsgcXu).

Thanks You for your time.