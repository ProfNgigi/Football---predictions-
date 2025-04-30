# Football---predictions-
backend/
  ├── config/
  │   └── db.js
  ├── models/
  │   ├── User.js
  │   └── Transaction.js
  ├── routes/
  │   ├── auth.js
  │   ├── admin.js
  │   └── predictions.js
  ├── middleware/
  │   ├── auth.js
  │   └── admin.js
  └── server.js
  import mongoose from 'mongoose';

const connectDB = async () => {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log('MongoDB connected');
  } catch (err) {
    console.error(err.message);
    process.exit(1);
  }
};

export default connectDB;
// User.js
import mongoose from 'mongoose';

const UserSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  isVIP: { type: Boolean, default: false },
  vipExpiresAt: Date,
  isAdmin: { type: Boolean, default: false }
});

export default mongoose.model('User', UserSchema);

// Transaction.js
const TransactionSchema = new mongoose.Schema({
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  code: { type: String, required: true, unique: true },
  durationDays: Number,
  verified: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now }
});

export default mongoose.model('Transaction', TransactionSchema);
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

export const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization').replace('Bearer ', '');
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);

    if (!user) throw new Error();
    req.user = user;
    next();
  } catch (err) {
    res.status(401).send({ error: 'Please authenticate' });
  }
};

export const isVIP = async (req, res, next) => {
  if (!req.user.isVIP || req.user.vipExpiresAt < new Date()) {
    return res.status(403).send({ error: 'VIP access required' });
  }
  next();
};
import express from 'express';
import Transaction from '../models/Transaction.js';
import { auth, isAdmin } from '../middleware/auth.js';

const router = express.Router();

// Get pending transactions
router.get('/transactions', auth, isAdmin, async (req, res) => {
  const transactions = await Transaction.find({ verified: false });
  res.send(transactions);
});

// Approve transaction
router.post('/transactions/approve', auth, isAdmin, async (req, res) => {
  const { transactionId, durationDays } = req.body;
  
  const transaction = await Transaction.findById(transactionId);
  const user = await User.findById(transaction.userId);

  user.isVIP = true;
  user.vipExpiresAt = new Date(Date.now() + durationDays * 86400000);
  transaction.verified = true;
  transaction.durationDays = durationDays;

  await user.save();
  await transaction.save();

  res.send({ message: 'VIP access granted' });
});

export default router;
import React, { useState } from 'react';
import axios from 'axios';

const VIPVerification = ({ user }) => {
  const [code, setCode] = useState('');

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      await axios.post('/api/transactions', { code, userId: user._id });
      alert('Code submitted for verification!');
    } catch (err) {
      alert('Submission failed');
    }
  };

  return (
    <div>
      <h2>Verify VIP Access</h2>
      <form onSubmit={handleSubmit}>
        <input
          type="text"
          value={code}
          onChange={(e) => setCode(e.target.value)}
          placeholder="Enter transaction code"
        />
        <button type="submit">Submit</button>
      </form>
    </div>
  );
};
const AdminDashboard = () => {
  const [transactions, setTransactions] = useState([]);
  const [duration, setDuration] = useState(30);

  useEffect(() => {
    const fetchTransactions = async () => {
      const res = await axios.get('/api/admin/transactions');
      setTransactions(res.data);
    };
    fetchTransactions();
  }, []);

  const approveTransaction = async (transactionId) => {
    await axios.post('/api/admin/transactions/approve', {
      transactionId,
      durationDays: duration
    });
    // Refresh list
  };

  return (
    <div>
      <h1>Pending Verifications</h1>
      <select onChange={(e) => setDuration(e.target.value)}>
        <option value="7">7 Days</option>
        <option value="30">30 Days</option>
        <option value="90">90 Days</option>
      </select>
      
      {transactions.map((txn) => (
        <div key={txn._id}>
          <p>Code: {txn.code}</p>
          <button onClick={() => approveTransaction(txn._id)}>
            Approve for {duration} days
          </button>
        </div>
      ))}
    </div>
  );
};
import cron from 'node-cron';
import User from './models/User.js';

// Run daily at midnight
cron.schedule('0 0 * * *', async () => {
  const expiredUsers = await User.find({
    isVIP: true,
    vipExpiresAt: { $lt: new Date() }
  });

  expiredUsers.forEach(async (user) => {
    user.isVIP = false;
    await user.save();
  });
  console.log(`Expired ${expiredUsers.length} VIP memberships`);
});
const mongoose = require('mongoose');
const User = require('../models/User');

// Connect to MongoDB
mongoose.connect(process.env.MONGO_URI);

// Find and expire VIP users
async function expireVIPs() {
  const expiredUsers = await User.find({
    isVIP: true,
    vipExpiresAt: { $lt: new Date() }
  });

  expiredUsers.forEach(async (user) => {
    user.isVIP = false;
    await user.save();
    console.log(`Expired VIP for ${user.email}`);
  });

  console.log('Done!');
  process.exit();
}

expireVIPs();
