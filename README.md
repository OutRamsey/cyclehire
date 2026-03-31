# cyclehire
Cycle Hire
# cyclehire
node_modules/
dist/
widget/node_modules/
widget/build/
.env
.env.local
.netlify/
.vite
.DS_Store
Bike Rental Widget (Netlify + React) - Quickstart

What this is
- A lightweight React widget that runs in an iframe on BigCommerce.
- Allows multiple bikes per order, per-item size, and per-item quantity.
- Uses a single shared date range (startDate, endDate) for the whole cart.
- PayPal checkout via a server-side create-order and capture-order flow.
- Data stored in MySQL (bikes, stock, bookings).

What’s included
- Frontend: React app (widget/) built with Vite
- Backend: Netlify Functions (netlify/functions/)
  - bikes.js
  - availability.js
  - paypal-create-order.js
  - paypal-capture-order.js
  - db.js (MySQL pool)
- MySQL schema and seed data (sql/)
- Netlify configuration (netlify.toml)

How to deploy (recommended)
1) Create a GitHub repo and push this code (as-is or your customization).
2) In Netlify:
   - New site from Git → connect to your GitHub repo.
   - Build: command: cd widget && npm ci && npm run build
   - Publish: widget/build
   - Functions: netlify/functions
   - Environment vars (set later in Netlify UI):
     - DB_HOST, DB_PORT, DB_USER, DB_PASSWORD, DB_NAME
     - PAYPAL_CLIENT_ID, PAYPAL_CLIENT_SECRET
     - PAYPAL_BASE (https://api-m.paypal.com or https://api-m.sandbox.paypal.com)
     - PAYPAL_USE_SANDBOX (true/false)
3) Seed your MySQL DB with the schema and seed data.
4) Get the Netlify widget URL and embed it as an iframe in BigCommerce.

Local development notes
- Netlify CLI can run the widget locally with functions: npm i -g netlify-cli; netlify dev
- You’ll still need DB connection details when testing locally.

Embedding on BigCommerce
- After deployment, copy the Netlify widget URL, e.g. https://yourname.netlify.app/widget
- Use an iframe in BigCommerce:
  <iframe src="https://yourname.netlify.app/widget" title="Bike Rental" width="100%" height="800" frameborder="0"
          sandbox="allow-scripts allow-forms allow-same-origin"></iframe>

If you want, I can tailor this starter to your exact branding/colors and provide a deployment guide with exact commands.

[build]
  command = "cd widget && npm ci && npm run build"
  publish = "widget/build"

[build.environment]
  # You can set defaults; override in Netlify UI
  PAYPAL_BASE = "https://api-m.paypal.com"
  PAYPAL_CLIENT_ID = ""
  PAYPAL_CLIENT_SECRET = ""
  DB_HOST = ""
  DB_PORT = "3306"
  DB_USER = ""
  DB_PASSWORD = ""
  DB_NAME = ""
  PAYPAL_USE_SANDBOX = "true"

[functions]
  directory = "netlify/functions"
  node_bundler = "esbuild"
// netlify/functions/db.js
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT || 3306,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  connectionLimit: 10,
  // enable multiple statements if you want to seed or run batch queries (optional)
  // multipleStatements: true
});

module.exports = pool;
// netlify/functions/bikes.js
const pool = require('./db');

exports.handler = async (event) => {
  try {
    const [rows] = await pool.query(
      `SELECT id, model_name AS modelName, price_per_day AS pricePerDay, image_url AS imageUrl, description
       FROM bikes`
    );
    return {
      statusCode: 200,
      body: JSON.stringify(rows)
    };
  } catch (err) {
    console.error(err);
    return { statusCode: 500, body: JSON.stringify({ error: 'Internal error' }) };
  }
};
// netlify/functions/availability.js
const pool = require('./db');

async function itemAvailable(bikeId, size, startDate, endDate, requestedQty) {
  const [stockRows] = await pool.query(
    `SELECT stock FROM stock WHERE bike_id = ? AND size = ?`, [bikeId, size]
  );
  const stock = stockRows.length ? stockRows<a href="" class="citation-link" target="_blank" style="vertical-align: super; font-size: 0.8em; margin-left: 3px;">[0]</a>.stock : 0;

  const [sumRows] = await pool.query(
    `SELECT COALESCE(SUM(quantity),0) AS reserved
     FROM bookings
     WHERE bike_id = ? AND size = ?
       AND end_date >= ? AND start_date <= ?
       AND status IN ('pending','confirmed')`, [bikeId, size, startDate, endDate]
  );
  const reserved = sumRows<a href="" class="citation-link" target="_blank" style="vertical-align: super; font-size: 0.8em; margin-left: 3px;">[0]</a>.reserved;

  const available = stock - reserved;
  return available >= (requestedQty || 1);
}

exports.handler = async (event) => {
  try {
    const body = JSON.parse(event.body || '{}');
    const { items, startDate, endDate } = body; // items: [{ bikeId, size, quantity }]

    const results = await Promise.all(items.map(async (it) => {
      const ok = await itemAvailable(it.bikeId, it.size, startDate, endDate, it.quantity);
      const [stockRows] = await pool.query(
        `SELECT stock FROM stock WHERE bike_id = ? AND size = ?`, [it.bikeId, it.size]
      );
      const stock = stockRows.length ? stockRows<a href="" class="citation-link" target="_blank" style="vertical-align: super; font-size: 0.8em; margin-left: 3px;">[0]</a>.stock : 0;
      return { bikeId: it.bikeId, size: it.size, quantity: it.quantity, available: ok, stock };
    }));

    return { statusCode: 200, body: JSON.stringify({ items: results }) };
  } catch (err) {
    console.error(err);
    return { statusCode: 500, body: JSON.stringify({ error: 'Availability check failed' }) };
  }
};
// netlify/functions/paypal-create-order.js
// Node 18 environment; using global fetch (no extra deps)
const pool = require('./db');

async function getPaypalToken() {
  const base = process.env.PAYPAL_BASE || 'https://api-m.paypal.com';
  const clientId = process.env.PAYPAL_CLIENT_ID;
  const clientSecret = process.env.PAYPAL_CLIENT_SECRET;
  const tokenRes = await fetch(`${base}/v1/oauth2/token`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
      'Authorization': 'Basic ' + Buffer.from(clientId + ':' + clientSecret).toString('base64')
    },
    body: 'grant_type=client_credentials'
  });
  const tokenData = await tokenRes.json();
  return tokenData.access_token;
}

async function createPaypalOrder(items, startDate, endDate, currency = 'USD') {
  // Calculate days
  const days = Math.ceil((new Date(endDate) - new Date(startDate)) / (1000 * 60 * 60 * 24));
  // Enrich items with bike data
  const bikeIds = [...new Set(items.map(i => i.bikeId))];
  const [rows] = await pool.query(`SELECT id, model_name, price_per_day FROM bikes WHERE id IN (?)`, [bikeIds]);
  const bikeMap = {};
  rows.forEach(r => { bikeMap[r.id] = { modelName: r.model_name, pricePerDay: r.price_per_day }; });

  // Build PayPal line items
  const lineItems = items.map(it => {
    const b = bikeMap[it.bikeId] || { modelName: 'Bike', pricePerDay: 0 };
    const unitValue = parseFloat(b.pricePerDay) * days;
    return {
      name: `${b.modelName} - ${it.size}`,
      unit_amount: { currency_code: currency, value: unitValue.toFixed(2) },
      quantity: String(it.quantity)
    };
  });

  const total = lineItems.reduce((sum, li) => sum + parseFloat(li.unit_amount.value) * parseInt(li.quantity, 10), 0).toFixed(2);

  const purchaseUnits = [{
    description: 'Bike rental',
    amount: { currency_code: currency, value: total, breakdown: { item_total: { currency_code: currency, value: total } } },
    items: lineItems
  }];

  const token = await getPaypalToken();
  const base = process.env.PAYPAL_BASE || 'https://api-m.paypal.com';
  const res = await fetch(`${base}/v2/checkout/orders`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({ intent: 'CAPTURE', purchase_units: purchaseUnits })
  });
  const data = await res.json();
  return { orderID: data.id, total };
}

exports.handler = async (event) => {
  try {
    const body = JSON.parse(event.body || '{}');
    const { items, startDate, endDate, currency } = body;
    const { orderID, total } = await createPaypalOrder(items, startDate, endDate, currency || 'USD');
    return {
      statusCode: 200,
      body: JSON.stringify({ orderID, total })
    };
  } catch (err) {
    console.error(err);
    return { statusCode: 500, body: JSON.stringify({ error: 'PayPal order creation failed' }) };
  }
};
// netlify/functions/paypal-capture-order.js
const pool = require('./db');
const fetch = require('node-fetch'); // if your environment supports global fetch, you can remove this line

async function
