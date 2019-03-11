---
name: A multiplayer "Place" game
route: /tutorials/multiplayergame
menu: Tutorials
---

# A multiplayer "Place" game with shared world state.

A multiplayer game is typically characterized by taking place within a single world that all players can affect. Let's build one!

This is commonly implemented by setting up a coordinate system which represents locations within the world. A simple key-value mapping stores the state of the world at a particular coordinate.

In this tutorial, we will write a very simple game with a shared world state. The world is represented as a square playing field and the only property that is available at each location is its 'color'. Some of you may recognize this as "place", which made its way around the Internet a while ago.

See and play with a working solution here: [https://studio.nearprotocol.com/?f=fnpeopb37&quickstart](https://studio.nearprotocol.com/?f=fnpeopb37&quickstart)

You can see a screenshot of this \(which has obviously been contributed to by many people\) below:

![Contract selection modal](https://github.com/pndpo/docs/tree/e82e3ffeb7ef4c570f46da3f6b9e19fd004e78ab/src/tutorials/public/screenshots/multiplayergame/near_place_screenshot.png)

## Step 1 -- Start a new fiddle in NEARstudio

Go to [https://studio.nearprotocol.com/](https://studio.nearprotocol.com/) and start a new project \(fiddle\) by selecting "Token Smart Contract in AssemblyScript" and click "Create".

![Contract selection modal](https://github.com/pndpo/docs/tree/e82e3ffeb7ef4c570f46da3f6b9e19fd004e78ab/src/tutorials/public/screenshots/multiplayergame/studio_choose_contract_modal.png)

This sample project has a token smart contract \(i.e. code that runs on blockchain\) and also some JavaScript tests that invoke smart contract functions.

You can try running these tests right away to see the code interacting with the blockchain by clicking "Test". It should open a new window and show the test results using the standard Jasmine browser UI.

Also note that **we are not going to keep any of the code from this template**, it's just there as a starting point.

## Step 2 - Write a smart contract

In this simple game, we need to create only two actions:

1. View the world state: `getCoords`
2. Make changes to the state at particular coordinates: `setCoords`

In a more complex game with a large world, it is optimal to avoid returning the state of the entire world at once. Because our game is small and simple, we don't have to worry about this.

Navigate to `assembly/main.ts`, and:

1. Delete everything that is there.
2. Implement the `setCoords` and `getCoords` functions using the `globalStorage` object's `setItem` and `getItem` functions:

```text
// assembly/main.ts

export function setCoords(coords: string, value: string): void {
  globalStorage.setItem(coords, value);
}

export function getCoords(coords: string): string {
  return globalStorage.getItem(coords);
}
```

We also need a `getMap` function, which returns the full state of the game \(we don't want to be making a separate call for every coordinate!\)

```text
// assembly/main.ts
...
export function getMap(): string[] {
  let num_rows = 10;
  let num_cols = 10;
  let total_cells = num_rows * num_cols;
  var arrResult:string[] = new Array(total_cells);
  let i = 0;
  for (let row=0; row<num_rows; row++) {
    for (let col=0; col<num_cols; col++) {
      let cellEntry = globalStorage.getItem(near.str(row) + "," + near.str(col));
      arrResult[i] = cellEntry;
      i++;
    }
  }
  return arrResult;
}
```

## Step 3 -- Write a couple of tests for the contract

We can test the contract right away by writing some code in JavaScript. Open `src/main.js` and modify it to call the functions that we just wrote.

First let's call `getMap`. It's a function which does not modify the state, so we can call it through a `callViewFunction` interface. Replace the contents of `main.js` with the following, and then try running it by clicking "test".

```text
// src/main.js
...

function sleep(time) {
  return new Promise(function (resolve, reject) {
    setTimeout(resolve, time);
  });
}

describe("NearPlace", function() {
  let contract;
  let accountId;

  // Contains all the steps that are necessary to
  //    establish a connection with a dev instance
  //    of the blockchain.
  beforeAll(async function() {
      const config = await nearlib.dev.getConfig();
      near = await nearlib.dev.connect();
      accountId = nearlib.dev.myAccountId;
      const url = new URL(window.location.href);
      config.contractName = url.searchParams.get("contractName");
      console.log("nearConfig", config);
      await sleep(1000);
      contract = await near.loadContract(config.contractName, {
        // NOTE: This configuration only needed while NEAR is still in development
        viewMethods: ["getMap"],
        changeMethods: ["setCoords"],
        sender: accountId
      });
  });

  describe("getMap", function() {
    it("can get the board state", async function() {
      const viewResult = await contract.getMap({});
      expect(viewResult.length).toBe(100); // board is 10 by 10
    });
  });
});
```

The getMap test simply invokes the getMap function of the contract. Note the syntax: `contract.getMap(args)`, where `args` is a JavaScript object containing the arguments. In this case, our function has no parameters, so we are passing an empty object.

Second, let's try to modify the game state! Add this to `main.js`, and run it by clicking "Test".

```text
  // src/main.js
  ...
  describe("setCoords", function() {
    it("modifies the board state", async function() {
      const setResult = await contract.setCoords({
        coords: "0,0",
        value: "111111"});
      console.log(setResult);
      const viewResult = await contract.getMap({});
      expect(viewResult.length).toBe(100); // board is 10 by 10
      // entry 0,0 should be 111111!
      expect(viewResult[0]).toBe("111111")
    });
  });
```

## Step 4 -- Make a simple UI

All the blockchain work is done! Let's make a very simple JavaScript user interface \(UI\).

We need a few more tweaks to `main.js` to include some UI JavaScript - add the following to the file:

```text
// src/main.js
...
// Loads nearlib and this contract into nearplace scope.
nearplace = {};
let initPromise;
initContract = function () {
  if (nearplace.contract) {
    return Promise.resolve();
  }
  if (!initPromise) {
    initPromise = doInitContract();
  }
  return initPromise;
}

async function doInitContract() {
  const config = await nearlib.dev.getConfig();
  console.log("nearConfig", config);
  nearplace.near = await nearlib.dev.connect();
  nearplace.contract = await nearplace.near.loadContract(config.contractName, {
    viewMethods: ["getMap"],
    changeMethods: ["setCoords"],
    sender: nearlib.dev.myAccountId
  });

  loadBoardAndDraw();
  nearplace.timedOut = false;
  const timeOutPeriod = 10 * 60 * 1000; // 10 min
  setInterval(() => { nearplace.timedOut = true; }, timeOutPeriod);
}

function sleep(time) {
  return new Promise(function (resolve, reject) {
    setTimeout(resolve, time);
  });
}

initContract().catch(console.error);

function loadBoardAndDraw() {
  if (nearplace.timedOut) {
    console.log("Please reload to continue");
    return;
  }
  const board = getBoard().then((fullMap) => {
    console.log(fullMap);
    var canvas = document.getElementById("myCanvas");
    var ctx = canvas.getContext("2d");
    var i = 0;
    for (var x = 0; x < 10; x++) {
      for (var y = 0; y < 10; y++) {
        var color = fullMap[i];
        if (!color) {
          color = "000000";
        }
        ctx.fillStyle = "#" + color;
        ctx.fillRect(x*10, y*10, 10, 10);
        i++;
      }
    }
  });
}

function getMousepos(canvas, evt){
  var rect = canvas.getBoundingClientRect();
  return {
    x: evt.clientX - rect.left,
    y: evt.clientY - rect.top
  };
}

function myCanvasClick(e) {
  const canvas = document.getElementById("myCanvas");
  const ctx = canvas.getContext("2d");
  const position = getMousepos(canvas, e);
  const x = Math.floor(position.x/10);
  const y = Math.floor(position.y/10);

  const coords = x + "," + y;
  const rgb = document.getElementById('picker').value;
  ctx.fillStyle = "#" + rgb;
  ctx.fillRect(x*10, y*10, 10, 10);

  var readMethodName = "setCoords";
  args = { coords: coords, value: rgb };
  nearplace.contract.setCoords(args);
}

async function getBoard() {
  const result = await nearplace.contract.getMap({})
  console.log(result);
  return result;
}
```

We are using the "jscolor picker" to pick a color. To implement this:

1. Download the jscolor .zip file using the instructions at [http://jscolor.com/](http://jscolor.com/)
2. Unzip the file and copy it into the `src/` directory in the Studio window \(you can drag and drop it\)

Finally, replace the content of the `main.html` file with the following:

```text
// src/main.html

<!DOCTYPE html>
<html lang="en">
  <head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <!-- The above 3 meta tags *must* come first in the head; any other head content must come *after* these tags -->
  <script src="https://cdn.jsdelivr.net/npm/nearlib@0.1.1/dist/nearlib.js"></script>
  <script src="./main.js"></script>
  <script src="jscolor.js"></script>
  <span id="container"></span>
  <title>NEAR PLACE</title>
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap-theme.min.css">
  <style>
    .glyphicon-refresh-animate {
            -animation: spin .7s infinite linear;
            -webkit-animation: spin2 .7s infinite linear;
    }

    @-webkit-keyframes spin2 {
            from { -webkit-transform: rotate(0deg);}
            to { -webkit-transform: rotate(360deg);}
    }

    @keyframes spin {
            from { transform: scale(1) rotate(0deg);}
            to { transform: scale(1) rotate(360deg);}
    }
  </style>
  </head>
  <body style="padding-top: 70px; padding-bottom: 30px;">
    <!-- Fixed navbar -->
    <nav class="navbar navbar-inverse navbar-fixed-top">
        <div class="container">
        <div class="navbar-header">
            <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#navbar" aria-expanded="false" aria-controls="navbar">
            <span class="sr-only">Toggle navigation</span>
            <span class="icon-bar"></span>
            <span class="icon-bar"></span>
            <span class="icon-bar"></span>
            </button>
            <a class="navbar-brand" href="#">NEAR PLACE</a>
        </div>
        <div id="navbar" class="navbar-collapse collapse">
            <ul class="nav navbar-nav">
            <li class="active"><a href="#">Home</a></li>
            <li><a href="#about">About</a></li>
            <li><a href="#contact">Contact</a></li>
            </ul>
        </div><!--/.nav-collapse -->
        </div>
    </nav>

    <div class="container" role="main">
        <div class="jumbotron">
            <h1>PLACE</h1>
            <p>Imagine drawing <b>forever</b> on the blockchain.</p>
          </div>
        <div align="center">
        <canvas
          id="myCanvas"
          class="drawingboard",
          width="100"
          height="100"
          onclick="myCanvasClick(event);"
          style="border:1px solid #000000;"></canvas>
        </canvas>
        </div>
        <div align="center">
        <input class="jscolor" id="picker" value="ab2567">
    </div>
  </body>
</html>
```

The game should now work and show the UI in NEAR Studio. To run the UI, use the "Run" button.

Happy gaming!
