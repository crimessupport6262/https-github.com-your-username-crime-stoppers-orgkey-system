// 📁 /org-key-system

// ========== FRONTEND ========== // File: frontend/src/App.jsx import { useState } from "react"; import OrgKeyAdminPanel from "./OrgKeyAdminPanel"; import Login from "./Login";

export default function App() { const [token, setToken] = useState(localStorage.getItem("token") || "");

return token ? ( <OrgKeyAdminPanel token={token} /> ) : ( <Login setToken={setToken} /> ); }

// File: frontend/src/Login.jsx import { useState } from "react";

export default function Login({ setToken }) { const [email, setEmail] = useState(""); const [password, setPassword] = useState("");

const handleLogin = async () => { const res = await fetch("https://your-backend-url.onrender.com/api/login", { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ email, password }), }); const data = await res.json(); if (data.token) { localStorage.setItem("token", data.token); setToken(data.token); } };

return ( <div className="p-6 max-w-md mx-auto"> <h2 className="text-xl font-bold mb-4">Admin Login</h2> <input type="email" placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} className="border rounded p-2 w-full mb-2" /> <input type="password" placeholder="Password" value={password} onChange={(e) => setPassword(e.target.value)} className="border rounded p-2 w-full mb-2" /> <button onClick={handleLogin} className="bg-blue-600 text-white px-4 py-2 rounded"> Login </button> </div> ); }

// File: frontend/src/OrgKeyAdminPanel.jsx import { useState } from "react";

export default function OrgKeyAdminPanel({ token }) { const [orgKey, setOrgKey] = useState(""); const [recipient, setRecipient] = useState("");

const generateKey = async () => { const res = await fetch("https://your-backend-url.onrender.com/api/generate-org-key", { method: "POST", headers: { "Content-Type": "application/json", Authorization: Bearer ${token}, }, body: JSON.stringify({ recipient }), }); const data = await res.json(); setOrgKey(data.key); };

return ( <div className="p-6 max-w-md mx-auto"> <h1 className="text-2xl font-bold mb-4">Organization Key Generator</h1> <input type="email" placeholder="Recipient Email" value={recipient} onChange={(e) => setRecipient(e.target.value)} className="border rounded p-2 w-full mb-2" /> <button onClick={generateKey} className="bg-green-600 text-white px-4 py-2 rounded"> Generate Key </button> {orgKey && ( <div className="mt-4 p-4 bg-gray-100 rounded shadow"> <strong>Generated Key:</strong> <span className="text-blue-600">{orgKey}</span> </div> )} </div> ); }

// ========== BACKEND ========== // File: backend/server.js const express = require("express"); const mongoose = require("mongoose"); const cors = require("cors"); const dotenv = require("dotenv"); const authRoutes = require("./routes/auth"); const keyRoutes = require("./routes/generate-org-key");

dotenv.config(); const app = express();

app.use(cors()); app.use(express.json()); app.use("/api", authRoutes); app.use("/api", keyRoutes);

mongoose.connect(process.env.MONGO_URI, () => { console.log("Connected to MongoDB"); app.listen(process.env.PORT || 3000, () => console.log("Server running")); });

// File: backend/routes/auth.js const express = require("express"); const jwt = require("jsonwebtoken"); const router = express.Router();

router.post("/login", (req, res) => { const { email, password } = req.body; if ( email === process.env.ADMIN_EMAIL && password === process.env.ADMIN_PASSWORD ) { const token = jwt.sign({ email }, process.env.JWT_SECRET, { expiresIn: "1h", }); return res.json({ token }); } res.status(401).json({ message: "Unauthorized" }); });

module.exports = router;

// File: backend/routes/generate-org-key.js const express = require("express"); const jwt = require("jsonwebtoken"); const OrgKey = require("../models/OrgKey"); const router = express.Router(); const nodemailer = require("nodemailer");

function generateOrgKey() { const date = new Date().toISOString().slice(0, 10).replace(/-/g, ""); const chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"; const randomPart = Array.from({ length: 8 }, () => chars.charAt(Math.floor(Math.random() * chars.length)) ).join(""); return ORG-${date}-${randomPart}; }

const authMiddleware = (req, res, next) => { const auth = req.headers.authorization; if (!auth) return res.status(401).json({ error: "Unauthorized" }); try { const token = auth.split(" ")[1]; jwt.verify(token, process.env.JWT_SECRET); next(); } catch (err) { res.status(401).json({ error: "Invalid token" }); } };

router.post("/generate-org-key", authMiddleware, async (req, res) => { const { recipient } = req.body; const key = generateOrgKey(); await OrgKey.create({ key, recipient });

const transporter = nodemailer.createTransport({ service: "gmail", auth: { user: process.env.EMAIL_FROM, pass: process.env.EMAIL_PASSWORD, }, });

await transporter.sendMail({ from: Crime Stoppers USA <${process.env.EMAIL_FROM}>, to: recipient, subject: "Your Organization Key - Crime Stoppers USA", html: <h2>🛡 Welcome to Crime Stoppers USA</h2> <p><b>Your Organization Key:</b> ${key}</p> <p>Use this key to access your secure resources.</p> });

res.json({ key }); });

module.exports = router;

// File: backend/models/OrgKey.js const mongoose = require("mongoose"); const OrgKeySchema = new mongoose.Schema({ key: String, recipient: String, createdAt: { type: Date, default: Date.now }, }); module.exports = mongoose.model("OrgKey", OrgKeySchema);

// File: backend/.env.example PORT=3000 MONGO_URI=your_mongodb_uri_here ADMIN_EMAIL=admin@example.com ADMIN_PASSWORD=supersecurepassword JWT_SECRET=supersecretkey EMAIL_FROM=your_gmail@gmail.com EMAIL_PASSWORD=your_gmail_app_password
