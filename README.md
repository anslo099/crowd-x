#!/bin/bash

# Create root folder
mkdir crowdx && cd crowdx

############## BACKEND ##############
mkdir backend && cd backend

# Backend package.json
cat <<EOF > package.json
{
  "name": "crowdx-backend",
  "version": "1.0.0",
  "main": "server.js",
  "type": "module",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.2",
    "mongoose": "^7.0.3",
    "ws": "^8.12.0"
  }
}
EOF

# Backend .env.example
cat <<EOF > .env.example
PORT=5000
MONGO_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/crowdx
JWT_SECRET=your_secure_jwt_secret
EOF

# Dockerfile
cat <<EOF > Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
EOF

echo "node_modules" > .dockerignore
echo ".env" >> .dockerignore

# server.js
cat <<'EOF' > server.js
import express from 'express';
import mongoose from 'mongoose';
import dotenv from 'dotenv';
import cors from 'cors';
import authRoutes from './routes/auth.js';
import marketRoutes from './routes/markets.js';
import orderRoutes from './routes/orders.js';
import dashboardRoutes from './routes/dashboard.js';
import { startPriceFeed } from './wsFeed.js';

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());
app.use('/api/auth', authRoutes);
app.use('/api/markets', marketRoutes);
app.use('/api/orders', orderRoutes);
app.use('/api/dashboard', dashboardRoutes);

mongoose.connect(process.env.MONGO_URI).then(() => {
  console.log('MongoDB connected');
  const server = app.listen(5000, () => console.log('API running on 5000'));
  startPriceFeed(server);
});
EOF

# Models and Routes
mkdir models routes

# User model
cat <<'EOF' > models/User.js
import mongoose from 'mongoose';
const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  balance: { type: Number, default: 10000 },
  orders: [{ symbol: String, type: String, amount: Number, price: Number }]
});
export default mongoose.model('User', userSchema);
EOF

# Auth route
cat <<'EOF' > routes/auth.js
import express from 'express';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

const router = express.Router();
router.post('/register', async (req,res)=>{
  const {email,password}=req.body;
  const exists=await User.findOne({email});
  if(exists) return res.status(400).json({msg:'User exists'});
  const hash=await bcrypt.hash(password,10);
  const user=await User.create({email,password:hash});
  res.json({msg:'Registered', user:user.email});
});
router.post('/login', async (req,res)=>{
  const {email,password}=req.body;
  const user=await User.findOne({email});
  if(!user) return res.status(400).json({msg:'Invalid credentials'});
  const match=await bcrypt.compare(password,user.password);
  if(!match) return res.status(400).json({msg:'Invalid credentials'});
  const token=jwt.sign({id:user._id},process.env.JWT_SECRET,{expiresIn:'1d'});
  res.json({token});
});
export default router;
EOF

# Markets route
cat <<'EOF' > routes/markets.js
import express from 'express';
const router = express.Router();
let prices=[{symbol:'BTC/USDT',price:50000},{symbol:'ETH/USDT',price:3200}];
router.get('/',(req,res)=>res.json(prices));
export const updatePrices=()=>{prices=prices.map(p=>({...p,price:+(p.price*(1+(Math.random()-0.5)/100)).toFixed(2)}))};
export default router;
EOF

# Orders route
cat <<'EOF' > routes/orders.js
import express from 'express';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';
const router=express.Router();
const auth=(req,res,next)=>{
  const token=req.headers.authorization?.split(' ')[1];
  if(!token) return res.sendStatus(401);
  jwt.verify(token,process.env.JWT_SECRET,(err,user)=>{if(err)return res.sendStatus(403);req.user=user;next();});
};
router.post('/',auth,async(req,res)=>{
  const {symbol,type,amount,price}=req.body;
  const user=await User.findById(req.user.id);
  const order={symbol,type,amount,price};
  user.orders.push(order);
  user.balance-=amount*price;
  await user.save();
  res.json({msg:'Order placed',order});
});
export default router;
EOF

# Dashboard route
cat <<'EOF' > routes/dashboard.js
import express from 'express';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';
const router=express.Router();
const auth=(req,res,next)=>{
  const token=req.headers.authorization?.split(' ')[1];
  if(!token)return res.sendStatus(401);
  jwt.verify(token,process.env.JWT_SECRET,(err,user)=>{if(err)return res.sendStatus(403);req.user=user;next();});
};
router.get('/',auth,async(req,res)=>{
  const user=await User.findById(req.user.id);
  res.json({balance:user.balance,orders:user.orders});
});
export default router;
EOF

# WebSocket price feed
cat <<'EOF' > wsFeed.js
import { WebSocketServer } from 'ws';
import { updatePrices } from './routes/markets.js';
export const startPriceFeed=(server)=>{
  const wss=new WebSocketServer({server});
  setInterval(()=>{
    updatePrices();
    const data=JSON.stringify({event:'prices',data:'updated'});
    wss.clients.forEach(client=>client.send(data));
  },2000);
};
EOF

cd ..

############## FRONTEND ##############
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir false --import-alias "@/*" <<< $'\nn\n'

cd frontend

# Frontend env example
cat <<EOF > .env.example
NEXT_PUBLIC_API_URL=http://localhost:5000/api
EOF

# API client
mkdir lib
cat <<'EOF' > lib/api.ts
import axios from 'axios';
const API=axios.create({baseURL:process.env.NEXT_PUBLIC_API_URL});
API.interceptors.request.use((config)=>{const token=localStorage.getItem('token');if(token)config.headers.Authorization=\`Bearer \${token}\`;return config;});
export default API;
EOF

# Login page
mkdir -p app/login
cat <<'EOF' > app/login/page.tsx
"use client";
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import API from '@/lib/api';
export default function Login() {
  const [email,setEmail]=useState('');
  const [password,setPassword]=useState('');
  const [error,setError]=useState('');
  const router=useRouter();
  const handleLogin=async(e:any)=>{e.preventDefault();
    try{const res=await API.post('/auth/login',{email,password});
      localStorage.setItem('token',res.data.token);
      router.push('/dashboard');
    }catch(err:any){setError(err.response.data.msg);}
  };
  return(<div className="max-w-md mx-auto mt-10 bg-gray-800 p-6 rounded">
    <h2 className="text-2xl mb-4">Login to Crowd X</h2>
    {error && <p className="text-red-400">{error}</p>}
    <form onSubmit={handleLogin} className="space-y-4">
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)}/>
      <input type="password" className="w-full p-2 bg-gray-700 rounded" placeholder="Password" value={password} onChange={e=>setPassword(e.target.value)}/>
      <button className="w-full bg-yellow-400 p-2 rounded text-black">Login</button>
    </form>
  </div>);
}
EOF

# Register page
mkdir -p app/register
cat <<'EOF' > app/register/page.tsx
"use client";
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import API from '@/lib/api';
export default function Register() {
  const [email,setEmail]=useState('');
  const [password,setPassword]=useState('');
  const [msg,setMsg]=useState('');
  const router=useRouter();
  const handleRegister=async(e:any)=>{e.preventDefault();
    try{await API.post('/auth/register',{email,password});
      setMsg('Registered! Redirecting...');
      setTimeout(()=>router.push('/login'),1500);
    }catch(err:any){setMsg(err.response.data.msg);}
  };
  return(<div className="max-w-md mx-auto mt-10 bg-gray-800 p-6 rounded">
    <h2 className="text-2xl mb-4">Register on Crowd X</h2>
    {msg && <p className="text-yellow-400">{msg}</p>}
    <form onSubmit={handleRegister} className="space-y-4">
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)}/>
      <input type="password" className="w-full p-2 bg-gray-700 rounded" placeholder="Password" value={password} onChange={e=>setPassword(e.target.value)}/>
      <button className="w-full bg-yellow-400 p-2 rounded text-black">Register</button>
    </form>
  </div>);
}
EOF

# Markets page
mkdir -p app/markets
cat <<'EOF' > app/markets/page.tsx
"use client";
import { useEffect,useState } from "react";
import API from "@/lib/api";
export default function Markets(){
  const [prices,setPrices]=useState<any[]>([]);
  useEffect(()=>{
    const fetchPrices=async()=>{const res=await API.get('/markets');setPrices(res.data);};
    fetchPrices();
    const ws=new WebSocket('ws://localhost:5000');
    ws.onmessage=(msg)=>{if(JSON.parse(msg.data).event==='prices')fetchPrices();};
    return()=>ws.close();
  },[]);
  return(<div>
    <h2 className="text-2xl mb-4">Markets</h2>
    <table className="w-full text-left border border-gray-700">
      <thead><tr className="border-b border-gray-700"><th>Symbol</th><th>Price</th></tr></thead>
      <tbody>{prices.map((p,i)=>(
        <tr key={i} className="border-b border-gray-700"><td>{p.symbol}</td><td className="text-yellow-400">\${p.price}</td></tr>
      ))}</tbody>
    </table>
  </div>);
}
EOF

# Trade page
mkdir -p app/trade/[symbol]
cat <<'EOF' > app/trade/[symbol]/page.tsx
"use client";
import { useParams } from "next/navigation";
import { useState } from "react";
import API from "@/lib/api";
export default function Trade(){
  const {symbol}=useParams();
  const [amount,setAmount]=useState('');
  const [price,setPrice]=useState('');
  const [msg,setMsg]=useState('');
  const placeOrder=async(type:string)=>{
    try{const res=await API.post('/orders',{symbol,type,amount:+amount,price:+price});setMsg(res.data.msg);}
    catch(err:any){setMsg(err.response.data.msg);}
  };
  return(<div>
    <h2 className="text-2xl mb-4">Trade {symbol}</h2>
    <div className="bg-gray-800 p-4 rounded space-y-3 max-w-md">
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Amount" value={amount} onChange={e=>setAmount(e.target.value)}/>
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Price" value={price} onChange={e=>setPrice(e.target.value)}/>
      <div className="flex space-x-2">
        <button onClick={()=>placeOrder('buy')} className="flex-1 bg-green-500 p-2 rounded">Buy</button>
        <button onClick={()=>placeOrder('sell')} className="flex-1 bg-red-500 p-2 rounded">Sell</button>
      </div>
      {msg && <p className="text-yellow-400">{msg}</p>}
    </div>
  </div>);
}
EOF

# Dashboard page
mkdir -p app/dashboard
cat <<'EOF' > app/dashboard/page.tsx
"use client";
import { useEffect,useState } from "react";
import API from "@/lib/api";
export default function Dashboard(){
  const [data,setData]=useState<any>(null);
  useEffect(()=>{API.get('/dashboard').then(res=>setData(res.data));},[]);
  if(!data)return<p>Loading...</p>;
  return(<div>
    <h2 className="text-2xl mb-4">Dashboard</h2>
    <p>Balance: <span className="text-yellow-400">\${data.balance}</span></p>
    <h3 className="mt-4 text-xl">Orders</h3>
    <ul>{data.orders.map((o:any,i:number)=>(<li key={i}>{o.type.toUpperCase()} {o.amount} {o.symbol} @ \${o.price}</li>))}</ul>
  </div>);
}
EOF

cd ../..

############## DOCKER ##############
cat <<EOF > docker-compose.yml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    env_file: ./backend/.env
    depends_on:
      - mongo
  mongo:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
volumes:
  mongo_data:
EOF

echo "✅ Crowd X project created! Fill .env files and run: zip -r crowdx.zip crowdx#!/bin/bash

# Create root folder
mkdir crowdx && cd crowdx

############## BACKEND ##############
mkdir backend && cd backend

# Backend package.json
cat <<EOF > package.json
{
  "name": "crowdx-backend",
  "version": "1.0.0",
  "main": "server.js",
  "type": "module",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.3",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.2",
    "mongoose": "^7.0.3",
    "ws": "^8.12.0"
  }
}
EOF

# Backend .env.example
cat <<EOF > .env.example
PORT=5000
MONGO_URI=mongodb+srv://<username>:<password>@cluster.mongodb.net/crowdx
JWT_SECRET=your_secure_jwt_secret
EOF

# Dockerfile
cat <<EOF > Dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
EOF

echo "node_modules" > .dockerignore
echo ".env" >> .dockerignore

# server.js
cat <<'EOF' > server.js
import express from 'express';
import mongoose from 'mongoose';
import dotenv from 'dotenv';
import cors from 'cors';
import authRoutes from './routes/auth.js';
import marketRoutes from './routes/markets.js';
import orderRoutes from './routes/orders.js';
import dashboardRoutes from './routes/dashboard.js';
import { startPriceFeed } from './wsFeed.js';

dotenv.config();
const app = express();
app.use(cors());
app.use(express.json());
app.use('/api/auth', authRoutes);
app.use('/api/markets', marketRoutes);
app.use('/api/orders', orderRoutes);
app.use('/api/dashboard', dashboardRoutes);

mongoose.connect(process.env.MONGO_URI).then(() => {
  console.log('MongoDB connected');
  const server = app.listen(5000, () => console.log('API running on 5000'));
  startPriceFeed(server);
});
EOF

# Models and Routes
mkdir models routes

# User model
cat <<'EOF' > models/User.js
import mongoose from 'mongoose';
const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  balance: { type: Number, default: 10000 },
  orders: [{ symbol: String, type: String, amount: Number, price: Number }]
});
export default mongoose.model('User', userSchema);
EOF

# Auth route
cat <<'EOF' > routes/auth.js
import express from 'express';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

const router = express.Router();
router.post('/register', async (req,res)=>{
  const {email,password}=req.body;
  const exists=await User.findOne({email});
  if(exists) return res.status(400).json({msg:'User exists'});
  const hash=await bcrypt.hash(password,10);
  const user=await User.create({email,password:hash});
  res.json({msg:'Registered', user:user.email});
});
router.post('/login', async (req,res)=>{
  const {email,password}=req.body;
  const user=await User.findOne({email});
  if(!user) return res.status(400).json({msg:'Invalid credentials'});
  const match=await bcrypt.compare(password,user.password);
  if(!match) return res.status(400).json({msg:'Invalid credentials'});
  const token=jwt.sign({id:user._id},process.env.JWT_SECRET,{expiresIn:'1d'});
  res.json({token});
});
export default router;
EOF

# Markets route
cat <<'EOF' > routes/markets.js
import express from 'express';
const router = express.Router();
let prices=[{symbol:'BTC/USDT',price:50000},{symbol:'ETH/USDT',price:3200}];
router.get('/',(req,res)=>res.json(prices));
export const updatePrices=()=>{prices=prices.map(p=>({...p,price:+(p.price*(1+(Math.random()-0.5)/100)).toFixed(2)}))};
export default router;
EOF

# Orders route
cat <<'EOF' > routes/orders.js
import express from 'express';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';
const router=express.Router();
const auth=(req,res,next)=>{
  const token=req.headers.authorization?.split(' ')[1];
  if(!token) return res.sendStatus(401);
  jwt.verify(token,process.env.JWT_SECRET,(err,user)=>{if(err)return res.sendStatus(403);req.user=user;next();});
};
router.post('/',auth,async(req,res)=>{
  const {symbol,type,amount,price}=req.body;
  const user=await User.findById(req.user.id);
  const order={symbol,type,amount,price};
  user.orders.push(order);
  user.balance-=amount*price;
  await user.save();
  res.json({msg:'Order placed',order});
});
export default router;
EOF

# Dashboard route
cat <<'EOF' > routes/dashboard.js
import express from 'express';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';
const router=express.Router();
const auth=(req,res,next)=>{
  const token=req.headers.authorization?.split(' ')[1];
  if(!token)return res.sendStatus(401);
  jwt.verify(token,process.env.JWT_SECRET,(err,user)=>{if(err)return res.sendStatus(403);req.user=user;next();});
};
router.get('/',auth,async(req,res)=>{
  const user=await User.findById(req.user.id);
  res.json({balance:user.balance,orders:user.orders});
});
export default router;
EOF

# WebSocket price feed
cat <<'EOF' > wsFeed.js
import { WebSocketServer } from 'ws';
import { updatePrices } from './routes/markets.js';
export const startPriceFeed=(server)=>{
  const wss=new WebSocketServer({server});
  setInterval(()=>{
    updatePrices();
    const data=JSON.stringify({event:'prices',data:'updated'});
    wss.clients.forEach(client=>client.send(data));
  },2000);
};
EOF

cd ..

############## FRONTEND ##############
npx create-next-app@latest frontend --typescript --tailwind --eslint --app --src-dir false --import-alias "@/*" <<< $'\nn\n'

cd frontend

# Frontend env example
cat <<EOF > .env.example
NEXT_PUBLIC_API_URL=http://localhost:5000/api
EOF

# API client
mkdir lib
cat <<'EOF' > lib/api.ts
import axios from 'axios';
const API=axios.create({baseURL:process.env.NEXT_PUBLIC_API_URL});
API.interceptors.request.use((config)=>{const token=localStorage.getItem('token');if(token)config.headers.Authorization=\`Bearer \${token}\`;return config;});
export default API;
EOF

# Login page
mkdir -p app/login
cat <<'EOF' > app/login/page.tsx
"use client";
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import API from '@/lib/api';
export default function Login() {
  const [email,setEmail]=useState('');
  const [password,setPassword]=useState('');
  const [error,setError]=useState('');
  const router=useRouter();
  const handleLogin=async(e:any)=>{e.preventDefault();
    try{const res=await API.post('/auth/login',{email,password});
      localStorage.setItem('token',res.data.token);
      router.push('/dashboard');
    }catch(err:any){setError(err.response.data.msg);}
  };
  return(<div className="max-w-md mx-auto mt-10 bg-gray-800 p-6 rounded">
    <h2 className="text-2xl mb-4">Login to Crowd X</h2>
    {error && <p className="text-red-400">{error}</p>}
    <form onSubmit={handleLogin} className="space-y-4">
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)}/>
      <input type="password" className="w-full p-2 bg-gray-700 rounded" placeholder="Password" value={password} onChange={e=>setPassword(e.target.value)}/>
      <button className="w-full bg-yellow-400 p-2 rounded text-black">Login</button>
    </form>
  </div>);
}
EOF

# Register page
mkdir -p app/register
cat <<'EOF' > app/register/page.tsx
"use client";
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import API from '@/lib/api';
export default function Register() {
  const [email,setEmail]=useState('');
  const [password,setPassword]=useState('');
  const [msg,setMsg]=useState('');
  const router=useRouter();
  const handleRegister=async(e:any)=>{e.preventDefault();
    try{await API.post('/auth/register',{email,password});
      setMsg('Registered! Redirecting...');
      setTimeout(()=>router.push('/login'),1500);
    }catch(err:any){setMsg(err.response.data.msg);}
  };
  return(<div className="max-w-md mx-auto mt-10 bg-gray-800 p-6 rounded">
    <h2 className="text-2xl mb-4">Register on Crowd X</h2>
    {msg && <p className="text-yellow-400">{msg}</p>}
    <form onSubmit={handleRegister} className="space-y-4">
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Email" value={email} onChange={e=>setEmail(e.target.value)}/>
      <input type="password" className="w-full p-2 bg-gray-700 rounded" placeholder="Password" value={password} onChange={e=>setPassword(e.target.value)}/>
      <button className="w-full bg-yellow-400 p-2 rounded text-black">Register</button>
    </form>
  </div>);
}
EOF

# Markets page
mkdir -p app/markets
cat <<'EOF' > app/markets/page.tsx
"use client";
import { useEffect,useState } from "react";
import API from "@/lib/api";
export default function Markets(){
  const [prices,setPrices]=useState<any[]>([]);
  useEffect(()=>{
    const fetchPrices=async()=>{const res=await API.get('/markets');setPrices(res.data);};
    fetchPrices();
    const ws=new WebSocket('ws://localhost:5000');
    ws.onmessage=(msg)=>{if(JSON.parse(msg.data).event==='prices')fetchPrices();};
    return()=>ws.close();
  },[]);
  return(<div>
    <h2 className="text-2xl mb-4">Markets</h2>
    <table className="w-full text-left border border-gray-700">
      <thead><tr className="border-b border-gray-700"><th>Symbol</th><th>Price</th></tr></thead>
      <tbody>{prices.map((p,i)=>(
        <tr key={i} className="border-b border-gray-700"><td>{p.symbol}</td><td className="text-yellow-400">\${p.price}</td></tr>
      ))}</tbody>
    </table>
  </div>);
}
EOF

# Trade page
mkdir -p app/trade/[symbol]
cat <<'EOF' > app/trade/[symbol]/page.tsx
"use client";
import { useParams } from "next/navigation";
import { useState } from "react";
import API from "@/lib/api";
export default function Trade(){
  const {symbol}=useParams();
  const [amount,setAmount]=useState('');
  const [price,setPrice]=useState('');
  const [msg,setMsg]=useState('');
  const placeOrder=async(type:string)=>{
    try{const res=await API.post('/orders',{symbol,type,amount:+amount,price:+price});setMsg(res.data.msg);}
    catch(err:any){setMsg(err.response.data.msg);}
  };
  return(<div>
    <h2 className="text-2xl mb-4">Trade {symbol}</h2>
    <div className="bg-gray-800 p-4 rounded space-y-3 max-w-md">
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Amount" value={amount} onChange={e=>setAmount(e.target.value)}/>
      <input className="w-full p-2 bg-gray-700 rounded" placeholder="Price" value={price} onChange={e=>setPrice(e.target.value)}/>
      <div className="flex space-x-2">
        <button onClick={()=>placeOrder('buy')} className="flex-1 bg-green-500 p-2 rounded">Buy</button>
        <button onClick={()=>placeOrder('sell')} className="flex-1 bg-red-500 p-2 rounded">Sell</button>
      </div>
      {msg && <p className="text-yellow-400">{msg}</p>}
    </div>
  </div>);
}
EOF

# Dashboard page
mkdir -p app/dashboard
cat <<'EOF' > app/dashboard/page.tsx
"use client";
import { useEffect,useState } from "react";
import API from "@/lib/api";
export default function Dashboard(){
  const [data,setData]=useState<any>(null);
  useEffect(()=>{API.get('/dashboard').then(res=>setData(res.data));},[]);
  if(!data)return<p>Loading...</p>;
  return(<div>
    <h2 className="text-2xl mb-4">Dashboard</h2>
    <p>Balance: <span className="text-yellow-400">\${data.balance}</span></p>
    <h3 className="mt-4 text-xl">Orders</h3>
    <ul>{data.orders.map((o:any,i:number)=>(<li key={i}>{o.type.toUpperCase()} {o.amount} {o.symbol} @ \${o.price}</li>))}</ul>
  </div>);
}
EOF

cd ../..

############## DOCKER ##############
cat <<EOF > docker-compose.yml
version: '3.8'
services:
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    env_file: ./backend/.env
    depends_on:
      - mongo
  mongo:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo_data:/data/db
volumes:
  mongo_data:
EOF

echo "✅ Crowd X project created! Fill .env files and run: zip -r crowdx.zip crowdx
