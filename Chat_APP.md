Creating a complete codebase for an online chat application in JavaScript involves multiple components: the backend server, frontend client, and database integration. Below, I’ll provide a simplified codebase that you can use as a foundation, implementing key features like real-time messaging with WebSocket, emoji support, and a dark mode toggle.

---

### **Backend (Node.js with WebSocket)**

**`server.js`**
```javascript
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const cors = require('cors');

const app = express();
const server = http.createServer(app);
const io = new Server(server, {
    cors: {
        origin: 'http://localhost:3000',
        methods: ['GET', 'POST'],
    },
});

app.use(cors());
app.use(express.json());

let messages = []; // Store messages temporarily (use a database in production)

// WebSocket connection
io.on('connection', (socket) => {
    console.log('A user connected:', socket.id);

    // Send existing messages to the newly connected user
    socket.emit('loadMessages', messages);

    // Handle new messages
    socket.on('newMessage', (message) => {
        messages.push(message);
        io.emit('newMessage', message); // Broadcast the message to all clients
    });

    // Handle disconnection
    socket.on('disconnect', () => {
        console.log('A user disconnected:', socket.id);
    });
});

server.listen(4000, () => {
    console.log('Server running on http://localhost:4000');
});
```

---

### **Frontend (React)**

**Install dependencies:**
```bash
npm install react react-dom socket.io-client @mui/material @emotion/react @emotion/styled
```

#### **`App.js`**
```javascript
import React, { useEffect, useState } from 'react';
import io from 'socket.io-client';
import EmojiPicker from 'emoji-picker-react'; // Optional: Install this with npm install emoji-picker-react
import { createTheme, ThemeProvider } from '@mui/material/styles';
import { Button, TextField, Box, Typography } from '@mui/material';

const socket = io('http://localhost:4000');

const App = () => {
    const [messages, setMessages] = useState([]);
    const [currentMessage, setCurrentMessage] = useState('');
    const [showEmojiPicker, setShowEmojiPicker] = useState(false);
    const [darkMode, setDarkMode] = useState(
        localStorage.getItem('darkMode') === 'true'
    );

    useEffect(() => {
        socket.on('loadMessages', (loadedMessages) => {
            setMessages(loadedMessages);
        });

        socket.on('newMessage', (message) => {
            setMessages((prev) => [...prev, message]);
        });

        return () => {
            socket.disconnect();
        };
    }, []);

    const sendMessage = () => {
        if (currentMessage.trim()) {
            const message = { text: currentMessage, time: new Date().toISOString() };
            socket.emit('newMessage', message);
            setCurrentMessage('');
        }
    };

    const handleEmojiClick = (emoji) => {
        setCurrentMessage(currentMessage + emoji.emoji);
    };

    const toggleDarkMode = () => {
        setDarkMode(!darkMode);
        localStorage.setItem('darkMode', !darkMode);
    };

    const theme = createTheme({
        palette: {
            mode: darkMode ? 'dark' : 'light',
        },
    });

    return (
        <ThemeProvider theme={theme}>
            <Box
                sx={{
                    minHeight: '100vh',
                    bgcolor: 'background.default',
                    color: 'text.primary',
                    display: 'flex',
                    flexDirection: 'column',
                    alignItems: 'center',
                    justifyContent: 'center',
                }}
            >
                <Typography variant="h4" gutterBottom>
                    Online Chat Application
                </Typography>
                <Button onClick={toggleDarkMode}>
                    Toggle {darkMode ? 'Light' : 'Dark'} Mode
                </Button>
                <Box
                    sx={{
                        width: '80%',
                        maxHeight: '50vh',
                        overflowY: 'auto',
                        border: '1px solid',
                        borderRadius: '8px',
                        padding: '16px',
                        margin: '16px 0',
                    }}
                >
                    {messages.map((msg, idx) => (
                        <Typography key={idx} sx={{ marginBottom: '8px' }}>
                            {msg.text} <small>{new Date(msg.time).toLocaleTimeString()}</small>
                        </Typography>
                    ))}
                </Box>
                <Box display="flex" gap={1} alignItems="center">
                    <TextField
                        variant="outlined"
                        placeholder="Type a message..."
                        value={currentMessage}
                        onChange={(e) => setCurrentMessage(e.target.value)}
                        fullWidth
                    />
                    <Button onClick={() => setShowEmojiPicker(!showEmojiPicker)}>
                        😀
                    </Button>
                    <Button variant="contained" onClick={sendMessage}>
                        Send
                    </Button>
                </Box>
                {showEmojiPicker && (
                    <EmojiPicker onEmojiClick={handleEmojiClick} />
                )}
            </Box>
        </ThemeProvider>
    );
};

export default App;
```

---

### **Summary of Features**
1. **Real-Time Messaging**: WebSocket is used to send and receive messages in real time.
2. **Emoji Integration**: The app includes an emoji picker using the `emoji-picker-react` library.
3. **Dark Mode**: A toggle button switches themes between light and dark modes, persisting the user’s choice in local storage.

Let me know if you’d like additional features or further assistance!
