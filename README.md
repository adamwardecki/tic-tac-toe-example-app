# Tic-Tac-Toe
Tic Tac Toe game based on ReactJS and Syncano

## Prolog
Computer games market is really huge and playing alone or with friend with the same room is not enough for a long time. In shops and network we can find almost infinite count of games offering multiplayer gameplay not in the same room but in different corners of the world. Creating this type of multiplayer mode is not so hard to do but only if we use proper tools. TicTacToe application already reflect this.

## Technical info
Visual layer was written in React, players data and current board status is reflection of Data Objects created in Syncano platform. The Flux architecture is kept by Actions and Stores provided by Reflux.  

## Before we begin
To make this application work some steps need to be done. First you need to have Node.js v6.2.2 installed. You can find it [here](https://nodejs.org/en/). After you've installed Node.js you will need application files. They're available in [this](https://github.com/Syncano-Community/tic-tac-toe-example-app) repository or can be just downloaded directly [here](https://github.com/Syncano-Community/tic-tac-toe-example-app/archive/master.zip). When you unzip files run you command line and move to destination where you've unzipped application files and run `npm install` command - this will install all required packages. After that run `npm start` command to start the Webpack server. That all! Tic-Tac-Toe application runs on `localhost:8080` so you can go and play, but remember! This game requires two players :). Of course you can hack this by opening another tab with `localhost:8080` and play with yourself.

If for some reasons you don't want to go through all those steps you can just check it [here](https://syncano-community.github.io/tic-tac-toe-example-app/).

# How does it work?

## Syncano backend
To make it more clear how application works you can see prepared Syncano backend by installing Demo App. First you need to have Syncano account - can be created [here](https://dashboard.syncano.io/#/signup). If you already have one just go to Syncano [dashboard](https://dashboard.syncano.io) and login. Go to Demo Apps in header section and install the one named Tic-Tac-Toe. Now you can see in shared Instances list Instance `tic-tac-toe` which contain all needed things to make application work.

If you would like to prepare your own Instance you have to edit `src/Utils/Config.js` file with your data.

## Connecting new players
Players playing this game are represented by Syncano Data Objects, There are only two players available for this game and every of them have `is_connected` field. This field is telling application if connection for next player is allowed. If both players have this field set to `true` application will display proper notification. After new player join the game `connectPlayer` Action will be called. `Actions.connectPlayer()` is updating Data Object `is_connected` field from  `false` to `true`.

`connectPlayer` action:
```js
import Reflux from 'reflux';
import Connection from '../Utils/Connection';
import Config from '../Utils/Config';

  ...

let Actions = Reflux.createActions({
  connectPlayer: {
    children: ['completed', 'failure'],
    asyncResult: true
  },

  ...

});
Actions.connectPlayer.listen((currentPlayerId) => {
  Object.assign(playersParams, {id: currentPlayerId});
  Connection.DataObject.please().update(playersParams, {is_connected: true})
    .then(Actions.connectPlayer.completed)
    .catch(Actions.connectPlayer.failure);
});
```

## Fetching initial data
First of all let's talk about components. I'm not going to talk about every single component because the most of them  - if you know React even basics - are really simple and almost the same - take `props` and show them in proper place. But one of them is more complicated and need special attention - `Board.jsx`. It joins most of other components inside and hold some simple logic like calling update Data Objects.
```ComponentWillMount``` method calls 3 ```Actions```: ```fetchBoard```, ```enableBoardPoll``` and ```enablePlayersPoll``` which properly download ```Data Objects```, start listening on changes in ```Data Objects``` holding board data and start listening on changes in ```Data Objects``` holding players informations. Because those ```Actions``` download data and ```Stores``` listening for ```Actions``` save this data inside of themselves and than trigger data into component, we  will see ready board when component is ready.

```ComponentWillMount``` method:

```js
componentWillMount() {
  Actions.fetchBoard();
  Actions.enableBoardPoll();
  Actions.enablePlayersPoll();
}
```

```fetchBoard``` action:

```js
import Reflux from 'reflux';
import Connection from '../Utils/Connection';
import Config from '../Utils/Config';

  ...

let Actions = Reflux.createActions({
  fetchBoard: {
    children: ['completed', 'failure'],
    asyncResult: true
  },

  ...

});

let boardParams = {
  instanceName: Config.instanceName,
  className: Config.boardClassName
};

  ...

Actions.fetchBoard.listen(() => {
  Connection.DataObject.please().list(boardParams)
    .then(Actions.fetchBoard.completed)
    .catch(Actions.fetchBoard.failure);
});
```

```enableBoardPoll``` and ```enablePlayersPoll``` actions:

```js
Actions.enableBoardPoll.listen(() => {
  Connection.Channel.please().get(channelParams)
    .then(Actions.enableBoardPoll.completed)
    .catch(Actions.enableBoardPoll.failure);
});

Actions.enablePlayersPoll.listen(() => {
  Object.assign(channelParams, {name: 'tictactoeplayers'});
  Connection.Channel.please().get(channelParams)
    .then(Actions.enablePlayersPoll.completed)
    .catch(Actions.enablePlayersPoll.failure);
});
```
Now when players are connected and we have data fetched we can see what is happening after player click on some field on board (```handleFieldClick``` method). Well, the clicked field is updated in Syncano via ```updateFileld``` action and turn is switched via ```switchTurn``` action. If you look at ```Data Objects``` in ```tictactoeplayers``` class you will noticed that there is a field named ```is_player_turn```. In ```updateFiled``` action call this field is updated in both objects to simulate switching turn. As you can see there is ```setState``` method inside which is updating clicked field value locally to avoid waiting user for appearing value after API call is done. It will be updated also after call is finished. User will not see this but the opponent does.

```js
handleFieldClick(dataObjectId, index) {
  let state = this.state;
  let value = state.turn;

  if (state.items[index].value === null) {
    state.items[index].value = value;
    this.setState(state, () => {
    });
    Actions.updateField(dataObjectId, value);
    Actions.switchTurn(state.currentPlayer.id, state.opponent.id);
  }
}
```

Not it's time to look on the opponent site and know how will he see the response on our click. To understand this we have to look into ```Store``` and find method named ```onEnableBoardPollCompleted``` and ```onEnablePlayersPollCompleted```. We can see that when some update on ```Data Objects``` appears proper action fetching ```Data Objects``` will be called. This will update whole board and opponent will see changes on his board.

Actions enabling listening on ```Data Objects``` changes:

```js
onEnableBoardPollCompleted(channel) {
  let poll = channel.poll();

  poll.on('message', () => {
    Actions.fetchBoard();
  });
},

onEnablePlayersPollCompleted(channel) {
  let poll = channel.poll();

  poll.on('message', () => {
    Actions.fetchPlayers();
  });
}
```

When you look on ```renderFields``` method you will see that field can be disabled (unable to be clicked and put symbol inside) in a few cases. Let's focus on one which is checking if it is our turn. Do you remember ```switchTurn``` action? Yeah! It's updating players ```Data Objects``` and because we have enabled listening on those objects changes both clients will be noticed about changes and only opponent will be able to click fields.

## Checking winner
Every game should have a winner! Fortunately the rules of this game are not complicated so we can define all winning cases.

Winning combinations

```js
winCombinations: [[0, 1, 2], [3, 4, 5], [6, 7, 8], [0, 3, 6], [1, 4, 7], [2, 5, 8], [0, 4, 8], [2, 4, 6]],
```

Each array element contains array with indexes of board which filled with the same values makes player win.

Example for ```[0, 3, 6]```:
```js
X  -  -
X  -  -
X  -  -
```

Checking winning combinations method:

```js
checkWinner() {
  let items = this.data.items; // Data Objects fetched from 'tictactoe' Syncano class
  let currentPlayer = this.data.currentPlayer; // Data Object representing current player

  this.data.winCombinations.some((comb) => { // iterate over winning combinations
    let testArr = [items[comb[0]].value, items[comb[1]].value, items[comb[2]].value]; // 3 board values with indexes from current winning combination

    if (currentPlayer && this.isWinner(testArr)) {
      let winner = _.find(this.data.players, ['play_as', testArr[0]]);

      Actions.setWinner(winner.id); // mark player as winner in Syncano
      comb.forEach((index) => {
        this.data.items[index].color = '#F44336'; // mark winning fields in other color
      });
    }
  });
  this.data.isGameOver = this.isGameOver(); // helper value representing finished game
  this.trigger(this.data); // trigger data into listening components
},

isWinner(items) {
  let first = items[0];

  if (items.every((item) => item === null) || this.data.winner) {
    return false;
  }

  return items.every((item) => { // check if every item from test array is equal to first - true means that we have a winner
    return item === first;
  });
},
```

## Disconnecting players
Alright, we finished playing and we want to let others to play this game. No problem! The Demo App you have installed at the beginning contains Schedule which trigger Script every 2 minutes. This Script is checking players activity and if any player didn't make move from 2 minutes will be disconnected.

Script cleaning inactive players

```js
var Moment = require('moment');
var _ = require('lodash');
var Syncano = require('syncano');
var connection = new Syncano({apiKey: CONFIG.apiKey, instance: CONFIG.instanceName});

connection.class(CONFIG.className).dataobject().list().then(function(resp) {
  var players = resp.objects;

  _.forEach(players, function(player) {
    var lastActivity = player.updated_at;

    if (Moment(Date.now()).diff(lastActivity, 'minutes') > 5) {
      connection.class(CONFIG.className).dataobject(player.id).update({is_connected: false}, function(resp) {
        console.log(player.name + ' disconnected...');
      })
    }
  });
})
```


## Summary
I'm sure you would ask me why only 2 payers can play this game. Don't worry :) it will happen soon. Stay tuned and we will expand this app on new features like rooms to allow more players play game in the same time.

You can find whole code on [GitHub](https://github.com/Syncano-Community/tic-tac-toe-example-app/archive/master.zip)
