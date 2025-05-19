# Crash
WhatsApp boot
const { makeWASocket, useSingleFileAuthState } = require('@whiskeysockets/baileys');
const fs = require('fs');
const path = require('path');

const { state, saveState } = useSingleFileAuthState('./auth_info.json');

let users = {};
const usersFile = path.join(__dirname, 'users.json');

if (fs.existsSync(usersFile)) {
  users = JSON.parse(fs.readFileSync(usersFile, 'utf-8'));
}

function saveUsers() {
  fs.writeFileSync(usersFile, JSON.stringify(users, null, 2));
}

function getUser(id) {
  if (!users[id]) {
    users[id] = {
      xp: 0,
      level: 1,
      wallet: 100,
      bank: 0,
      items: {},
      cooldowns: {}
    };
  }
  return users[id];
}

// إضافة نقاط XP للمستخدم
function addXP(id, amount) {
  const user = getUser(id);
  user.xp += amount;
  const neededXP = user.level * 100;
  if (user.xp >= neededXP) {
    user.level++;
    user.xp -= neededXP;
    return true;
  }
  return false;
}

// مثال أمر أوامر لعرض جميع الأوامر
async function sendCommandsList(sock, jid) {
  const commandsList = `
📜 أوامر البوت:
- مستواي : معرفة مستواك ونقاط الخبرة.
- صورة أنمي : صورة أنمي عشوائية.
- متجر : عرض المتجر والشراء.
- صندوق : فتح صندوق عشوائي.
- يومي : مكافأة يومية.
- سرقة النقاط : لعبة سرقة نقاط من مستخدم آخر.
- المتصدرين : عرض أعلى المستخدمين نقاطًا.
- حقيبتي : عرض الأغراض التي تملكها.
- بيع [اسم الغرض] : بيع غرض من حقيبتك.
- مشاركة [@المستخدم] [النقاط] : مشاركة نقاط مع مستخدم.
- بنك : إيداع وسحب العملات.
- صناعة [اسم الغرض] : صناعة غرض جديد.
- غزو دانجن : لعبة غزو الدانجن لكسب نقاط.
- لعبة حظ : لعبة حظ لزيادة العملات.
  `.trim();
  await sock.sendMessage(jid, { text: commandsList });
}

// التعامل مع الأوامر
async function handleCommands(sock, msg) {
  const sender = msg.key.remoteJid;
  const isGroup = sender.endsWith('@g.us');
  if (!isGroup) return; // يعمل فقط في المجموعات

  const text = (msg.message.conversation || '').trim().toLowerCase();
  const user = getUser(sender);

  // أمر أوامر
  if (text === 'أوامر' || text === 'commands') {
    await sendCommandsList(sock, sender);
    return;
  }

  // إضافة XP مع كل رسالة
  const leveledUp = addXP(sender, 10);
  saveUsers();
  if (leveledUp) {
    await sock.sendMessage(sender, { text: `🎉 مبروك! وصلت للمستوى ${user.level}` });
  }

  // المزيد من الأوامر سيتم إضافتها هنا حسب طلبك

  // الرد الافتراضي
  await sock.sendMessage(sender, { text: 'اكتب "أوامر" لمعرفة كل الأوامر المتاحة.' });
}

// بدء البوت
async function startBot() {
  const sock = makeWASocket({ auth: state, printQRInTerminal: true });

  sock.ev.on('creds.update', saveState);

  sock.ev.on('messages.upsert', async ({ messages }) => {
    try {
      const msg = messages[0];
      if (!msg.message || msg.key.fromMe) return;
      await handleCommands(sock, msg);
    } catch (e) {
      console.error('Error handling message:', e);
    }
  });
}

startBot();
