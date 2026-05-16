# 📱 Baileys - WhatsApp WebSocket API untuk Node.js

**Baileys** adalah library Node.js yang powerful untuk mengontrol WhatsApp melalui koneksi WebSocket. Library ini memungkinkan Anda membuat bot WhatsApp, mengotomatisasi pesan, dan mengintegrasikan WhatsApp ke aplikasi Anda tanpa perlu aplikasi official WhatsApp.

![fazz](https://raw.githubusercontent.com/fazzcodex/database-forme/refs/heads/main/file_0000000057c8720bbacdf1f1a7d5c80b.png)
---

## 📋 Daftar Isi

- [Fitur Utama](#fitur-utama)
- [Requirement](#requirement)
- [Instalasi](#instalasi)
- [Quick Start](#quick-start)
- [Autentikasi](#autentikasi)
- [Konfigurasi Koneksi](#konfigurasi-koneksi)
- [Event & Listener](#event--listener)
- [Mengirim Pesan](#mengirim-pesan)
- [Manajemen Grup](#manajemen-grup)
- [Media & File](#media--file)
- [Error Handling](#error-handling)
- [Best Practices](#best-practices)
- [Contoh Implementasi Lengkap](#contoh-implementasi-lengkap)

---

## ✨ Fitur Utama

- ✅ **Multi-Device Support** - Support untuk multiple devices/session
- ✅ **Pesan Teks** - Kirim & terima pesan teks
- ✅ **Media Support** - Image, Video, Audio, Documents, Stickers
- ✅ **Group Management** - Create, edit, manage groups
- ✅ **Status & Stories** - Support untuk WhatsApp Status
- ✅ **Contact & Chat** - Kelola kontak dan chat
- ✅ **Message React & Quote** - Reply, quote, react to messages
- ✅ **Typing Indicator** - Show typing status
- ✅ **User Status** - Get user online/offline status
- ✅ **Link Preview** - Generate link preview untuk pesan

---

## 🔧 Requirement

- **Node.js** >= 20.0.0 (direkomendasikan LTS terbaru)
- **npm** atau **yarn**
- **WhatsApp Account** (hanya 1 account per instance)
- **System Requirements**:
  - Minimum 512MB RAM
  - Koneksi internet yang stabil
  - Port WebSocket terbuka (biasanya port 443)

---

## 📦 Instalasi

### Via NPM

```bash
npm install @fazzcode/baileys
```

### Via Yarn

```bash
yarn add @fazzcode/baileys
```

### Dependency Tambahan (Optional)

```bash
# Untuk link preview
npm install link-preview-js

# Untuk image processing
npm install jimp@latest

# Untuk audio processing
npm install audio-decode

# Logging
npm install pino
```

---

## ⚡ Quick Start

Berikut adalah contoh paling sederhana untuk memulai bot WhatsApp:

```javascript
import makeWASocket, { 
  DisconnectReason, 
  useMultiFileAuthState 
} from '@fazzcode/baileys';
import { Boom } from '@hapi/boom';

const startBot = async () => {
  // Setup authentication state
  const { state, saveCreds } = await useMultiFileAuthState('auth_info');

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true
  });

  // Save credentials after every connection update
  sock.ev.on('creds.update', saveCreds);

  // Handle connection events
  sock.ev.on('connection.update', async (update) => {
    const { connection, lastDisconnect, qr } = update;

    if (qr) {
      console.log('📱 Scan QR code di atas dengan WhatsApp Anda');
    }

    if (connection === 'open') {
      console.log('✅ Bot terhubung dengan WhatsApp');
    }

    if (connection === 'close') {
      const shouldReconnect = 
        (lastDisconnect?.error)?.output?.statusCode !== DisconnectReason.loggedOut;
      if (shouldReconnect) {
        startBot();
      }
    }
  });

  // Handle incoming messages
  sock.ev.on('messages.upsert', async (m) => {
    for (const msg of m.messages) {
      if (!msg.message) continue; // Skip empty messages

      const text = msg.message.conversation || 
                   msg.message.extendedTextMessage?.text || '';

      console.log(`📬 Pesan dari ${msg.key.remoteJid}: ${text}`);

      // Auto reply
      await sock.sendMessage(msg.key.remoteJid, {
        text: 'Terima kasih pesan Anda! Bot sedang belajar.'
      });
    }
  });
};

startBot().catch(console.error);
```

---

## 🔐 Autentikasi

### 1. Multi-File Auth State (Recommended)

Menyimpan kredensial dalam multiple file untuk keamanan lebih baik:

```javascript
import { useMultiFileAuthState } from '@fazzcode/baileys';

const { state, saveCreds } = await useMultiFileAuthState('auth_info');

// Structure folder:
// auth_info/
// ├── creds.json        (main credentials)
// ├── sessions.json
// ├── pre-keys.json
// └── app-state-sync-key-*.json
```

**File yang Generated:**

```
auth_info/
├── creds.json              # Main credentials & connection info
├── sessions.json           # Device sessions
├── app-state-sync-key-{id}.json  # Session keys untuk sync
└── pre-keys.json          # Pre-encryption keys
```

### 2. Pairing Code

Untuk WhatsApp Business, gunakan pairing code instead of QR:

```javascript
import makeWASocket from '@fazzcode/baileys';

const sock = makeWASocket({
  auth: state,
  method: 'phone-number'  // Request pairing code
});

// **IMPORTANT**: Pairing code harus di-request IMMEDIATELY setelah makeWASocket
// dengan delay minimal 1.5 detik untuk WhatsApp Business compatibility
sock.ev.on('connection.update', async (update) => {
  if (update.receivedPairingCode) {
    console.log('🔐 Pairing Code:', update.pairingCode);
  }
});
```

### 3. Save & Load Credentials

```javascript
// Load existing credentials
const loadCreds = async () => {
  try {
    const creds = JSON.parse(
      fs.readFileSync('auth_info/creds.json', 'utf-8')
    );
    return creds;
  } catch (error) {
    console.log('Credentials not found, starting fresh');
    return null;
  }
};

// Save credentials after update
sock.ev.on('creds.update', () => {
  saveCreds();
  console.log('✅ Credentials saved');
});
```

---

## ⚙️ Konfigurasi Koneksi

### Default Configuration

```javascript
const sock = makeWASocket({
  // ===== AUTHENTICATION =====
  auth: state,
  printQRInTerminal: true,

  // ===== BROWSER SETTINGS =====
  browser: ['Ubuntu', 'Chrome', '20.0.04'],  // ⭐ Important untuk stability
  
  // ===== CONNECTION SETTINGS =====
  connectTimeoutMs: 20000,          // Timeout untuk koneksi (ms)
  keepAliveIntervalMs: 30000,       // Keep-alive interval (ms)
  waWebSocketUrl: 'wss://web.whatsapp.com/ws/chat',
  
  // ===== MESSAGE SETTINGS =====
  markOnlineOnConnect: true,        // Mark as online when connected
  emitOwnEvents: true,              // Emit own messages as events
  maxMsgRetryCount: 5,              // Max retry untuk gagal kirim
  
  // ===== SYNC SETTINGS =====
  syncFullHistory: true,            // Sync full chat history
  shouldSyncHistoryMessage: () => true,
  
  // ===== QUERY & TIMEOUT =====
  defaultQueryTimeoutMs: 60000,     // Default query timeout
  fireInitQueries: true,            // Fire initial queries
  
  // ===== OTHER =====
  countryCode: 'US',                // Country code
  linkPreviewImageThumbnailWidth: 192,  // Link preview width
  enableAutoSessionRecreation: true,
  enableRecentMessageCache: true
});
```

### Custom Configuration Example

```javascript
import { Browsers } from '@fazzcode/baileys';

const customConfig = {
  auth: state,
  printQRInTerminal: false,  // Disable QR di terminal
  
  // Use different browser profile untuk bypass detection
  browser: Browsers.macOS('Safari'),  // Safari on macOS
  // atau
  // browser: Browsers.windows('Chrome'),  // Chrome on Windows
  // browser: Browsers.iOS('Safari'),       // iPhone Safari
  
  // Custom logger
  logger: customLogger.child({ module: 'whatsapp' }),
  
  // Message patches
  patchMessageBeforeSending: (msg) => {
    // Modify message sebelum dikirim
    msg.timestamp = Date.now();
    return msg;
  },
  
  // Ignore certain JIDs
  shouldIgnoreJid: (jid) => {
    return jid.includes('status@broadcast');
  },
  
  // Custom message retrieval
  getMessage: async (key) => {
    // Fetch message dari database
    return await db.getMessage(key);
  },
  
  // Custom group metadata
  cachedGroupMetadata: async (jid) => {
    return await db.getGroupInfo(jid);
  }
};

const sock = makeWASocket(customConfig);
```

---

## 📡 Event & Listener

### Connection Events

```javascript
sock.ev.on('connection.update', (update) => {
  const { 
    connection,           // Connection status
    lastDisconnect,       // Last disconnect info
    isNewLogin,          // Is new login?
    qr,                  // QR code
    receivedPairingCode  // Received pairing code?
  } = update;

  console.log('Connection Update:', {
    connection,
    lastDisconnect,
    isNewLogin
  });

  if (connection === 'open') {
    console.log('✅ Connected');
  } else if (connection === 'close') {
    console.log('❌ Disconnected');
    
    // Check disconnect reason
    const reason = lastDisconnect?.error?.output?.statusCode;
    if (reason === DisconnectReason.loggedOut) {
      console.log('👤 Logout by user');
    } else if (reason === DisconnectReason.connectionClosed) {
      console.log('🔌 Connection closed');
    }
  }
});
```

### Message Events

```javascript
// Incoming & Outgoing Messages
sock.ev.on('messages.upsert', async ({ messages, type }) => {
  for (const msg of messages) {
    const {
      key,           // Message key (remoteJid, fromMe, id, etc)
      message,       // Message content
      messageTimestamp,
      status,        // Message status (pending, sent, delivered, read)
      participant    // For groups: who sent it
    } = msg;

    // Message type
    if (type === 'notify') {
      console.log('📬 New message received');
    } else if (type === 'append') {
      console.log('📝 Message appended');
    }
  }
});

// Message reactions
sock.ev.on('messages.reaction', async (reactions) => {
  for (const reaction of reactions) {
    console.log('👍 Reaction:', reaction);
  }
});
```

### Chat Events

```javascript
sock.ev.on('chats.set', (chats) => {
  console.log(`📚 Loaded ${chats.length} chats`);
});

sock.ev.on('chats.update', (chats) => {
  for (const chat of chats) {
    console.log(`Chat updated: ${chat.name}`);
  }
});

sock.ev.on('chats.delete', (keys) => {
  console.log(`🗑️ ${keys.length} chats deleted`);
});
```

### Group Events

```javascript
sock.ev.on('groups.update', (updates) => {
  for (const update of updates) {
    console.log('👥 Group updated:', update);
  }
});

sock.ev.on('group-participants.update', (update) => {
  const { id, participants, action } = update;
  console.log(`Participant ${action} in ${id}:`, participants);
});
```

### Contact Events

```javascript
sock.ev.on('contacts.set', (contacts) => {
  console.log(`📇 Loaded ${contacts.length} contacts`);
});

sock.ev.on('contacts.upsert', (contacts) => {
  console.log('Contacts updated:', contacts);
});
```

### Status & Presence

```javascript
sock.ev.on('presence.update', (presences) => {
  for (const [jid, presence] of Object.entries(presences)) {
    // last_seen: timestamp kapan last online
    // online: true/false
    console.log(`${jid}: ${presence.lastSeen ? 'Online' : 'Offline'}`);
  }
});
```

### Credentials Update

```javascript
sock.ev.on('creds.update', () => {
  saveCreds();
  console.log('🔐 Credentials updated');
});
```

---

## 💬 Mengirim Pesan

### 1. Pesan Teks

```javascript
// Simple text message
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'Halo, ini pesan dari bot!'
});

// Extended text message (dengan formatting)
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'Pesan dengan *bold* dan _italic_',
  // WhatsApp support formatting terbatas
});
```

### 2. Format JID (Jabber ID)

```javascript
// Personal chat
const jid1 = '6281234567890@s.whatsapp.net';

// Group chat
const jid2 = '120363023179xxxxxx-1234567890@g.us';

// Status/Broadcast
const jid3 = 'status@broadcast';

// Function untuk convert no telepon ke JID
const phoneToJid = (phone) => {
  let cleaned = phone.replace(/\D/g, '');
  if (!cleaned.startsWith('62')) {
    cleaned = '62' + cleaned.substring(1);
  }
  return `${cleaned}@s.whatsapp.net`;
};
```

### 3. Pesan dengan Mention

```javascript
await sock.sendMessage('120363023179xxxxxx-1234567890@g.us', {
  text: 'Halo @6281234567890 dan @6289876543210',
  mentions: ['6281234567890@s.whatsapp.net', '6289876543210@s.whatsapp.net']
});
```

### 4. Pesan dengan Quote/Reply

```javascript
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'Ini adalah reply',
  quoted: msg  // msg dari messages.upsert event
});
```

### 5. Pesan dengan Reaction/Emoji

```javascript
await sock.sendMessage(msg.key.remoteJid, {
  react: {
    text: '😂',  // Emoji reaction
    key: msg.key
  }
});
```

### 6. Pesan Template/Button

```javascript
// Template message dengan buttons
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  templateButtons: [
    {
      index: 1,
      urlButton: {
        displayText: 'Visit Website',
        url: 'https://example.com'
      }
    },
    {
      index: 2,
      callButton: {
        displayText: 'Call Me',
        phoneNumber: '+6281234567890'
      }
    },
    {
      index: 3,
      quickReplyButton: {
        displayText: 'Reply Quick',
        id: 'quick_reply_1'
      }
    }
  ],
  body: 'Pilih salah satu opsi di bawah'
});
```

### 7. Pesan dengan Link Preview

```javascript
// Kirim link dengan preview
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'Check this out: https://github.com/WhiskeySockets/Baileys',
  linkPreview: true  // Generate preview otomatis
});

// Atau custom link preview
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  text: 'https://example.com',
  linkPreview: {
    title: 'Example Website',
    description: 'This is an example',
    jpegThumbnail: Buffer.from('...')  // Image buffer
  }
});
```

### 8. Typing Indicator

```javascript
// Show typing status
await sock.presenceSubscribe('6281234567890@s.whatsapp.net');
await sock.sendPresenceUpdate('typing', '6281234567890@s.whatsapp.net');

// After finishing, send 'available'
setTimeout(() => {
  sock.sendPresenceUpdate('available', '6281234567890@s.whatsapp.net');
}, 3000);
```

---

## 🖼️ Media & File

### 1. Mengirim Gambar

```javascript
import fs from 'fs';

// From local file
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  image: fs.readFileSync('path/to/image.jpg'),
  caption: 'Ini adalah caption untuk gambar'
});

// From URL
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  image: { url: 'https://example.com/image.jpg' },
  caption: 'Gambar dari URL'
});

// From Buffer
const imageBuffer = Buffer.from(imageData);
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  image: imageBuffer,
  caption: 'Gambar dari buffer'
});
```

### 2. Mengirim Video

```javascript
// Local video
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  video: fs.readFileSync('path/to/video.mp4'),
  caption: 'Ini adalah video',
  gifPlayback: false  // Set true untuk video loop
});

// Video dari URL
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  video: { url: 'https://example.com/video.mp4' },
  caption: 'Video dari URL'
});
```

### 3. Mengirim Audio

```javascript
// Audio/Musik
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  audio: fs.readFileSync('path/to/audio.mp3'),
  mimetype: 'audio/mpeg',
  ptt: false  // Set true untuk voice note
});

// Voice Note (PTT)
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  audio: fs.readFileSync('path/to/voice.ogg'),
  mimetype: 'audio/ogg; codecs=opus',
  ptt: true  // Mark sebagai voice note
});
```

### 4. Mengirim Document/File

```javascript
// Document
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  document: fs.readFileSync('path/to/document.pdf'),
  mimetype: 'application/pdf',
  fileName: 'My Document.pdf'
});

// Text file
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  document: Buffer.from('File content here'),
  mimetype: 'text/plain',
  fileName: 'text.txt'
});
```

### 5. Mengirim Sticker

```javascript
// Sticker dari file
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  sticker: fs.readFileSync('path/to/sticker.webp'),
  pack: 'My Bot',        // Sticker pack name
  author: 'FazzCode',    // Sticker author
  categories: ['😀', '👍']  // Sticker categories
});

// Sticker dari URL
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  sticker: { url: 'https://example.com/sticker.webp' },
  pack: 'My Bot',
  author: 'FazzCode'
});
```

### 6. Mengirim Kontak

```javascript
// Send contact
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  contacts: {
    displayName: 'My Contact',
    contacts: [
      {
        vcard: `BEGIN:VCARD\nVERSION:3.0\nFN:John Doe\nTEL;type=CELL;type=VOICE;waid=6281234567890:+62 812 3456 7890\nEND:VCARD`
      }
    ]
  }
});
```

### 7. Mengirim Location

```javascript
// Send location
await sock.sendMessage('6281234567890@s.whatsapp.net', {
  location: {
    degreesLatitude: -6.2088,
    degreesLongitude: 106.8456,
    name: 'Jakarta, Indonesia',
    address: 'Jalan Sudirman'
  }
});
```

### 8. Mengunduh Media dari Pesan

```javascript
import { downloadMediaMessage } from '@fazzcode/baileys';

sock.ev.on('messages.upsert', async ({ messages }) => {
  for (const msg of messages) {
    const messageType = Object.keys(msg.message || {})[0];
    
    if (messageType === 'imageMessage' || 
        messageType === 'videoMessage' ||
        messageType === 'audioMessage' ||
        messageType === 'documentMessage') {
      
      try {
        const buffer = await downloadMediaMessage(
          msg,
          'buffer',  // Download type: buffer, stream, filename
          {},
          {
            logger,
            reuploadRequest: sock.updateMediaMessage
          }
        );
        
        // Save to file
        fs.writeFileSync(`downloaded_media`, buffer);
        console.log('✅ Media downloaded');
      } catch (error) {
        console.error('❌ Download failed:', error);
      }
    }
  }
});
```

---

## 👥 Manajemen Grup

### 1. Create Group

```javascript
// Create new group
const groupInfo = await sock.groupCreate(
  'Group Name',  // Group name
  [
    '6281234567890@s.whatsapp.net',
    '6289876543210@s.whatsapp.net'
  ],  // Participants
  'Group description'  // Description (optional)
);

console.log('✅ Group created:', groupInfo.gid);
```

### 2. Join/Leave Group

```javascript
// Leave group
await sock.groupLeave('120363023179xxxxxx-1234567890@g.us');

// Accept group invite (automatic on response)
// Note: Untuk join melalui link, gunakan invite code
await sock.groupAcceptInvite('invite-code');
```

### 3. Kelola Anggota

```javascript
// Add members
await sock.groupParticipantsUpdate(
  '120363023179xxxxxx-1234567890@g.us',
  ['6281234567890@s.whatsapp.net'],
  'add'
);

// Remove members
await sock.groupParticipantsUpdate(
  '120363023179xxxxxx-1234567890@g.us',
  ['6281234567890@s.whatsapp.net'],
  'remove'
);

// Promote to admin
await sock.groupParticipantsUpdate(
  '120363023179xxxxxx-1234567890@g.us',
  ['6281234567890@s.whatsapp.net'],
  'promote'
);

// Demote admin
await sock.groupParticipantsUpdate(
  '120363023179xxxxxx-1234567890@g.us',
  ['6281234567890@s.whatsapp.net'],
  'demote'
);
```

### 4. Update Group Info

```javascript
// Update group subject (name)
await sock.groupUpdateSubject(
  '120363023179xxxxxx-1234567890@g.us',
  'New Group Name'
);

// Update group description
await sock.groupUpdateDescription(
  '120363023179xxxxxx-1234567890@g.us',
  'New group description'
);

// Update group icon
await sock.updateGroupPicture(
  '120363023179xxxxxx-1234567890@g.us',
  fs.readFileSync('path/to/image.jpg')
);

// Update group settings
await sock.groupSettingUpdate(
  '120363023179xxxxxx-1234567890@g.us',
  'announcement'  // Only admins can send messages
);

// Reset invite link
await sock.groupInviteCode('120363023179xxxxxx-1234567890@g.us');
```

### 5. Get Group Info

```javascript
// Get group metadata
const metadata = await sock.groupMetadata(
  '120363023179xxxxxx-1234567890@g.us'
);

console.log({
  name: metadata.subject,
  description: metadata.desc,
  participants: metadata.participants.length,
  admins: metadata.participants.filter(p => p.admin).map(p => p.id),
  owner: metadata.owner,
  created: new Date(metadata.creation * 1000)
});
```

### 6. Kelola Group Admin

```javascript
// Grant admin role
await sock.groupParticipantsUpdate(
  '120363023179xxxxxx-1234567890@g.us',
  ['6281234567890@s.whatsapp.net'],
  'promote'
);

// Remove admin role
await sock.groupParticipantsUpdate(
  '120363023179xxxxxx-1234567890@g.us',
  ['6281234567890@s.whatsapp.net'],
  'demote'
);
```

---

## ⚠️ Error Handling

### Basic Error Handling

```javascript
import Boom from '@hapi/boom';

sock.ev.on('connection.update', (update) => {
  const { connection, lastDisconnect } = update;

  if (connection === 'close') {
    const error = lastDisconnect?.error;
    
    if (error instanceof Boom) {
      const statusCode = error.output.statusCode;
      
      switch (statusCode) {
        case 401:  // Unauthorized
          console.log('❌ Login failed');
          break;
        case 408:  // Connection lost
          console.log('🔌 Connection lost, reconnecting...');
          startBot();
          break;
        case 411:  // Multi-device mismatch
          console.log('⚠️ Multi-device mismatch');
          break;
        case 503:  // Service unavailable
          console.log('🔧 WhatsApp service unavailable');
          break;
        default:
          console.log(`Unknown error: ${statusCode}`);
      }
    }
  }
});
```

### Message Sending Error

```javascript
const sendMessageSafe = async (jid, content) => {
  try {
    const result = await sock.sendMessage(jid, content);
    console.log('✅ Message sent');
    return result;
  } catch (error) {
    console.error('❌ Send failed:', error.message);
    
    if (error.message.includes('401')) {
      console.log('Re-login required');
    } else if (error.message.includes('rate limited')) {
      console.log('⏱️ Rate limited, retrying later...');
      // Implement exponential backoff
    }
    
    return null;
  }
};
```

### Retry Logic dengan Exponential Backoff

```javascript
const sendMessageWithRetry = async (jid, content, maxRetries = 3) => {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await sock.sendMessage(jid, content);
    } catch (error) {
      const delay = Math.pow(2, i) * 1000;  // 1s, 2s, 4s
      
      if (i < maxRetries - 1) {
        console.log(`Retry ${i + 1}/${maxRetries} after ${delay}ms`);
        await new Promise(resolve => setTimeout(resolve, delay));
      } else {
        throw error;
      }
    }
  }
};
```

---

## 🎯 Best Practices

### 1. Authentication & Pairing Code

```javascript
// ⭐ IMPORTANT: Request pairing code immediately dengan delay 1.5s
setTimeout(async () => {
  try {
    const pairingCode = await sock.requestPairingCode('6281234567890');
    console.log('Pairing Code:', pairingCode);
  } catch (error) {
    console.error('Failed to get pairing code:', error);
  }
}, 1500);
```

### 2. Browser Configuration untuk Stability

```javascript
// Use correct browser settings untuk avoid detection/blocking
const sock = makeWASocket({
  // ✅ RECOMMENDED FOR STABILITY
  browser: ['Ubuntu', 'Chrome', '20.0.04'],
  
  // Alternative options
  // browser: Browsers.macOS('Safari'),
  // browser: Browsers.windows('Edge'),
  // browser: Browsers.iOS('Safari'),  // iPhone
  
  // Other important settings
  connectTimeoutMs: 20000,
  keepAliveIntervalMs: 30000,
  enableAutoSessionRecreation: true
});
```

### 3. Prevent WhatsApp Error 428 (Connection Replaced)

```javascript
// Use exponential backoff untuk reconnect
let reconnectAttempts = 0;
const maxReconnectAttempts = 5;

const handleDisconnect = async () => {
  reconnectAttempts++;
  
  if (reconnectAttempts > maxReconnectAttempts) {
    console.error('❌ Max reconnect attempts reached');
    process.exit(1);
  }
  
  const delay = Math.pow(2, reconnectAttempts) * 1000;
  console.log(`Reconnecting after ${delay}ms (attempt ${reconnectAttempts})`);
  
  await new Promise(resolve => setTimeout(resolve, delay));
  startBot();
};

sock.ev.on('connection.update', ({ connection, lastDisconnect }) => {
  if (connection === 'close') {
    const statusCode = lastDisconnect?.error?.output?.statusCode;
    
    if (statusCode === 428) {  // Connection replaced
      console.log('⚠️ Connection replaced, initiating reconnect...');
      handleDisconnect();
    }
  }
});
```

### 4. Graceful Shutdown

```javascript
const gracefulShutdown = async () => {
  console.log('\n📴 Shutting down...');
  
  try {
    // Close connection gracefully
    await sock.end();
    console.log('✅ Connection closed');
  } catch (error) {
    console.error('Error during shutdown:', error);
  }
  
  process.exit(0);
};

process.on('SIGINT', gracefulShutdown);
process.on('SIGTERM', gracefulShutdown);
```

### 5. Message Processing

```javascript
// Filter empty/invalid messages
const isValidMessage = (msg) => {
  return msg?.message && msg?.key?.remoteJid;
};

// Extract message content safely
const getMessageContent = (msg) => {
  const messageTypes = {
    conversation: msg.message?.conversation,
    extendedText: msg.message?.extendedTextMessage?.text,
    imageCaption: msg.message?.imageMessage?.caption,
    videoCaption: msg.message?.videoMessage?.caption
  };
  
  return Object.values(messageTypes).find(content => content?.trim());
};

// Check if message from self
const isFromMe = (msg) => msg.key.fromMe;

// Check if group message
const isGroupMessage = (msg) => msg.key.remoteJid.endsWith('@g.us');
```

### 6. Rate Limiting

```javascript
import PQueue from 'p-queue';

const messageQueue = new PQueue({
  concurrency: 1,          // Send 1 message at a time
  interval: 1000,          // Per 1 second
  carryoverConcurrencyCount: false
});

// Send message dengan queue
const sendMessageQueued = async (jid, content) => {
  return messageQueue.add(() => 
    sock.sendMessage(jid, content)
  );
};
```

### 7. Database Integration (Contoh: SQLite)

```javascript
import Database from 'better-sqlite3';

const db = new Database('bot.db');

// Create tables
db.exec(`
  CREATE TABLE IF NOT EXISTS messages (
    id TEXT PRIMARY KEY,
    jid TEXT,
    text TEXT,
    timestamp INTEGER,
    fromMe BOOLEAN
  );
  
  CREATE TABLE IF NOT EXISTS groups (
    jid TEXT PRIMARY KEY,
    name TEXT,
    members INTEGER,
    lastUpdated INTEGER
  );
`);

// Save message
const saveMessage = (msg) => {
  const stmt = db.prepare(`
    INSERT INTO messages (id, jid, text, timestamp, fromMe)
    VALUES (?, ?, ?, ?, ?)
  `);
  
  stmt.run(
    msg.key.id,
    msg.key.remoteJid,
    getMessageContent(msg),
    msg.messageTimestamp * 1000,
    msg.key.fromMe
  );
};

// Save group info
const saveGroupInfo = (jid, metadata) => {
  const stmt = db.prepare(`
    INSERT OR REPLACE INTO groups (jid, name, members, lastUpdated)
    VALUES (?, ?, ?, ?)
  `);
  
  stmt.run(
    jid,
    metadata.subject,
    metadata.participants.length,
    Date.now()
  );
};
```

---

## 📝 Contoh Implementasi Lengkap

### Bot Echo Sederhana

```javascript
import makeWASocket, { 
  DisconnectReason, 
  useMultiFileAuthState,
  Browsers
} from '@fazzcode/baileys';
import { Boom } from '@hapi/boom';
import fs from 'fs';

const startBot = async () => {
  const { state, saveCreds } = await useMultiFileAuthState('auth_info');

  const sock = makeWASocket({
    auth: state,
    browser: ['Ubuntu', 'Chrome', '20.0.04'],
    printQRInTerminal: true,
    connectTimeoutMs: 20000,
    keepAliveIntervalMs: 30000,
    enableAutoSessionRecreation: true
  });

  // Save credentials
  sock.ev.on('creds.update', saveCreds);

  // Handle connection
  sock.ev.on('connection.update', async (update) => {
    const { connection, qr } = update;

    if (qr) {
      console.log('📱 Scan QR code untuk login');
    }

    if (connection === 'open') {
      console.log('✅ Bot online');
    }
  });

  // Handle messages
  sock.ev.on('messages.upsert', async ({ messages }) => {
    for (const msg of messages) {
      // Ignore own messages
      if (msg.key.fromMe) continue;

      // Ignore empty messages
      if (!msg.message) continue;

      const from = msg.key.remoteJid;
      const text = msg.message.conversation || 
                   msg.message.extendedTextMessage?.text || '';

      // Log
      console.log(`\n📬 [${new Date().toLocaleTimeString()}] ${from}:`);
      console.log(`   ${text}`);

      // Echo reply
      if (text) {
        await sock.sendMessage(from, {
          text: `Echo: ${text}`
        });
      }
    }
  });

  // Handle errors
  sock.ev.on('connection.update', async (update) => {
    const { connection, lastDisconnect } = update;

    if (connection === 'close') {
      const statusCode = lastDisconnect?.error?.output?.statusCode;

      if (statusCode === DisconnectReason.loggedOut) {
        console.log('👤 Logged out');
        return;
      }

      console.log(`❌ Disconnected (${statusCode}), reconnecting...`);
      setTimeout(startBot, 5000);
    }
  });
};

// Start
startBot().catch(console.error);
```

### Bot dengan Command System

```javascript
import makeWASocket, { useMultiFileAuthState } from '@fazzcode/baileys';

const PREFIX = '!';

const commands = {
  hello: async (sock, msg) => {
    await sock.sendMessage(msg.key.remoteJid, {
      text: '👋 Halo, ada yang bisa dibantu?'
    });
  },

  info: async (sock, msg) => {
    const text = `
📱 Bot Information:
- Name: Echo Bot
- Version: 1.0.0
- Status: Online
- Uptime: ${Math.floor(process.uptime())} seconds
    `;
    await sock.sendMessage(msg.key.remoteJid, { text });
  },

  help: async (sock, msg) => {
    const text = `
📚 Available Commands:
!hello    - Greeting
!info     - Bot info
!help     - Show this help
!ping     - Check latency
    `;
    await sock.sendMessage(msg.key.remoteJid, { text });
  },

  ping: async (sock, msg) => {
    const start = Date.now();
    await sock.sendMessage(msg.key.remoteJid, {
      text: `⏱️ Pong! (${Date.now() - start}ms)`
    });
  }
};

const startBot = async () => {
  const { state, saveCreds } = await useMultiFileAuthState('auth_info');

  const sock = makeWASocket({
    auth: state,
    browser: ['Ubuntu', 'Chrome', '20.0.04'],
    printQRInTerminal: true
  });

  sock.ev.on('creds.update', saveCreds);

  sock.ev.on('messages.upsert', async ({ messages }) => {
    for (const msg of messages) {
      if (msg.key.fromMe || !msg.message) continue;

      const text = msg.message.conversation?.trim() || '';

      if (!text.startsWith(PREFIX)) continue;

      const [command, ...args] = text.slice(PREFIX.length).split(' ');

      if (command in commands) {
        try {
          await commands[command](sock, msg);
        } catch (error) {
          console.error(`Error executing ${command}:`, error);
          await sock.sendMessage(msg.key.remoteJid, {
            text: '❌ Error executing command'
          });
        }
      }
    }
  });

  sock.ev.on('connection.update', async (update) => {
    if (update.connection === 'open') {
      console.log('✅ Bot online');
    }
  });
};

startBot().catch(console.error);
```

### Bot Group Manager

```javascript
const handleGroupMessage = async (sock, msg) => {
  const groupJid = msg.key.remoteJid;
  const sender = msg.key.participant || msg.key.remoteJid;
  const text = msg.message?.conversation || '';

  // Get group metadata
  const metadata = await sock.groupMetadata(groupJid);
  const isAdmin = metadata.admins?.includes(sender);

  if (!text.startsWith('!')) return;

  const [command, ...args] = text.slice(1).split(' ');

  switch (command) {
    case 'add':
      if (!isAdmin) {
        await sock.sendMessage(groupJid, {
          text: '❌ Hanya admin yang bisa menambah member'
        });
        return;
      }

      const phoneToAdd = args[0];
      if (!phoneToAdd) {
        await sock.sendMessage(groupJid, {
          text: '❌ Format: !add 62812345678'
        });
        return;
      }

      const jidToAdd = phoneToAdd + '@s.whatsapp.net';
      await sock.groupParticipantsUpdate(groupJid, [jidToAdd], 'add');
      await sock.sendMessage(groupJid, {
        text: `✅ ${phoneToAdd} telah ditambahkan`
      });
      break;

    case 'info':
      const info = `
👥 Group Info:
- Name: ${metadata.subject}
- Members: ${metadata.participants.length}
- Created: ${new Date(metadata.creation * 1000).toLocaleDateString()}
- Description: ${metadata.desc || 'N/A'}
      `;
      await sock.sendMessage(groupJid, { text: info });
      break;

    case 'members':
      const memberList = metadata.participants
        .slice(0, 10)
        .map((p, i) => `${i + 1}. ${p.id}`)
        .join('\n');

      await sock.sendMessage(groupJid, {
        text: `📋 Group Members (${metadata.participants.length}):\n${memberList}`
      });
      break;
  }
};
```

---

## 📚 Dokumentasi Lengkap

- **Event Types**: Connection, Message, Group, Chat, Contact, Presence
- **Message Types**: Text, Image, Video, Audio, Document, Sticker, Location, Contact
- **JID Format**: `number@s.whatsapp.net` (personal), `id@g.us` (group)
- **Message Status**: pending, sent, delivered, read, played
- **Error Codes**: 401 (Unauthorized), 408 (Lost), 411 (Device Mismatch), 503 (Service Down)

---

## 🔗 Contact

- **Baileys GitHub**: None
- **FazzCode Channel**: https://whatsapp.com/channel/0029Vb66VpnA2pLGOYBeuq0S

---

## ⚖️ License

MIT License - FazzCode © 2026

---

## ⚠️ Disclaimer

- Penggunaan Baileys harus sesuai dengan Terms of Service WhatsApp
- Jangan gunakan untuk spam atau aktivitas ilegal
- Bot bisa di-ban jika melanggar WhatsApp policies
- Use responsibly!

---

**Made with ❤️ by FazzCode**

Untuk support dan update terbaru, join channel Whatsapp: [FazzCodeStudio](https://whatsapp.com/channel/0029Vb66VpnA2pLGOYBeuq0S)
