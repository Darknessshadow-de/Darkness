
📁 **PROJECT ROOT**
```
auction-platform/
│
├── backend/
│   ├── package.json
│   ├── server.js
│   ├── controllers/
│   ├── routes/
│   ├── db/
│   ├── .env
│   └── README.md
│
└── frontend/
    ├── package.json
    ├── vite.config.js
    ├── index.html
    └── src/
        ├── App.jsx
        ├── main.jsx
        ├── pages/
        ├── components/
        └── api/
```
================================================================================

# ========================= BACKEND (Node.js) =========================
📁 **backend/package.json**
```json
{
  "name": "auction-backend",
  "version": "1.0.0",
  "main": "server.js",
  "type": "module",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "socket.io": "^4.7.0"
  }
}
```

📁 **backend/server.js**
```javascript
import express from 'express';
import cors from 'cors';
import http from 'http';
import jwt from 'jsonwebtoken';
import { Server } from 'socket.io';
import dotenv from 'dotenv';
dotenv.config();

const app = express();
app.use(cors());
app.use(express.json());

const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });

let users = [];
let accounts = [];
let auctions = [];
let bids = [];
let messages = [];

function auth(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ error: "Missing Token" });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: "Invalid Token" });
  }
}

app.post('/auth/register', (req, res) => {
  const { email, password } = req.body;
  const id = users.length + 1;
  users.push({ id, email, password });
  res.json({ success: true });
});

app.post('/auth/login', (req, res) => {
  const { email, password } = req.body;
  const user = users.find(u => u.email === email && u.password === password);
  if (!user) return res.status(400).json({ error: "Invalid login" });
  const token = jwt.sign({ id: user.id, email: user.email }, process.env.JWT_SECRET);
  res.json({ token });
});

app.post('/accounts', auth, (req, res) => {
  const { title, description, base_price } = req.body;
  const id = accounts.length + 1;
  accounts.push({ id, seller_id: req.user.id, title, description, base_price });
  res.json({ id });
});

app.get('/accounts', (req, res) => res.json(accounts));

app.post('/auctions', auth, (req, res) => {
  const id = auctions.length + 1;
  auctions.push({ id, game_account_id: req.body.account_id, current_price: 0, status: 'live' });
  res.json({ id });
});

app.post('/auctions/:id/bid', auth, (req, res) => {
  const auction = auctions.find(a => a.id == req.params.id);
  const { amount } = req.body;

  if (!auction) return res.status(404).json({ error: "Auction not found" });
  if (amount <= auction.current_price) return res.status(400).json({ error: "Bid too low" });

  bids.push({ auction_id: auction.id, bidder_id: req.user.id, amount });
  auction.current_price = amount;

  io.of('/auctions').to(`auction:${auction.id}`).emit('bid-updated', {
    auctionId: auction.id,
    newPrice: amount,
    bidderId: req.user.id
  });

  res.json({ success: true });
});

io.of('/auctions').on('connection', socket => {
  socket.on('join-auction', id => socket.join(`auction:${id}`));

  socket.on('chat-message', msg => {
    messages.push(msg);
    io.of('/auctions').to(`auction:${msg.auctionId}`).emit('chat-message', msg);
  });
});

server.listen(4000, () => console.log('Backend Running on Port 4000'));
```

📁 **backend/.env**
```
JWT_SECRET=SUPERSECRET123
```````````
📁 **frontend/package.json**
```json
{
  "name": "auction-frontend",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "socket.io-client": "^4.7.0"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }

```
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Auction Platform</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```
``
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App.jsx';

ReactDOM.createRoot(document.getElementById('root')).render(<App />);
``
``
import React, { useState } from 'react';
import AuctionView from './components/AuctionView.jsx';

export default function App() {
  const [auctionId, setAuctionId] = useState(1);
  const [token, setToken] = useState("");

  return (
    <div>
      <h1>Free Fire ID Auction Platform</h1>

      <input placeholder="JWT Token" value={token} onChange={e=>setToken(e.target.value)} />
      <input placeholder="Auction ID" value={auctionId} onChange={e=>setAuctionId(e.target.value)} />

      <AuctionView auctionId={auctionId} token={token} />
    </div>
  );
}
`````````
import React, { useEffect, useState } from 'react';
import io from 'socket.io-client';

const socket = io('http://localhost:4000/auctions');

export default function AuctionView({ auctionId, token }) {
  const [price, setPrice] = useState(0);
  const [chat, setChat] = useState([]);
  const [msg, setMsg] = useState('');

  useEffect(() => {
    socket.emit('join-auction', auctionId);

    socket.on('bid-updated', data => {
      if (data.auctionId == auctionId) setPrice(data.newPrice);
    });

    socket.on('chat-message', data => {
      if (data.auctionId == auctionId) setChat(c => [...c, data]);
    });

    return () => {
      socket.off('bid-updated');
      socket.off('chat-message');
    };
  }, [auctionId]);

  const bid = () => {
    fetch(`http://localhost:4000/auctions/${auctionId}/bid`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
      body: JSON.stringify({ amount: price + 100 })
    });
  };

  const sendMsg = () => {
    socket.emit('chat-message', { auctionId, content: msg, user: 'You' });
    setMsg('');
  };

  return (
    <div>
      <h2>Auction #{auctionId}</h2>
      <p>Price: ₹{price}</p>
      <button onClick={bid}>Bid +100</button>

      <h3>Chat</h3>
      {chat.map((m, i) => <div key={i}><b>{m.user}</b>: {m.content}</div>)}

      <input value={msg} onChange={e => setMsg(e.target.value)} />
      <button onClick={sendMsg}>Send</button>
    </div>
  );
}
