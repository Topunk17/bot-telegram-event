require('dotenv').config()
const TelegramBot = require('node-telegram-bot-api')
const mysql = require('mysql2/promise')

const bot = new TelegramBot(process.env.BOT_TOKEN, { polling: true })

// DB
const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASS,
  database: process.env.DB_NAME
})

const ADMIN_ID = process.env.ADMIN_ID

// ================== AUTO CREATE TABLE ==================
async function initDB() {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS users (
      id INT AUTO_INCREMENT PRIMARY KEY,
      telegram_id BIGINT,
      saldo INT DEFAULT 0
    )
  `)

  await pool.query(`
    CREATE TABLE IF NOT EXISTS events (
      id INT AUTO_INCREMENT PRIMARY KEY,
      title VARCHAR(255),
      reward INT,
      status VARCHAR(20) DEFAULT 'active'
    )
  `)

  await pool.query(`
    CREATE TABLE IF NOT EXISTS submissions (
      id INT AUTO_INCREMENT PRIMARY KEY,
      user_id BIGINT,
      event_id INT,
      link TEXT,
      status VARCHAR(20) DEFAULT 'pending'
    )
  `)

  await pool.query(`
    CREATE TABLE IF NOT EXISTS withdrawals (
      id INT AUTO_INCREMENT PRIMARY KEY,
      user_id BIGINT,
      amount INT,
      metode VARCHAR(50),
      tujuan VARCHAR(100),
      status VARCHAR(20) DEFAULT 'pending'
    )
  `)

  await pool.query(`
    CREATE TABLE IF NOT EXISTS transactions (
      id INT AUTO_INCREMENT PRIMARY KEY,
      user_id BIGINT,
      type VARCHAR(20),
      amount INT,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `)

  console.log("✅ Database & tabel siap")
}

// ================== INIT ==================
initDB().then(() => {
  console.log("🚀 Bot siap jalan")
}).catch(err => {
  console.log("❌ DB Error:", err)
})

// ================== UTIL ==================
const isAdmin = (id) => String(id) === String(ADMIN_ID)

async function getUser(id) {
  const [u] = await pool.query("SELECT * FROM users WHERE telegram_id=?", [id])
  if (!u.length) {
    await pool.query("INSERT INTO users (telegram_id) VALUES (?)", [id])
  }
}

function box(title, body="") {
  return `┏━━━━━━━━━━━━━━━┓\n┃ ${title}\n┗━━━━━━━━━━━━━━━┛\n${body}`
}

// ================== START ==================
bot.onText(/\/start/, async (msg) => {
  await getUser(msg.chat.id)

  bot.sendMessage(msg.chat.id, "👋 Welcome", {
    reply_markup: {
      keyboard: [
        ["📋 Event"],
        ["💰 Saldo", "💸 Withdraw"],
        ["📊 Riwayat"]
      ],
      resize_keyboard: true
    }
  })
})

// ================== EVENT ==================
bot.onText(/📋 Event/, async (msg) => {
  const [events] = await pool.query("SELECT * FROM events WHERE status='active'")

  let text = "📢 Event:\n\n"
  events.forEach(e => {
    text += `${e.id}. ${e.title}\n💰 ${e.reward}\n\n`
  })

  bot.sendMessage(msg.chat.id, text || "Belum ada event")
})

// ================== SUBMIT ==================
bot.on('message', async (msg) => {
  const text = msg.text
  if (!text || !text.includes("facebook.com")) return

  const [event] = await pool.query("SELECT * FROM events WHERE status='active' LIMIT 1")
  if (!event.length) return

  await pool.query(
    "INSERT INTO submissions (user_id, event_id, link) VALUES (?, ?, ?)",
    [msg.chat.id, event[0].id, text]
  )

  bot.sendMessage(msg.chat.id, "✅ Submit berhasil")
})

// ================== SALDO ==================
bot.onText(/💰 Saldo/, async (msg) => {
  const [[u]] = await pool.query("SELECT saldo FROM users WHERE telegram_id=?", [msg.chat.id])
  bot.sendMessage(msg.chat.id, `💰 Saldo: ${u?.saldo || 0}`)
})

// ================== WITHDRAW ==================
const wdState = {}

bot.onText(/💸 Withdraw/, async (msg) => {
  const [[u]] = await pool.query("SELECT saldo FROM users WHERE telegram_id=?", [msg.chat.id])

  wdState[msg.chat.id] = { step: 1 }

  bot.sendMessage(msg.chat.id, `Saldo: ${u?.saldo || 0}\nMasukkan jumlah (min 10000):`)
})

bot.on('message', async (msg) => {
  const st = wdState[msg.chat.id]
  if (!st) return

  if (st.step === 1) {
    const amount = parseInt(msg.text)

    if (isNaN(amount) || amount < 10000)
      return bot.sendMessage(msg.chat.id, "Minimal 10.000")

    const [[u]] = await pool.query("SELECT saldo FROM users WHERE telegram_id=?", [msg.chat.id])
    if (amount > u.saldo)
      return bot.sendMessage(msg.chat.id, "Saldo tidak cukup")

    st.amount = amount
    st.step = 2
    return bot.sendMessage(msg.chat.id, "Metode? (DANA/OVO)")
  }

  if (st.step === 2) {
    st.metode = msg.text
    st.step = 3
    return bot.sendMessage(msg.chat.id, "Nomor tujuan:")
  }

  if (st.step === 3) {
    await pool.query(
      "INSERT INTO withdrawals (user_id, amount, metode, tujuan) VALUES (?, ?, ?, ?)",
      [msg.chat.id, st.amount, st.metode, msg.text]
    )

    delete wdState[msg.chat.id]

    bot.sendMessage(msg.chat.id, "✅ Request dikirim")
  }
})

// ================== RIWAYAT ==================
bot.onText(/📊 Riwayat/, async (msg) => {
  const [rows] = await pool.query(
    "SELECT * FROM transactions WHERE user_id=? ORDER BY id DESC LIMIT 10",
    [msg.chat.id]
  )

  let text = "📊 Riwayat:\n\n"
  rows.forEach(t => {
    text += `${t.type} ${t.amount}\n`
  })

  bot.sendMessage(msg.chat.id, text || "Belum ada riwayat")
})

// ================== ADMIN ==================
bot.onText(/\/admin/, async (msg) => {
  if (!isAdmin(msg.chat.id)) return

  bot.sendMessage(msg.chat.id, "ADMIN PANEL", {
    reply_markup: {
      inline_keyboard: [
        [{ text: "📥 Submission", callback_data: "sub" }],
        [{ text: "💸 Withdraw", callback_data: "wd" }]
      ]
    }
  })
})

// ================== ADMIN CALLBACK ==================
bot.on('callback_query', async (q) => {
  if (!isAdmin(q.message.chat.id)) return

  // SUBMISSION
  if (q.data === "sub") {
    const [subs] = await pool.query("SELECT * FROM submissions WHERE status='pending' LIMIT 1")
    if (!subs.length) return bot.sendMessage(q.message.chat.id, "Kosong")

    const s = subs[0]

    bot.sendMessage(q.message.chat.id,
      `User: ${s.user_id}\nLink: ${s.link}`,
      {
        reply_markup: {
          inline_keyboard: [[
            { text: "✅", callback_data: `acc_${s.id}` },
            { text: "❌", callback_data: `rej_${s.id}` }
          ]]
        }
      }
    )
  }

  if (q.data.startsWith("acc_") || q.data.startsWith("rej_")) {
    const [act, id] = q.data.split("_")
    const [[s]] = await pool.query("SELECT * FROM submissions WHERE id=?", [id])

    if (act === "acc") {
      await pool.query("UPDATE submissions SET status='accepted' WHERE id=?", [id])

      const [[e]] = await pool.query("SELECT reward FROM events WHERE id=?", [s.event_id])

      await pool.query("UPDATE users SET saldo = saldo + ? WHERE telegram_id=?", [e.reward, s.user_id])

      await pool.query(
        "INSERT INTO transactions (user_id,type,amount) VALUES (?, 'reward', ?)",
        [s.user_id, e.reward]
      )
    }

    if (act === "rej") {
      await pool.query("UPDATE submissions SET status='rejected' WHERE id=?", [id])
    }
  }

  // WITHDRAW
  if (q.data === "wd") {
    const [wds] = await pool.query("SELECT * FROM withdrawals WHERE status='pending' LIMIT 1")
    if (!wds.length) return bot.sendMessage(q.message.chat.id, "Kosong")

    const w = wds[0]

    bot.sendMessage(q.message.chat.id,
      `User: ${w.user_id}\n💰 ${w.amount}\n${w.metode} ${w.tujuan}`,
      {
        reply_markup: {
          inline_keyboard: [[
            { text: "✅", callback_data: `wdacc_${w.id}` },
            { text: "❌", callback_data: `wdrej_${w.id}` }
          ]]
        }
      }
    )
  }

  if (q.data.startsWith("wdacc_") || q.data.startsWith("wdrej_")) {
    const [act, id] = q.data.split("_")
    const [[w]] = await pool.query("SELECT * FROM withdrawals WHERE id=?", [id])

    if (act === "wdacc") {
      await pool.query("UPDATE users SET saldo = saldo - ? WHERE telegram_id=?", [w.amount, w.user_id])
      await pool.query("UPDATE withdrawals SET status='approved' WHERE id=?", [id])

      await pool.query(
        "INSERT INTO transactions (user_id,type,amount) VALUES (?, 'withdraw', ?)",
        [w.user_id, w.amount]
      )
    }

    if (act === "wdrej") {
      await pool.query("UPDATE withdrawals SET status='rejected' WHERE id=?", [id])
    }
  }
})
