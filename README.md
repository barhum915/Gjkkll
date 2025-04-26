{
  "name": "telegram-store-bot",
  "version": "1.0.0",
  "description": "متجر تلجرام متكامل مع إدارة رصيد وربط مع Google Sheets وSatoFill",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "axios": "^1.6.5",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "node-telegram-bot-api": "^0.61.0",
    "google-spreadsheet": "^3.3.0",
    "nodemailer": "^6.9.8"
  }
}{
  "BOT_TOKEN": "7694710744:AAGa4TiKeD_GNDj-Ufui3u6BjC4ddf6qGhM",
  "6676082269": 6676082269,
  "EMAIL_SENDER": "ibrahimbkar9@gmail.com@gmail.com",
  "EMAIL_PASSWORD": "11(22)332209/9/15",
  "EMAIL_RECEIVER": "ibrahimbkari9@gmail.com",
  "SATO_API_BASE_URL": "https://satofill.com/wp-json/mystore/v1",
  "SATO_API_TOKEN": "eoSXcwmecTkWdYXFvVjivQPwI9oiuPyN",
  "USD_TO_SYP_RATE": 12000,
  "EXTRA_SYP": 1000,
  "MIN_DEPOSIT_USD": 5,
  "SYRIATEL_NUMBERS": ["70028134", "80155384"],
  "SHAM_CASH_ID": "14afe15078916b0f7a8d08b7daf43a00"
}# Gjkkll
const TelegramBot = require('node-telegram-bot-api');
const axios = require('axios');
const express = require('express');
const { GoogleSpreadsheet } = require('google-spreadsheet');
const nodemailer = require('nodemailer');
const fs = require('fs');
const config = require('./config.json');

const bot = new TelegramBot(config.BOT_TOKEN, { polling: true });

const app = express();
app.listen(process.env.PORT || 3000, () => {
  console.log('Bot is running...');
});

// حساب الدولار
function calculateSYP(usd) {
  return (usd * config.USD_TO_SYP_RATE) + config.EXTRA_SYP;
}

// إرسال ايميل بعد الطلب
async function sendEmail(subject, text) {
  let transporter = nodemailer.createTransport({
    service: 'gmail',
    auth: {
      user: config.EMAIL_SENDER,
      pass: config.EMAIL_PASSWORD
    }
  });

  await transporter.sendMail({
    from: config.EMAIL_SENDER,
    to: config.EMAIL_RECEIVER,
    subject: subject,
    text: text
  });
}// حفظ بيانات المستخدمين (بذاكرة بسيطة مؤقتة)
let users = {};

// قائمة الأوامر الأساسية
bot.onText(/\/start/, (msg) => {
  const chatId = msg.chat.id;
  users[chatId] = users[chatId] || { balance: 0, purchases: [] };
  let welcome = `أهلا وسهلا بك في متجرنا!

- لعرض الرصيد: /balance
- لشحن رصيدك: /deposit
- لشراء منتج: /products
- لحسابك الكامل: /myaccount
`;

  if (chatId == config.ADMIN_ID) {
    welcome += "\n- لوحة الإدارة: /admin";
  }

  bot.sendMessage(chatId, welcome);
});

// عرض الرصيد
bot.onText(/\/balance/, (msg) => {
  const chatId = msg.chat.id;
  users[chatId] = users[chatId] || { balance: 0, purchases: [] };
  bot.sendMessage(chatId, `رصيدك الحالي: ${users[chatId].balance.toFixed(2)}$`);
});

// حساب المستخدم
bot.onText(/\/myaccount/, (msg) => {
  const chatId = msg.chat.id;
  const user = users[chatId] || { balance: 0, purchases: [] };

  let text = `معلومات حسابك:\n\n`;
  text += `رصيدك: ${user.balance.toFixed(2)}$\n`;
  text += `عدد الطلبات: ${user.purchases.length}\n`;

  bot.sendMessage(chatId, text);
});// عرض المنتجات
bot.onText(/\/products/, async (msg) => {
  const chatId = msg.chat.id;

  try {
    const response = await axios.get(`${config.SATO_API_BASE_URL}/products/`, {
      headers: { Authorization: `Bearer ${config.SATO_API_TOKEN}` }
    });

    const products = response.data;

    let message = "قائمة المنتجات المتاحة:\n\n";
    products.forEach((product, index) => {
      const profit = 0.5; // ربح ثابت مثلا 0.5$
      const finalPrice = (parseFloat(product.price) + profit).toFixed(2);
      message += `${index + 1}. ${product.name}\nالسعر: ${finalPrice}$\nمعرف المنتج: ${product.id}\n\n`;
    });

    bot.sendMessage(chatId, message);
  } catch (error) {
    bot.sendMessage(chatId, "خطأ في جلب المنتجات، حاول لاحقاً.");
    console.error(error);
  }
});

// شراء منتج
bot.onText(/\/buy (.+)/, async (msg, match) => {
  const chatId = msg.chat.id;
  const input = match[1].split(" ");
  const productId = input[0];
  const quantity = parseInt(input[1]) || 1;
  const urlsocial = input[2];

  users[chatId] = users[chatId] || { balance: 0, purchases: [] };

  try {
    const response = await axios.get(`${config.SATO_API_BASE_URL}/products/`, {
      headers: { Authorization: `Bearer ${config.SATO_API_TOKEN}` }
    });

    const products = response.data;
    const product = products.find(p => p.id == productId);

    if (!product) {
      return bot.sendMessage(chatId, "المنتج غير موجود.");
    }

    const profit = 0.5;
    const pricePerUnit = parseFloat(product.price) + profit;
    const totalCost = pricePerUnit * quantity;

    if (users[chatId].balance < totalCost) {
      return bot.sendMessage(chatId, "رصيدك غير كافي لاتمام هذه العملية.");
    }

    // تنفيذ عملية الشراء
    await axios.post(`${config.SATO_API_BASE_URL}/create-order`, {
      product_id: productId,
      quantity: quantity,
      urlsocial: urlsocial
    }, {
      headers: { Authorization: `Bearer ${config.SATO_API_TOKEN}` }
    });

    users[chatId].balance -= totalCost;
    users[chatId].purchases.push({ productId, quantity });

    bot.sendMessage(chatId, `تم شراء المنتج بنجاح! خصمنا من رصيدك ${totalCost.toFixed(2)}$.`);
    sendEmail("عملية شراء جديدة", `المستخدم ${chatId} اشترى المنتج ${productId} بكمية ${quantity}.`);

  } catch (error) {
    bot.sendMessage(chatId, "فشل تنفيذ الطلب، حاول لاحقاً.");
    console.error(error);
  }
});// شحن الرصيد
bot.onText(/\/deposit/, (msg) => {
  const chatId = msg.chat.id;

  const message = `
طرق شحن الرصيد:

- سيريتل كاش: 
رقم: 70028134 أو 80155384

- شام كاش:
رمز المحفظة: 14afe15078916b0f7a8d08b7daf43a00

ملاحظة:
- أقل إيداع: 5$ (ما يعادل بالليرة السورية).
- تحويل أقل من ذلك لا يُقبل.
- بعد التحويل، ارسل صورة التحويل مع كتابة المبلغ.

(سعر الدولار محسوب مع زيادة 1000 ليرة سورية)

`;

  bot.sendMessage(chatId, message);
});

// استقبال صور التحويل
bot.on('message', async (msg) => {
  const chatId = msg.chat.id;

  if (msg.photo) {
    users[chatId] = users[chatId] || { balance: 0, purchases: [] };
    bot.sendMessage(chatId, `تم استلام صورة التحويل. سيتم التأكد وإضافة الرصيد خلال وقت قصير.`);

    // ترسل الصورة للادمن مع معلومات الشخص
    bot.sendMessage(config.ADMIN_ID, `مستخدم جديد أرسل صورة تحويل.\nمعلوماته:\nID: ${chatId}\nاسم المستخدم: @${msg.from.username || 'لا يوجد'}\nيرجى التأكد والإضافة.`);

    // حفظ الصورة او التعامل معها
  }
});// لوحة الإدارة
bot.onText(/\/admin/, (msg) => {
  const chatId = msg.chat.id;

  if (chatId != config.ADMIN_ID) {
    return bot.sendMessage(chatId, "هذه الميزة خاصة بالمسؤول فقط.");
  }

  const message = `
مرحباً بك في لوحة الإدارة:

- عرض المستخدمين: /users
- تعديل رصيد مستخدم: /setbalance [id] [new_balance]
`;

  bot.sendMessage(chatId, message);
});

// عرض جميع المستخدمين
bot.onText(/\/users/, (msg) => {
  const chatId = msg.chat.id;

  if (chatId != config.ADMIN_ID) return;

  let message = "قائمة المستخدمين:\n\n";

  Object.keys(users).forEach((id) => {
    message += `ID: ${id} - الرصيد: ${users[id].balance.toFixed(2)}$\n`;
  });

  bot.sendMessage(chatId, message);
});

// تعديل رصيد مستخدم
bot.onText(/\/setbalance (.+)/, (msg, match) => {
  const chatId = msg.chat.id;

  if (chatId != config.ADMIN_ID) return;

  const input = match[1].split(" ");
  const userId = input[0];
  const newBalance = parseFloat(input[1]);

  if (!users[userId]) {
    return bot.sendMessage(chatId, "المستخدم غير موجود.");
  }

  users[userId].balance = newBalance;
  bot.sendMessage(chatId, `تم تحديث رصيد المستخدم ${userId} إلى ${newBalance}$`);
});// استيراد مكتبة fs
const fs = require('fs');

// تحميل المستخدمين من ملف users.json عند التشغيل
if (fs.existsSync('users.json')) {
  const data = fs.readFileSync('users.json');
  users = JSON.parse(data);
}

// حفظ المستخدمين تلقائياً كل دقيقة
setInterval(() => {
  fs.writeFileSync('users.json', JSON.stringify(users, null, 2));
}, 60 * 1000);
