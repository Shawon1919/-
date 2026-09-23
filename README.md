// server.js
// WhatsApp Reminder Bot - বাংলা + Banglish
// Node.js 18+

import "dotenv/config";
import express from "express";
import fs from "fs";
import crypto from "crypto";

const app = express();
app.use(express.json());

const PORT = process.env.PORT || 3000;
const VERIFY_TOKEN = process.env.VERIFY_TOKEN;
const WHATSAPP_TOKEN = process.env.WHATSAPP_TOKEN;
const PHONE_NUMBER_ID = process.env.PHONE_NUMBER_ID;
const GRAPH_VERSION = process.env.GRAPH_VERSION || "v23.0";
const TIMEZONE = "Asia/Dhaka";

const DB_FILE = "./reminders.json";

// ---------------- DATABASE ----------------

function loadReminders() {
  if (!fs.existsSync(DB_FILE)) {
    fs.writeFileSync(DB_FILE, "[]");
  }

  return JSON.parse(fs.readFileSync(DB_FILE, "utf8"));
}

function saveReminders(data) {
  fs.writeFileSync(
    DB_FILE,
    JSON.stringify(data, null, 2),
    "utf8"
  );
}

// ---------------- WHATSAPP SEND ----------------

async function sendWhatsApp(to, message) {
  const url =
    `https://graph.facebook.com/${GRAPH_VERSION}/` +
    `${PHONE_NUMBER_ID}/messages`;

  const response = await fetch(url, {
    method: "POST",

    headers: {
      "Authorization": `Bearer ${WHATSAPP_TOKEN}`,
      "Content-Type": "application/json"
    },

    body: JSON.stringify({
      messaging_product: "whatsapp",

      to,

      type: "text",

      text: {
        body: message
      }
    })
  });

  const data = await response.json();

  if (!response.ok) {
    console.error(data);
    throw new Error("WhatsApp message failed");
  }

  return data;
}

// ---------------- বাংলা DIGIT ----------------

function convertBanglaNumber(text) {
  const bangla = "০১২৩৪৫৬৭৮৯";

  return text.replace(/[০-৯]/g, d => {
    return bangla.indexOf(d);
  });
}

// ---------------- TIME PARSER ----------------

function getTime(text) {

  text = convertBanglaNumber(text)
    .toLowerCase();

  let hour;
  let minute = 0;

  const match = text.match(
    /(\d{1,2})(?:[:.](\d{1,2}))?\s*(am|pm|টায়|টায়|টা)?/
  );

  if (!match) return null;

  hour = Number(match[1]);

  if (match[2]) {
    minute = Number(match[2]);
  }

  const full = match[0];

  if (
    text.includes("বিকাল") ||
    text.includes("বিকেলে") ||
    text.includes("সন্ধ্যা") ||
    text.includes("রাত") ||
    text.includes("রাতে")
  ) {
    if (hour < 12) hour += 12;
  }

  if (
    text.includes("সকাল") ||
    text.includes("সকালে")
  ) {
    if (hour === 12) hour = 0;
  }

  if (full.includes("pm")) {
    if (hour < 12) hour += 12;
  }

  if (full.includes("am")) {
    if (hour === 12) hour = 0;
  }

  if (hour > 23 || minute > 59) {
    return null;
  }

  return {
    hour,
    minute
  };
}

// ---------------- DATE PARSER ----------------

function getDate(text) {

  const now = new Date();

  const date = new Date(now);

  text = text.toLowerCase();

  if (
    text.includes("আগামীকাল") ||
    text.includes("কাল") ||
    text.includes("tomorrow") ||
    text.includes("tmr")
  ) {
    date.setDate(date.getDate() + 1);
  }

  return date;
}

// ---------------- TASK CLEANER ----------------

function getTask(text) {

  let task = text;

  const removeWords = [

    "remind",
    "reminder",

    "মনে করিয়ে দিও",
    "মনে করিয়ে দিও",
    "মনে করিয়ে দাও",
    "মনে করিয়ে দাও",

    "মনে করিও",

    "আগামীকাল",
    "কাল",

    "tomorrow",
    "tmr",

    "সকাল",
    "সকালে",

    "দুপুর",

    "বিকাল",
    "বিকেলে",

    "সন্ধ্যা",

    "রাত",
    "রাতে",

    "প্রতিদিন",
    "daily",

    "প্রতি সপ্তাহে",
    "weekly",

    "every day",
    "every week"
  ];

  for (const word of removeWords) {
    task = task.replace(
      new RegExp(word, "gi"),
      " "
    );
  }

  task = task.replace(
    /\b\d{1,2}(?:[:.]\d{1,2})?\s*(?:am|pm)?\b/gi,
    " "
  );

  task = task.replace(
    /\s+/g,
    " "
  );

  return task.trim();
}

// ---------------- REMINDER PARSER ----------------

function parseReminder(text) {

  const time = getTime(text);

  if (!time) {
    return null;
  }

  const date = getDate(text);

  date.setHours(
    time.hour,
    time.minute,
    0,
    0
  );

  const lower = text.toLowerCase();

  let repeat = "once";

  if (
    lower.includes("প্রতিদিন") ||
    lower.includes("daily") ||
    lower.includes("every day")
  ) {
    repeat = "daily";
  }

  if (
    lower.includes("প্রতি সপ্তাহে") ||
    lower.includes("weekly") ||
    lower.includes("every week")
  ) {
    repeat = "weekly";
  }

  if (
    repeat === "once" &&
    date.getTime() <= Date.now()
  ) {
    date.setDate(
      date.getDate() + 1
    );
  }

  let task = getTask(text);

  if (!task) {
    task = "Reminder";
  }

  return {

    task,

    time: date.toISOString(),

    repeat
  };
}

// ---------------- FORMAT DATE ----------------

function formatDate(date) {

  return new Intl.DateTimeFormat(
    "bn-BD",
    {
      timeZone: TIMEZONE,

      year: "numeric",
      month: "long",
      day: "numeric",

      hour: "numeric",
      minute: "2-digit",

      hour12: true
    }
  ).format(new Date(date));
}

// ---------------- COMMAND: LIST ----------------

async function handleList(phone) {

  const reminders =
    loadReminders()
      .filter(r => r.phone === phone);

  if (!reminders.length) {

    await sendWhatsApp(
      phone,
      "📭 তোমার কোনো Reminder নেই।"
    );

    return;
  }

  let message =
    "📋 তোমার Reminder তালিকা:\n\n";

  reminders.forEach((r, index) => {

    message +=
      `${index + 1}. 📝 ${r.task}\n` +
      `⏰ ${formatDate(r.time)}\n` +
      `🔁 ${r.repeat}\n\n`;
  });

  await sendWhatsApp(
    phone,
    message
  );
}

// ---------------- COMMAND: DELETE ----------------

async function handleDelete(
  phone,
  number
) {

  const data = loadReminders();

  const userReminders =
    data.filter(
      r => r.phone === phone
    );

  const index =
    Number(number) - 1;

  if (
    index < 0 ||
    index >= userReminders.length
  ) {

    await sendWhatsApp(
      phone,
      "❌ এই নম্বরের Reminder পাওয়া যায়নি।"
    );

    return;
  }

  const target =
    userReminders[index];

  const newData =
    data.filter(
      r => r.id !== target.id
    );

  saveReminders(newData);

  await sendWhatsApp(
    phone,
    "🗑️ Reminder মুছে ফেলা হয়েছে।"
  );
}

// ---------------- HELP ----------------

async function handleHelp(phone) {

  const message = `
🤖 WhatsApp Reminder Bot

বাংলা:
কাল সকাল ৮টায় মুরগিকে খাবার দিতে মনে করিয়ে দিও

Banglish:
kal sokal 8tay murgi k khabar dite mone koriye dio

আরও:

daily 7:00 পানি খেতে

weekly 20:00 হিসাব করতে

list
delete 1
help

📌 Reminder শেষ হলেও Bot মেসেজ delete করবে না।
`;

  await sendWhatsApp(
    phone,
    message
  );
}

// ---------------- WEBHOOK VERIFY ----------------

app.get(
  "/webhook",
  (req, res) => {

    const mode =
      req.query["hub.mode"];

    const token =
      req.query["hub.verify_token"];

    const challenge =
      req.query["hub.challenge"];

    if (
      mode === "subscribe" &&
      token === VERIFY_TOKEN
    ) {

      return res
        .status(200)
        .send(challenge);
    }

    return res
      .sendStatus(403);
  }
);

// ---------------- WEBHOOK MESSAGE ----------------

app.post(
  "/webhook",
  async (req, res) => {

    res.sendStatus(200);

    try {

      const message =
        req.body
          ?.entry?.[0]
          ?.changes?.[0]
          ?.value
          ?.messages?.[0];

      if (!message) return;

      if (message.type !== "text") {
        return;
      }

      const phone =
        message.from;

      const text =
        message.text.body.trim();

      // HELP

      if (
        /^help$/i.test(text) ||
        text === "সাহায্য"
      ) {

        await handleHelp(phone);

        return;
      }

      // LIST

      if (
        /^list$/i.test(text) ||
        text === "লিস্ট" ||
        text === "তালিকা"
      ) {

        await handleList(phone);

        return;
      }

      // DELETE

      const deleteMatch =
        text.match(
          /^(delete|ডিলিট|মুছে ফেলো)\s*(\d+)$/i
        );

      if (deleteMatch) {

        await handleDelete(
          phone,
          deleteMatch[2]
        );

        return;
      }

      // CREATE REMINDER

      const reminder =
        parseReminder(text);

      if (!reminder) {

        await sendWhatsApp(
          phone,

          `❓ আমি সময় বুঝতে পারিনি।

উদাহরণ:

কাল সকাল ৮টায় মুরগিকে খাবার দিতে মনে করিয়ে দিও

অথবা:

kal sokal 8tay doctor`
        );

        return;
      }

      const data =
        loadReminders();

      const newReminder = {

        id: crypto.randomUUID(),

        phone,

        task: reminder.task,

        time: reminder.time,

        repeat: reminder.repeat,

        sent: false,

        createdAt:
          new Date().toISOString()
      };

      data.push(
        newReminder
      );

      saveReminders(data);

      await sendWhatsApp(
        phone,

        `✅ Reminder সেট হয়েছে!

📝 ${reminder.task}

⏰ ${formatDate(reminder.time)}

🔁 ${
  reminder.repeat === "once"
    ? "একবার"
    : reminder.repeat === "daily"
    ? "প্রতিদিন"
    : "প্রতি সপ্তাহে"
}

📌 Reminder মেসেজ অটো-ডিলিট হবে না।`
      );

    } catch (error) {

      console.error(
        "Webhook Error:",
        error
      );
    }
  }
);

// ---------------- REMINDER CHECKER ----------------

async function checkReminders() {

  const data =
    loadReminders();

  let changed = false;

  for (const reminder of data) {

    if (reminder.sent) {
      continue;
    }

    const due =
      new Date(
        reminder.time
      ).getTime();

    if (
      due > Date.now()
    ) {
      continue;
    }

    try {

      await sendWhatsApp(

        reminder.phone,

        `🔔 REMINDER

📝 ${reminder.task}

⏰ ${formatDate(
  reminder.time
)}

✅ Reminder-এর সময় হয়েছে।

📌 এই মেসেজটি Bot নিজে থেকে delete করবে না।`
      );

      if (
        reminder.repeat === "daily"
      ) {

        const next =
          new Date(
            reminder.time
          );

        next.setDate(
          next.getDate() + 1
        );

        reminder.time =
          next.toISOString();

      }

      else if (
        reminder.repeat === "weekly"
      ) {

        const next =
          new Date(
            reminder.time
          );

        next.setDate(
          next.getDate() + 7
        );

        reminder.time =
          next.toISOString();

      }

      else {

        reminder.sent = true;
      }

      changed = true;

    } catch (error) {

      console.error(
        "Reminder Error:",
        error
      );
    }
  }

  if (changed) {

    saveReminders(data);
  }
}

// প্রতি ১ মিনিটে Reminder চেক

setInterval(
  checkReminders,
  60 * 1000
);

// ---------------- START SERVER ----------------

app.listen(
  PORT,
  () => {

    console.log(
      `🤖 WhatsApp Reminder Bot চলছে: PORT ${PORT}`
    );

  }
);
