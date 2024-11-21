# GAME_LOGICS

Creating a complete codebase for the multiplayer Tic-Tac-Toe (XOX) game outlined in the previous prompt requires setting up both the frontend (client-side) and backend (server-side) components. Below is a simplified version of the codebase for both parts using **React.js** for the frontend and **Node.js** with **Socket.io** for real-time communication on the backend.

This example will provide a basic framework, and you can expand it with additional features like authentication, advanced matchmaking, etc.

---

### **1. Backend (Node.js + Socket.io)**

**Step 1: Initialize a Node.js project**
```bash
mkdir tic-tac-toe-backend
cd tic-tac-toe-backend
npm init -y
npm install express socket.io
```

**Step 2: Set up the server (server.js)**

```javascript
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

let games = {}; // Object to store game states

// Serve static files (e.g., front-end)
app.use(express.static('public'));

// Create a new game room
io.on('connection', (socket) => {
    console.log(`User connected: ${socket.id}`);
  
    // Listen for creating a new game
    socket.on('create-game', (playerName) => {
        const gameId = socket.id; // Use socket id as unique game room ID
        games[gameId] = {
            player1: { id: socket.id, name: playerName, symbol: 'X' },
            player2: null,
            board: Array(9).fill(null),
            turn: 'X',
        };
        socket.join(gameId);
        console.log(`Game created with ID: ${gameId}`);
        socket.emit('game-created', gameId);
    });

    // Listen for joining an existing game
    socket.on('join-game', (gameId, playerName) => {
        if (games[gameId] && games[gameId].player2 === null) {
            games[gameId].player2 = { id: socket.id, name: playerName, symbol: 'O' };
            io.to(gameId).emit('game-started', games[gameId]);
        } else {
            socket.emit('game-error', 'Game is already full or doesn’t exist.');
        }
    });

    // Listen for player moves
    socket.on('make-move', (gameId, index) => {
        const game = games[gameId];
        if (!game || game.board[index] || game.turn !== (socket.id === game.player1.id ? 'X' : 'O')) {
            return;
        }
        game.board[index] = socket.id === game.player1.id ? 'X' : 'O';
        game.turn = game.turn === 'X' ? 'O' : 'X'; // Switch turn
        io.to(gameId).emit('move-made', game.board, game.turn);

        // Check for win or draw
        const winner = checkWinner(game.board);
        if (winner) {
            io.to(gameId).emit('game-over', winner);
            delete games[gameId];
        } else if (game.board.every(cell => cell !== null)) {
            io.to(gameId).emit('game-over', 'draw');
            delete games[gameId];
        }
    });

    // Listen for disconnect
    socket.on('disconnect', () => {
        console.log(`User disconnected: ${socket.id}`);
        // Handle game cleanup if a player disconnects
    });
});

// Function to check if there is a winner
const checkWinner = (board) => {
    const winningCombinations = [
        [0, 1, 2],
        [3, 4, 5],
        [6, 7, 8],
        [0, 3, 6],
        [1, 4, 7],
        [2, 5, 8],
        [0, 4, 8],
        [2, 4, 6]
    ];
    for (const [a, b, c] of winningCombinations) {
        if (board[a] && board[a] === board[b] && board[a] === board[c]) {
            return board[a];
        }
    }
    return null;
};

server.listen(4000, () => {
    console.log('Server is running on port 4000');
});
```

---

### **2. Frontend (React.js)**

**Step 1: Set up the React project**
```bash
npx create-react-app tic-tac-toe-frontend
cd tic-tac-toe-frontend
npm install socket.io-client
```

**Step 2: Create the Game components**

- **App.js (Main Component)**
```javascript
import React, { useState, useEffect } from 'react';
import io from 'socket.io-client';

const socket = io('http://localhost:4000');

function App() {
  const [gameId, setGameId] = useState('');
  const [playerName, setPlayerName] = useState('');
  const [board, setBoard] = useState(Array(9).fill(null));
  const [turn, setTurn] = useState('X');
  const [gameStatus, setGameStatus] = useState('');
  
  useEffect(() => {
    socket.on('game-created', (id) => {
      setGameId(id);
    });

    socket.on('game-started', (game) => {
      setBoard(game.board);
      setTurn(game.turn);
      setGameStatus(`Your turn (${game.turn})`);
    });

    socket.on('move-made', (board, nextTurn) => {
      setBoard(board);
      setTurn(nextTurn);
    });

    socket.on('game-over', (result) => {
      setGameStatus(result === 'draw' ? 'It\'s a draw!' : `${result} wins!`);
    });

    socket.on('game-error', (message) => {
      alert(message);
    });

    return () => {
      socket.off('game-created');
      socket.off('game-started');
      socket.off('move-made');
      socket.off('game-over');
      socket.off('game-error');
    };
  }, []);

  const handleCreateGame = () => {
    socket.emit('create-game', playerName);
  };

  const handleJoinGame = () => {
    socket.emit('join-game', gameId, playerName);
  };

  const handleMove = (index) => {
    if (board[index] || gameStatus !== '' || turn !== (playerName === 'Player 1' ? 'X' : 'O')) return;
    socket.emit('make-move', gameId, index);
  };

  return (
    <div className="App">
      <h1>Tic-Tac-Toe</h1>
      {!gameId && (
        <div>
          <input
            type="text"
            value={playerName}
            onChange={(e) => setPlayerName(e.target.value)}
            placeholder="Enter your name"
          />
          <button onClick={handleCreateGame}>Create Game</button>
        </div>
      )}

      {gameId && !gameStatus && (
        <div>
          <input
            type="text"
            value={gameId}
            onChange={(e) => setGameId(e.target.value)}
            placeholder="Enter game ID to join"
          />
          <button onClick={handleJoinGame}>Join Game</button>
        </div>
      )}

      <div>
        <h2>{gameStatus}</h2>
        <div style={{ display: 'grid', gridTemplateColumns: 'repeat(3, 100px)' }}>
          {board.map((cell, index) => (
            <button
              key={index}
              onClick={() => handleMove(index)}
              style={{ width: '100px', height: '100px', fontSize: '24px' }}
            >
              {cell}
            </button>
          ))}
        </div>
      </div>
    </div>
  );
}

export default App;
```

**Step 3: Run the app**

- **Run the backend server**:
```bash
node server.js
```

- **Run the frontend React app**:
```bash
npm start
```

### **How the App Works**
1. Players can create a game by entering their name, which will generate a game ID.
2. The other player can join the game by using the game ID.
3. The game board updates in real-time as players make moves.
4. When a player wins or the game ends in a draw, the game will display the result.

---

### **Next Steps & Features to Add**
1. **Player Authentication**: Integrate JWT-based login or OAuth (Google/Facebook).
2. **Styling**: Use CSS/SCSS to make the app look better.
3. **Match History and Leaderboard**: Track player stats and show historical data.
4. **Voice/Video Chat**: Use WebRTC for in-game communication.
5. **Bot Mode**: Add an AI opponent to play against when no second player is available.

This setup provides a basic framework for your Tic-Tac-Toe game with real-time multiplayer functionality.
