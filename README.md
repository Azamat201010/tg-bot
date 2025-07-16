# tg-bot
require('dotenv').config();
const TelegramBot = require('node-telegram-bot-api');
const fs = require('fs');

const token = process.env.BOT_TOKEN;
const bot = new TelegramBot(token, { polling: true });

const CONFIG_PATH = './config.txt';

function parseConfig() {
  if (!fs.existsSync(CONFIG_PATH)) {
    fs.writeFileSync(CONFIG_PATH, `adminLogin=Azamat\nadminPassword=2010\nadmins=\nvalidCodes=\nlastMovieFileId=\nlastMovieType=\nlastCaption=`);
  }
  const data = fs.readFileSync(CONFIG_PATH, 'utf8');
  const config = {};
  data.split('\n').forEach(line => {
    const [key, value] = line.split('=');
    config[key] = value || '';
  });
  config.admins = config.admins ? config.admins.split(',').map(id => parseInt(id)) : [];
  config.validCodes = config.validCodes ? config.validCodes.split(',') : [];
  return config;
}

function saveConfig(config) {
  const lines = [
    `adminLogin=${config.adminLogin}`,
    `adminPassword=${config.adminPassword}`,
    `admins=${config.admins.join(',')}`,
    `validCodes=${config.validCodes.join(',')}`,
    `lastMovieFileId=${config.lastMovieFileId || ''}`,
    `lastMovieType=${config.lastMovieType || ''}`,
    `lastCaption=${config.lastCaption || ''}`,
  ];
  fs.writeFileSync(CONFIG_PATH, lines.join('\n'));
}

const loginStep = {};

bot.onText(/\/start/, (msg) => {
  const chatId = msg.chat.id;
  const username = msg.from.username ? '@' + msg.from.username : (msg.from.first_name || "Nomaʼlum");

  if (!fs.existsSync('./id.txt') || !fs.readFileSync('./id.txt', 'utf8').includes(chatId.toString())) {
    fs.appendFileSync('./id.txt', `${chatId} - ${username}\n`);
  }

  bot.sendMessage(chatId, "👋 Salom! Kod yuboring — agar to‘g‘ri bo‘lsa, sizga media beriladi.");
  bot.sendMessage(chatId, "Kodlar ro'yxatini ko'rish uchun /list buyrug'ini yuboring.");
});

bot.onText(/\/admin/, (msg) => {
  loginStep[msg.chat.id] = { step: 'login' };
  bot.sendMessage(msg.chat.id, "🧑‍💻 Iltimos, loginingizni kiriting:");
});

bot.onText(/\/list/, (msg) => {
  bot.sendMessage(msg.chat.id, "Kalmar oyini 3 sezon (kod: 1–6 gacha)");
});

bot.onText(/\/history/, (msg) => {
  const config = parseConfig();
  if (config.admins.includes(msg.chat.id)) {
    bot.sendMessage(msg.chat.id, "👤 Foydalanuvchi ID raqamini yuboring: /hst [ID]");
  } else {
    bot.sendMessage(msg.chat.id, "🚫 Siz admin emassiz.");
  }
});

bot.onText(/\/hst (\d+)/, (msg, match) => {
  const chatId = msg.chat.id;
  const config = parseConfig();

  if (!config.admins.includes(chatId)) {
    bot.sendMessage(chatId, "🚫 Siz admin emassiz.");
    return;
  }

  const targetId = match[1];
  const filePath = `./history_${targetId}.txt`;

  if (fs.existsSync(filePath)) {
    const content = fs.readFileSync(filePath, 'utf8').trim();
    if (content) {
      bot.sendMessage(chatId, `📂 ${targetId} ning tarixi:\n\n` + content);
    } else {
      bot.sendMessage(chatId, "🗃 Tarix topilmadi.");
    }
  } else {
    bot.sendMessage(chatId, "🗃 Tarix topilmadi.");
  }
});

bot.on('message', (msg) => {
  const chatId = msg.chat.id;
  const text = msg.text;
  let config = parseConfig();

  if (loginStep[chatId]) {
    const step = loginStep[chatId].step;

    if (step === 'login') {
      loginStep[chatId].login = text;
      loginStep[chatId].step = 'password';
      bot.sendMessage(chatId, "🔐 Endi parolingizni kiriting:");
      return;
    }

    if (step === 'password') {
      const login = loginStep[chatId].login;
      const password = text;

      if (login === config.adminLogin && password === config.adminPassword) {
        if (!config.admins.includes(chatId)) {
          config.admins.push(chatId);
          saveConfig(config);
        }
        delete loginStep[chatId];
        bot.sendMessage(chatId, "✅ Siz admin sifatida tizimga kirdingiz. Endi media yuborishingiz mumkin.\nTarixni ko‘rish uchun /history ni bosing.");
      } else {
        delete loginStep[chatId];
        bot.sendMessage(chatId, "❌ Login yoki parol noto‘g‘ri.");
      }
      return;
    }
  }

  if (config.admins.includes(chatId)) {
    let fileId = null;
    let type = null;

    if (msg.video) {
      fileId = msg.video.file_id;
      type = 'video';
    } else if (msg.photo) {
      const largest = msg.photo[msg.photo.length - 1];
      fileId = largest.file_id;
      type = 'photo';
    }

    if (fileId && type) {
      config.lastMovieFileId = fileId;
      config.lastMovieType = type;

      if (msg.caption) {
        const parts = msg.caption.split('|');
        const code = parts[0].trim();
        const description = parts[1] ? parts[1].trim() : "";
        if (!config.validCodes.includes(code)) {
          config.validCodes.push(code);
        }
        config.lastCaption = description;
      } else {
        config.lastCaption = "";
      }

      saveConfig(config);
      bot.sendMessage(chatId, "✅ Media saqlandi.");
      return;
    }
  }

  if (!config.admins.includes(chatId)) {
    let fileId = null;
    let type = null;

    if (msg.video) {
      fileId = msg.video.file_id;
      type = 'video';
    } else if (msg.photo) {
      const largest = msg.photo[msg.photo.length - 1];
      fileId = largest.file_id;
      type = 'photo';
    }

    if (fileId && type) {
      const entry = `${type.toUpperCase()}: ${fileId}\n`;
      if (!fs.existsSync('./media.txt')) fs.writeFileSync('./media.txt', '');
      if (!fs.readFileSync('./media.txt', 'utf8').includes(fileId)) {
        fs.appendFileSync('./media.txt', entry);
      }
      bot.sendMessage(chatId, "✅ Mediyangiz qabul qilindi.");
      return;
    }
  }

  if (text && config.validCodes.includes(text)) {
    if (config.lastMovieFileId && config.lastMovieType) {
      const caption = `🍿 Mana sizning media.\n${config.lastCaption || ""}`;
      if (config.lastMovieType === 'video') {
        bot.sendVideo(chatId, config.lastMovieFileId, { caption });
      } else if (config.lastMovieType === 'photo') {
        bot.sendPhoto(chatId, config.lastMovieFileId, { caption });
      }
    } else {
      bot.sendMessage(chatId, "❗ Media hali qo‘shilmagan.");
    }
    return;
  }

  if (text && !text.startsWith('/')) {
    const line = `${new Date().toLocaleString()} | ${chatId}: ${text}\n`;
    const filePath = `./history_${chatId}.txt`;
    fs.appendFileSync(filePath, line);
  }
});

