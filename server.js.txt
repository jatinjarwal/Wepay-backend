// WePay - Group Expense Tracker Backend 

const express = require('express');
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const dotenv = require('dotenv');
const cors = require('cors');
const { body, validationResult } = require('express-validator');

dotenv.config();
const app = express();
app.use(express.json());
app.use(cors());

// Connect to MongoDB
mongoose.connect(process.env.MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
}).then(() => console.log('MongoDB connected')).catch(err => console.log('Mongo error:', err));

// User Schema & Model
const userSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  password: String,
}, { timestamps: true });
const User = mongoose.model('User', userSchema);

// Group Schema & Model
const groupSchema = new mongoose.Schema({
  name: String,
  members: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
  expenses: [{
    description: String,
    amount: Number,
    category: String,
    paidBy: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
    splitBetween: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
    createdAt: { type: Date, default: Date.now }
  }]
}, { timestamps: true });
const Group = mongoose.model('Group', groupSchema);

// Auth middleware
const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded._id);
    if (!user) throw new Error();
    req.user = user;
    next();
  } catch {
    res.status(401).send({ error: 'Authentication failed' });
  }
};

// Signup
app.post('/signup', [
  body('name').notEmpty().trim().escape(),
  body('email').isEmail(),
  body('password').isLength({ min: 6 })
], async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

  try {
    const { name, email, password } = req.body;
    const hashedPassword = await bcrypt.hash(password, 10);
    const user = new User({ name, email, password: hashedPassword });
    await user.save();
    res.status(201).send({ message: 'Signup successful' });
  } catch (err) {
    res.status(400).send({ error: 'Signup error', detail: err });
  }
});

// Login
app.post('/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });
  if (!user || !(await bcrypt.compare(password, user.password)))
    return res.status(400).send({ error: 'Invalid credentials' });

  const token = jwt.sign({ _id: user._id }, process.env.JWT_SECRET, { expiresIn: '7d' });
  res.send({ token });
});

// Create a Group
app.post('/groups', auth, async (req, res) => {
  const { name, members } = req.body;
  const newGroup = new Group({ name, members });
  await newGroup.save();
  res.status(201).send(newGroup);
});

// Add Expense with Duplicate Check
app.post('/groups/:id/expenses', auth, async (req, res) => {
  const group = await Group.findById(req.params.id);
  if (!group) return res.status(404).send({ error: 'Group not found' });

  const { description, amount, category, paidBy, splitBetween } = req.body;

  const duplicate = group.expenses.find(exp =>
    exp.description === description &&
    exp.amount === amount &&
    exp.paidBy.toString() === paidBy &&
    new Date(exp.createdAt).toDateString() === new Date().toDateString()
  );
  if (duplicate) return res.status(409).send({ error: 'Duplicate expense detected' });

  group.expenses.push({ description, amount, category, paidBy, splitBetween });
  await group.save();
  res.send(group);
});

// Get Group Balances
app.get('/groups/:id/balances', auth, async (req, res) => {
  const group = await Group.findById(req.params.id)
    .populate('members')
    .populate('expenses.paidBy')
    .populate('expenses.splitBetween');

  if (!group) return res.status(404).send({ error: 'Group not found' });

  const balances = {};
  for (const expense of group.expenses) {
    const share = expense.amount / expense.splitBetween.length;
    for (const member of expense.splitBetween) {
      if (member._id.toString() === expense.paidBy._id.toString()) continue;
      balances[member._id] = balances[member._id] || {};
      balances[member._id][expense.paidBy._id] =
        (balances[member._id][expense.paidBy._id] || 0) + share;
    }
  }
  res.send({ group: group.name, members: group.members, balances });
});

// Launch Server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`WePay backend running on port ${PORT}`));
