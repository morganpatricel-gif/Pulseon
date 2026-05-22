# Pulseon
Pulseon App
const express = require("express");
const { Pool } = require("pg");
const Stripe = require("stripe");

const app = express();
app.use(express.json());

const db = new Pool({
  connectionString: process.env.DATABASE_URL
});

const stripe = new Stripe(process.env.STRIPE_SECRET);

// ----------------------
// HEALTH + MONITORING
// ----------------------
app.get("/", (req, res) => {
  res.json({ status: "PRODUCTION ONLINE" });
});

// ----------------------
// AUTH (MINIMAL JWT PLACEHOLDER)
// ----------------------
app.post("/auth", (req, res) => {
  const { email } = req.body;
  res.json({ token: "token-" + email });
});

// ----------------------
// CREATE BEAT
// ----------------------
app.post("/beats", async (req, res) => {
  const { title, price, creator_id } = req.body;

  const result = await db.query(
    "INSERT INTO beats (title, price, creator_id) VALUES ($1,$2,$3) RETURNING *",
    [title, price, creator_id]
  );

  res.json(result.rows[0]);
});

// ----------------------
// GET FEED (VIRAL RANKING)
// ----------------------
app.get("/feed", async (req, res) => {
  const result = await db.query("SELECT * FROM beats");

  const ranked = result.rows.sort((a, b) => {
    return (b.plays || 0) + (b.shares || 0) * 4 - ((a.plays || 0) + (a.shares || 0) * 4);
  });

  res.json(ranked);
});

// ----------------------
// STRIPE CONNECT PAYMENT
// ----------------------
app.post("/buy", async (req, res) => {
  const { beat_id } = req.body;

  const beat = await db.query("SELECT * FROM beats WHERE id=$1", [beat_id]);

  const price = beat.rows[0].price;

  const session = await stripe.checkout.sessions.create({
    mode: "payment",
    payment_method_types: ["card"],
    line_items: [{
      price_data: {
        currency: "usd",
        product_data: { name: beat.rows[0].title },
        unit_amount: price * 100
      },
      quantity: 1
    }],
    success_url: "https://yourapp.com/success",
    cancel_url: "https://yourapp.com/cancel"
  });

  res.json(session);
});

// ----------------------
// STRIPE WEBHOOK (LEDGER)
// ----------------------
app.post("/webhook", express.raw({ type: "application/json" }), async (req, res) => {
  const event = JSON.parse(req.body);

  if (event.type === "checkout.session.completed") {
    const amount = event.data.object.amount_total / 100;

    await db.query(
      "INSERT INTO transactions (type, amount) VALUES ($1,$2)",
      ["sale", amount]
    );
  }

  res.sendStatus(200);
});

// ----------------------
// LEDGER EXPORT (TAX READY)
// ----------------------
app.get("/ledger", async (req, res) => {
  const result = await db.query("SELECT * FROM transactions");
  res.json(result.rows);
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log("API running")); 
