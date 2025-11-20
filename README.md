const express = require("express");
const mongoose = require("mongoose");
const bcrypt = require("bcryptjs");
const jwt = require("jsonwebtoken");
const app = express();
app.use(express.json());

mongoose.connect("mongodb://localhost/medicine_app", { useNewUrlParser: true, useUnifiedTopology: true });

// User Schema
const userSchema = new mongoose.Schema({
  username: String,
  password: String,
  cart: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Medicine' }]
});
const User = mongoose.model("User", userSchema);

// Medicine Schema
const medicineSchema = new mongoose.Schema({
  name: String,
  price: Number,
  stock: Number
});
const Medicine = mongoose.model("Medicine", medicineSchema);

// Order Schema
const orderSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  medicines: [{
    medicine: { type: mongoose.Schema.Types.ObjectId, ref: 'Medicine' },
    quantity: Number
  }],
  status: { type: String, default: "pending" },
  createdAt: { type: Date, default: Date.now }
});
const Order = mongoose.model("Order", orderSchema);

// Registration
app.post("/register", async (req, res) => {
  const { username, password } = req.body;
  const hash = await bcrypt.hash(password, 10);
  const user = new User({ username, password: hash });
  await user.save();
  res.send({ message: "Registration successful" });
});

// Login
app.post("/login", async (req, res) => {
  const { username, password } = req.body;
  const user = await User.findOne({ username });
  if (user && await bcrypt.compare(password, user.password)) {
    const token = jwt.sign({ userId: user._id }, "SECRET_KEY");
    res.send({ token });
  } else {
    res.status(401).send({ message: "Invalid credentials" });
  }
});

// Auth Middleware
function auth(req, res, next) {
  const token = req.headers.authorization;
  if (!token) return res.status(403).send({ message: "No token" });
  try {
    const decoded = jwt.verify(token, "SECRET_KEY");
    req.userId = decoded.userId;
    next();
  } catch {
    res.status(401).send({ message: "Invalid token" });
  }
}

// Medicine List & Search
app.get("/medicines", async (req, res) => {
  const { q } = req.query;
  const filter = q ? { name: { $regex: q, $options: 'i' } } : {};
  const medicines = await Medicine.find(filter);
  res.send(medicines);
});

// Add to Cart
app.post("/cart", auth, async (req, res) => {
  const { medicineId } = req.body;
  const user = await User.findById(req.userId);
  user.cart.push(medicineId);
  await user.save();
  res.send({ message: "Added to cart" });
});

// Place Order (Cash on Delivery)
app.post("/order", auth, async (req, res) => {
  const user = await User.findById(req.userId).populate("cart");
  if (user.cart.length === 0) return res.status(400).send({ message: "Cart is empty" });
  // Map cart to medicine + quantity
  const order = new Order({
    user: user._id,
    medicines: user.cart.map(id => ({ medicine: id, quantity: 1 })),
    status: "pending"
  });
  await order.save();
  user.cart = [];
  await user.save();
  res.send({ message: "Order placed" });
});

// Order History
app.get("/orders", auth, async (req, res) => {
  const orders = await Order.find({ user: req.userId }).populate("medicines.medicine");
  res.send(orders);
});

// Server Start
app.listen(3000, () => console.log("Server running on port 3000"));# MDH
It is a pharmacy delivery apps
