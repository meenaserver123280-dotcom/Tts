#!/usr/bin/env python3
"""
Telegram Multi-Account Manager - Single File
Bot Token Based | Turso DB | Termux Compatible
"""

import os
import sys
import asyncio
import signal
import json
import time
from datetime import datetime
from pathlib import Path
import sqlite3

# ==================== IMPORTS ====================
try:
    import httpx
except ImportError:
    print("\033[91m✗ httpx not installed. Run: pip install httpx\033[0m")
    sys.exit(1)

try:
    from telethon import TelegramClient, events
    from telethon.errors import (
        PhoneCodeInvalidError, PhoneCodeExpiredError,
        SessionPasswordNeededError, PhoneNumberInvalidError,
        FloodWaitError
    )
    from telethon.tl.types import User, Chat, Channel
except ImportError:
    print("\033[91m✗ Telethon not installed. Run: pip install telethon\033[0m")
    sys.exit(1)

try:
    import turso_client
except ImportError:
    print("\033[91m✗ turso-client not installed. Run: pip install turso-client\033[0m")
    sys.exit(1)

# ==================== COLORS ====================
class Colors:
    RED = "\033[91m"; GREEN = "\033[92m"; YELLOW = "\033[93m"
    BLUE = "\033[94m"; MAGENTA = "\033[95m"; CYAN = "\033[96m"
    RESET = "\033[0m"; BOLD = "\033[1m"
    CLEAR = "\033[2J\033[H"

# ==================== CONFIGURATION ====================
# 🔴 IMPORTANT: Replace these with YOUR values

# Telegram Bot Token (from @BotFather)
BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"  # ← CHANGE THIS

# Telegram API - Get from https://my.telegram.org/apps
API_ID = 0  # ← CHANGE THIS (required for user account login via Telethon)
API_HASH = ""  # ← CHANGE THIS

# Turso DB - Get from https://turso.tech
TURSO_DB_URL = "libsql://your-database.turso.io"  # ← CHANGE THIS
TURSO_AUTH_TOKEN = ""  # ← CHANGE THIS (leave empty for local Turso)

SESSION_DIR = "sessions"
os.makedirs(SESSION_DIR, exist_ok=True)

# ==================== BOT API HELPER ====================
class BotAPI:
    """Helper class to interact with Telegram Bot API"""
    
    def __init__(self, token):
        self.token = token
        self.base_url = f"https://api.telegram.org/bot{token}"
        self.client = httpx.AsyncClient(timeout=30)
        self.me = None
        self.update_offset = 0
    
    async def get_me(self):
        """Get bot info"""
        resp = await self.client.get(f"{self.base_url}/getMe")
        data = resp.json()
        if data.get("ok"):
            self.me = data["result"]
            return self.me
        return None
    
    async def send_message(self, chat_id, text, parse_mode="Markdown"):
        """Send message via bot"""
        resp = await self.client.post(f"{self.base_url}/sendMessage", json={
            "chat_id": chat_id,
            "text": text,
            "parse_mode": parse_mode
        })
        return resp.json()
    
    async def send_otp(self, chat_id, otp_code):
        """Send OTP code to user"""
        return await self.send_message(
            chat_id, 
            f"🔑 *Your OTP Code:* `{otp_code}`\n\nEnter this code in the terminal.",
            "Markdown"
        )
    
    async def send_login_code(self, phone, chat_id):
        """Send login code via bot"""
        # This simulates sending a code request
        return await self.send_message(
            chat_id,
            f"📱 *Login Request*\nPhone: `{phone}`\n\nCheck your Telegram app for the login code.",
            "Markdown"
        )
    
    async def get_updates(self):
        """Get bot updates (polling)"""
        resp = await self.client.get(
            f"{self.base_url}/getUpdates",
            params={
                "offset": self.update_offset,
                "timeout": 30,
                "allowed_updates": json.dumps(["message", "callback_query"])
            }
        )
        data = resp.json()
        if data.get("ok") and data.get("result"):
            updates = data["result"]
            if updates:
                self.update_offset = updates[-1]["update_id"] + 1
                return updates
        return []
    
    async def close(self):
        """Close HTTP client"""
        await self.client.aclose()

# ==================== DATABASE (Turso) ====================
class Database:
    def __init__(self):
        """Initialize Turso DB connection"""
        try:
            if TURSO_AUTH_TOKEN:
                self.client = turso_client.TursoClient(
                    url=TURSO_DB_URL,
                    auth_token=TURSO_AUTH_TOKEN
                )
            else:
                self.client = turso_client.TursoClient(url=TURSO_DB_URL)
            
            print(f"{Colors.GREEN}✓ Turso DB connected{Colors.RESET}")
            self._create_tables()
        except Exception as e:
            print(f"{Colors.RED}✗ Turso DB Error: {e}{Colors.RESET}")
            print(f"{Colors.YELLOW}Make sure TURSO_DB_URL and TURSO_AUTH_TOKEN are correct{Colors.RESET}")
            sys.exit(1)
    
    def _create_tables(self):
        """Create tables if not exist"""
        queries = [
            """CREATE TABLE IF NOT EXISTS accounts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                phone TEXT UNIQUE NOT NULL,
                phone_number TEXT UNIQUE NOT NULL,
                session_name TEXT,
                user_id INTEGER,
                first_name TEXT,
                last_name TEXT,
                username TEXT,
                added_on TEXT DEFAULT (datetime('now')),
                last_active TEXT,
                is_active INTEGER DEFAULT 1
            )""",
            """CREATE TABLE IF NOT EXISTS messages (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                phone TEXT NOT NULL,
                chat_id INTEGER,
                chat_name TEXT,
                message TEXT,
                direction TEXT,
                timestamp TEXT DEFAULT (datetime('now'))
            )""",
            """CREATE TABLE IF NOT EXISTS contacts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                phone TEXT NOT NULL,
                chat_id INTEGER NOT NULL,
                chat_name TEXT,
                chat_type TEXT,
                UNIQUE(phone, chat_id)
            )""",
            """CREATE TABLE IF NOT EXISTS bot_settings (
                key TEXT PRIMARY KEY,
                value TEXT
            )"""
        ]
        
        for query in queries:
            try:
                self.client.execute(query)
            except Exception as e:
                print(f"{Colors.YELLOW}⚠ Table create warning: {e}{Colors.RESET}")
    
    def save_account(self, phone, session_name=None, user_info=None):
        """Save account"""
        clean_phone = phone.replace("+", "").strip()
        
        # Check existing
        result = self.client.execute(
            "SELECT id, is_active FROM accounts WHERE phone = ?", [phone]
        )
        rows = result.rows if hasattr(result, 'rows') else result.fetchall()
        
        if rows:
            row = rows[0]
            if row[1] == 0:  # Reactivate
                self.client.execute(
                    "UPDATE accounts SET is_active = 1, session_name = ? WHERE id = ?",
                    [session_name or "", row[0]]
                )
            else:
                # Update session
                self.client.execute(
                    "UPDATE accounts SET session_name = ? WHERE phone = ?",
                    [session_name or "", phone]
                )
            
            # Update user info if provided
            if user_info:
                self.client.execute(
                    """UPDATE accounts SET user_id = ?, first_name = ?, 
                       last_name = ?, username = ? WHERE phone = ?""",
                    [user_info.get('id'), user_info.get('first_name'),
                     user_info.get('last_name', ''), user_info.get('username', ''),
                     phone]
                )
        else:
            # New account
            self.client.execute(
                """INSERT INTO accounts 
                   (phone, phone_number, session_name, user_id, first_name, last_name, username) 
                   VALUES (?, ?, ?, ?, ?, ?, ?)""",
                [phone, clean_phone, session_name or "",
                 user_info.get('id') if user_info else None,
                 user_info.get('first_name') if user_info else None,
                 user_info.get('last_name', '') if user_info else None,
                 user_info.get('username', '') if user_info else None]
            )
        
        return True
    
    def get_accounts(self):
        """Get all active accounts"""
        result = self.client.execute(
            "SELECT * FROM accounts WHERE is_active = 1 ORDER BY added_on DESC"
        )
        rows = result.rows if hasattr(result, 'rows') else result.fetchall()
        
        accounts = []
        for row in rows:
            accounts.append({
                'id': row[0], 'phone': row[1], 'phone_number': row[2],
                'session_name': row[3], 'user_id': row[4],
                'first_name': row[5], 'last_name': row[6],
                'username': row[7], 'added_on': row[8],
                'last_active': row[9], 'is_active': bool(row[10])
            })
        return accounts
    
    def get_account_by_phone(self, phone):
        """Get single account"""
        result = self.client.execute(
            "SELECT * FROM accounts WHERE phone = ? AND is_active = 1", [phone]
        )
        rows = result.rows if hasattr(result, 'rows') else result.fetchall()
        if rows:
            row = rows[0]
            return {
                'id': row[0], 'phone': row[1], 'phone_number': row[2],
                'session_name': row[3], 'user_id': row[4],
                'first_name': row[5], 'last_name': row[6],
                'username': row[7], 'added_on': row[8],
                'last_active': row[9], 'is_active': bool(row[10])
            }
        return None
    
    def update_last_active(self, phone):
        """Update last active"""
        self.client.execute(
            "UPDATE accounts SET last_active = datetime('now') WHERE phone = ?", [phone]
        )
    
    def remove_account(self, phone):
        """Soft delete account"""
        self.client.execute(
            "UPDATE accounts SET is_active = 0 WHERE phone = ?", [phone]
        )
    
    def log_message(self, phone, chat_id, chat_name, message, direction):
        """Log message"""
        self.client.execute(
            """INSERT INTO messages (phone, chat_id, chat_name, message, direction) 
               VALUES (?, ?, ?, ?, ?)""",
            [phone, chat_id, chat_name[:100] if chat_name else "", 
             message[:200] if message else "", direction]
        )
    
    def save_contact(self, phone, chat_id, chat_name, chat_type):
        """Save contact"""
        self.client.execute(
            """INSERT OR IGNORE INTO contacts (phone, chat_id, chat_name, chat_type) 
               VALUES (?, ?, ?, ?)""",
            [phone, chat_id, chat_name[:100] if chat_name else "", chat_type]
        )
    
    def get_contacts(self, phone=None):
        """Get contacts"""
        if phone:
            result = self.client.execute(
                "SELECT * FROM contacts WHERE phone = ? ORDER BY chat_name", [phone]
            )
        else:
            result = self.client.execute(
                "SELECT * FROM contacts ORDER BY chat_name"
            )
        rows = result.rows if hasattr(result, 'rows') else result.fetchall()
        
        contacts = []
        for row in rows:
            contacts.append({
                'id': row[0], 'phone': row[1], 'chat_id': row[2],
                'chat_name': row[3], 'chat_type': row[4]
            })
        return contacts
    
    def get_stats(self):
        """Get stats"""
        result1 = self.client.execute(
            "SELECT COUNT(*) FROM accounts WHERE is_active = 1"
        )
        result2 = self.client.execute("SELECT COUNT(*) FROM messages")
        
        r1 = result1.rows if hasattr(result1, 'rows') else result1.fetchall()
        r2 = result2.rows if hasattr(result2, 'rows') else result2.fetchall()
        
        return {
            'accounts': r1[0][0] if r1 else 0,
            'messages': r2[0][0] if r2 else 0
        }
    
    def close(self):
        """Close connection"""
        try:
            self.client.close()
        except:
            pass

# ==================== ACCOUNT MANAGER ====================
class AccountManager:
    def __init__(self, db):
        self.db = db
        self.clients = {}  # phone -> TelegramClient
        self.active_account = None
    
    def _get_session_path(self, phone):
        """Get session file path"""
        clean = phone.replace("+", "").strip()
        return f"{SESSION_DIR}/session_{clean}"
    
    async def load_account(self, phone):
        """Load existing Telethon session"""
        session_path = self._get_session_path(phone)
        client = TelegramClient(session_path, API_ID, API_HASH)
        
        try:
            await client.connect()
            if await client.is_user_authorized():
                me = await client.get_me()
                self.clients[phone] = client
                print(f"{Colors.GREEN}✓ Loaded: {me.first_name} ({phone}){Colors.RESET}")
                
                # Update DB
                self.db.save_account(phone, session_path, {
                    'id': me.id, 'first_name': me.first_name,
                    'last_name': me.last_name or '', 'username': me.username or ''
                })
                self.db.update_last_active(phone)
                
                return client
            else:
                print(f"{Colors.YELLOW}⚠ Session expired for {phone}{Colors.RESET}")
                await client.disconnect()
                return None
        except Exception as e:
            print(f"{Colors.RED}✗ Error loading {phone}: {e}{Colors.RESET}")
            return None
    
    async def load_all_accounts(self):
        """Load all saved accounts"""
        accounts = self.db.get_accounts()
        if not accounts:
            print(f"{Colors.YELLOW}⚠ No saved accounts{Colors.RESET}")
            return 0
        
        print(f"\n{Colors.BOLD}{Colors.BLUE}Loading {len(accounts)} accounts...{Colors.RESET}\n")
        
        loaded = 0
        for acc in accounts:
            client = await self.load_account(acc['phone'])
            if client:
                loaded += 1
        
        if loaded > 0:
            self.active_account = list(self.clients.keys())[0]
            me = await self.clients[self.active_account].get_me()
            print(f"\n{Colors.GREEN}✓ Active: {me.first_name} ({self.active_account}){Colors.RESET}")
        
        return loaded
    
    async def login_new_account(self, phone):
        """Login a new account via Telethon"""
        session_path = self._get_session_path(phone)
        client = TelegramClient(session_path, API_ID, API_HASH)
        
        await client.connect()
        
        if await client.is_user_authorized():
            me = await client.get_me()
            print(f"{Colors.GREEN}✓ Already logged in as {me.first_name}{Colors.RESET}")
            self.clients[phone] = client
            self.db.save_account(phone, session_path, {
                'id': me.id, 'first_name': me.first_name,
                'last_name': me.last_name or '', 'username': me.username or ''
            })
            return client
        
        try:
            # Send OTP
            print(f"\n{Colors.YELLOW}📱 Sending OTP to {phone}...{Colors.RESET}")
            sent = await client.send_code_request(phone)
            print(f"{Colors.GREEN}✓ OTP sent! ({sent.code_length} digits){Colors.RESET}")
            
            # Get OTP
            otp = input(f"{Colors.CYAN}🔑 Enter OTP: {Colors.RESET}").strip()
            
            try:
                await client.sign_in(phone=phone, code=otp)
                me = await client.get_me()
                print(f"{Colors.GREEN}✓ Welcome {me.first_name}!{Colors.RESET}")
                
            except SessionPasswordNeededError:
                # 2FA
                print(f"\n{Colors.YELLOW}🔒 Two-step verification enabled{Colors.RESET}")
                password = input(f"{Colors.CYAN}🔐 Enter 2FA password: {Colors.RESET}").strip()
                await client.sign_in(password=password)
                me = await client.get_me()
                print(f"{Colors.GREEN}✓ 2FA passed! Welcome {me.first_name}{Colors.RESET}")
            
            # Save
            self.clients[phone] = client
            self.db.save_account(phone, session_path, {
                'id': me.id, 'first_name': me.first_name,
                'last_name': me.last_name or '', 'username': me.username or ''
            })
            
            return client
            
        except PhoneCodeInvalidError:
            print(f"{Colors.RED}✗ Invalid OTP{Colors.RESET}")
            return None
        except PhoneCodeExpiredError:
            print(f"{Colors.RED}✗ OTP expired{Colors.RESET}")
            return None
        except PhoneNumberInvalidError:
            print(f"{Colors.RED}✗ Invalid phone number{Colors.RESET}")
            return None
        except FloodWaitError as e:
            print(f"{Colors.RED}✗ Wait {e.seconds}s{Colors.RESET}")
            return None
        except Exception as e:
            print(f"{Colors.RED}✗ Error: {e}{Colors.RESET}")
            return None
    
    async def switch_account(self, phone):
        """Switch active account"""
        if phone in self.clients:
            self.active_account = phone
            self.db.update_last_active(phone)
            me = await self.clients[phone].get_me()
            print(f"{Colors.GREEN}✓ Switched to {me.first_name} ({phone}){Colors.RESET}")
            return True
        print(f"{Colors.RED}✗ Account not found{Colors.RESET}")
        return False
    
    async def get_active_me(self):
        """Get active account info"""
        if self.active_account and self.active_account in self.clients:
            return await self.clients[self.active_account].get_me()
        return None
    
    async def get_dialogs(self, phone=None, force_refresh=False):
        """Get all chats for account"""
        phone = phone or self.active_account
        if not phone or phone not in self.clients:
            print(f"{Colors.RED}✗ No active account{Colors.RESET}")
            return []
        
        client = self.clients[phone]
        
        try:
            print(f"{Colors.YELLOW}Fetching chats...{Colors.RESET}")
            dialogs = await client.get_dialogs()
            contacts = []
            
            for dialog in dialogs:
                entity = dialog.entity
                if isinstance(entity, User):
                    ctype = "🤖 Bot" if entity.bot else "👤 User"
                elif isinstance(entity, Chat):
                    ctype = "👥 Group"
                elif isinstance(entity, Channel):
                    ctype = "👥 Supergroup" if entity.megagroup else "📢 Channel"
                else:
                    ctype = "❓ Unknown"
                
                contacts.append({
                    'id': dialog.id,
                    'name': dialog.name or "Unknown",
                    'entity': entity,
                    'type': ctype,
                    'unread': dialog.unread_count,
                    'message': dialog.message.text[:60] if dialog.message and dialog.message.text else ""
                })
                
                # Save to DB
                self.db.save_contact(phone, dialog.id, dialog.name, ctype)
            
            return contacts
            
        except Exception as e:
            print(f"{Colors.RED}✗ Error: {e}{Colors.RESET}")
            return []
    
    async def send_message(self, phone, target, message):
        """Send message via Telethon"""
        if phone not in self.clients:
            print(f"{Colors.RED}✗ Account not loaded{Colors.RESET}")
            return False
        
        client = self.clients[phone]
        
        try:
            entity = await client.get_entity(target)
            await client.send_message(entity, message)
            
            display = getattr(entity, 'title', None) or getattr(entity, 'first_name', str(target))
            self.db.log_message(phone, entity.id, display, message, 'sent')
            
            print(f"{Colors.GREEN}✓ Message sent to {display}{Colors.RESET}")
            return True
        except ValueError:
            print(f"{Colors.RED}✗ Chat '{target}' not found{Colors.RESET}")
            return False
        except Exception as e:
            print(f"{Colors.RED}✗ Error: {e}{Colors.RESET}")
            return False
    
    async def broadcast(self, message, target_type="all"):
        """Broadcast to all chats across all accounts"""
        results = []
        for phone, client in self.clients.items():
            try:
                dialogs = await client.get_dialogs()
                count = 0
                for dialog in dialogs:
                    entity = dialog.entity
                    if target_type == "users" and not isinstance(entity, User):
                        continue
                    elif target_type == "groups" and not (isinstance(entity, Chat) or (isinstance(entity, Channel) and entity.megagroup)):
                        continue
                    
                    try:
                        await client.send_message(entity, message)
                        count += 1
                        await asyncio.sleep(0.3)
                    except:
                        continue
                
                results.append({"phone": phone, "sent": count})
                print(f"{Colors.GREEN}✓ {phone}: Sent to {count}{Colors.RESET}")
            except Exception as e:
                print(f"{Colors.RED}✗ {phone}: {e}{Colors.RESET}")
        
        return results
    
    async def logout(self, phone):
        """Logout and remove session"""
        if phone in self.clients:
            try:
                await self.clients[phone].log_out()
                await self.clients[phone].disconnect()
                del self.clients[phone]
            except:
                pass
            
            # Remove session file
            session_path = self._get_session_path(phone)
            for f in [session_path, session_path + ".session"]:
                if os.path.exists(f):
                    os.remove(f)
            
            self.db.remove_account(phone)
            
            if self.active_account == phone:
                self.active_account = list(self.clients.keys())[0] if self.clients else None
            
            print(f"{Colors.GREEN}✓ Logged out {phone}{Colors.RESET}")
            return True
        return False
    
    def close_all(self):
        """Close all clients"""
        for phone, client in self.clients.items():
            try:
                client.disconnect()
            except:
                pass
        self.clients.clear()

# ==================== BOT COMMANDS ====================
async def handle_bot_commands(bot, db, manager):
    """Handle incoming bot commands"""
    while True:
        try:
            updates = await bot.get_updates()
            
            for update in updates:
                if 'message' not in update:
                    continue
                
                msg = update['message']
                chat_id = msg['chat']['id']
                text = msg.get('text', '')
                user = msg.get('from_user', {})
                username = user.get('username', 'Unknown')
                
                if text.startswith('/start'):
                    welcome = f"""
🎉 *Welcome to Telegram Multi-Account Manager!*

I can help you manage multiple Telegram accounts from Termux.

*Available Commands:*
📱 `/add +911234567890` - Add new account
📋 `/list` - List all accounts
👥 `/chats` - View your chats
💬 `/send @username Hello` - Send message
📢 `/broadcast Hello everyone!` - Broadcast
🗑 `/remove +911234567890` - Remove account
📊 `/stats` - View statistics
ℹ️ `/help` - Show this help

*Setup:*
1. Set BOT_TOKEN in the script
2. Run the script
3. Use commands here
                    """
                    await bot.send_message(chat_id, welcome)
                
                elif text.startswith('/add'):
                    parts = text.split()
                    if len(parts) < 2:
                        await bot.send_message(chat_id, "❌ Usage: `/add +911234567890`")
                        continue
                    
                    phone = parts[1]
                    if not phone.startswith('+'):
                        phone = '+' + phone
                    
                    await bot.send_message(chat_id, f"📱 Adding account {phone}...\nOpen the Termux terminal to enter OTP.")
                    
                    # This will be handled by Termux terminal input
                    await bot.send_message(
                        chat_id,
                        f"🔄 Check Termux terminal. Enter the OTP when prompted.\nPhone: `{phone}`"
                    )
                    
                    # Note: OTP input happens in terminal, not via bot
                    # The bot just notifies
                
                elif text == '/list':
                    accounts = db.get_accounts()
                    if not accounts:
                        await bot.send_message(chat_id, "📋 No accounts added yet.\nUse `/add +911234567890`")
                    else:
                        msg_text = "📋 *Your Accounts:*\n\n"
                        for acc in accounts:
                            name = acc.get('first_name') or acc.get('phone')
                            status = "✅" if acc['phone'] in manager.clients else "❌"
                            msg_text += f"{status} `{acc['phone']}` - {name}\n"
                        msg_text += f"\nTotal: {len(accounts)}"
                        await bot.send_message(chat_id, msg_text)
                
                elif text == '/chats':
                    if not manager.active_account:
                        await bot.send_message(chat_id, "❌ No active account. Add one first.")
                        continue
                    
                    await bot.send_message(chat_id, "👥 Fetching your chats...")
                    contacts = await manager.get_dialogs(force_refresh=True)
                    
                    if not contacts:
                        await bot.send_message(chat_id, "No chats found.")
                    else:
                        msg_text = f"👥 *Chats ({len(contacts)}):*\n\n"
                        for i, c in enumerate(contacts[:30], 1):
                            unread = f" ({c['unread']} new)" if c['unread'] > 0 else ""
                            msg_text += f"{i}. {c['type']} {c['name']}{unread}\n"
                        
                        if len(contacts) > 30:
                            msg_text += f"\n...and {len(contacts)-30} more"
                        
                        await bot.send_message(chat_id, msg_text)
                
                elif text.startswith('/send'):
                    parts = text.split(maxsplit=2)
                    if len(parts) < 3:
                        await bot.send_message(chat_id, "❌ Usage: `/send @username Your message here`")
                        continue
                    
                    target = parts[1]
                    message = parts[2]
                    
                    if not manager.active_account:
                        await bot.send_message(chat_id, "❌ No active account.")
                        continue
                    
                    await bot.send_message(chat_id, f"📤 Sending to {target}...")
                    success = await manager.send_message(manager.active_account, target, message)
                    
                    if success:
                        await bot.send_message(chat_id, "✅ Message sent!")
                    else:
                        await bot.send_message(chat_id, "❌ Failed to send message")
                
                elif text.startswith('/broadcast'):
                    if not manager.clients:
                        await bot.send_message(chat_id, "❌ No accounts loaded.")
                        continue
                    
                    message = text[11:].strip()
                    if not message:
                        await bot.send_message(chat_id, "❌ Usage: `/broadcast Your message`")
                        continue
                    
                    await bot.send_message(chat_id, "📢 Broadcasting message to all chats...")
                    results = await manager.broadcast(message)
                    
                    result_text = "📢 *Broadcast Results:*\n\n"
                    for r in results:
                        result_text += f"✅ `{r['phone']}`: Sent to {r['sent']} chats\n"
                    await bot.send_message(chat_id, result_text)
                
                elif text.startswith('/remove'):
                    parts = text.split()
                    if len(parts) < 2:
                        await bot.send_message(chat_id, "❌ Usage: `/remove +911234567890`")
                        continue
                    
                    phone = parts[1]
                    if not phone.startswith('+'):
                        phone = '+' + phone
                    
                    await manager.logout(phone)
                    await bot.send_message(chat_id, f"✅ Removed {phone}")
                
                elif text == '/stats':
                    stats = db.get_stats()
                    accounts = db.get_accounts()
                    active_now = len(manager.clients)
                    
                    msg_text = f"""
📊 *Bot Statistics*

📱 Total Accounts: {stats['accounts']}
🟢 Active Now: {active_now}
💬 Total Messages: {stats['messages']}

*Accounts:*
                    """
                    for acc in accounts:
                        name = acc.get('first_name') or 'Unknown'
                        status = "🟢" if acc['phone'] in manager.clients else "🔴"
                        msg_text += f"\n{status} {name} - `{acc['phone']}`"
                    
                    await bot.send_message(chat_id, msg_text)
                
                elif text == '/help':
                    help_text = """
🤖 *Available Commands:*

📱 `/add +911234567890` - Add Telegram account
   You'll enter OTP in Termux terminal

📋 `/list` - Show all linked accounts

🔄 `/switch +911234567890` - Switch active account

👥 `/chats` - Show chats for active account

💬 `/send @user Hello` - Send message

📢 `/broadcast Message` - Send to all chats

🗑 `/remove +911234567890` - Remove account

📊 `/stats` - System statistics

❌ `/stop` - Stop the bot

ℹ️ `/help` - Show this message

*Note:* OTP entry happens in Termux terminal, not via bot.
                    """
                    await bot.send_message(chat_id, help_text)
                
                elif text == '/stop':
                    await bot.send_message(chat_id, "👋 Stopping bot...")
                    os.kill(os.getpid(), signal.SIGINT)
            
            await asyncio.sleep(1)
            
        except Exception as e:
            print(f"{Colors.RED}✗ Bot update error: {e}{Colors.RESET}")
            await asyncio.sleep(5)

# ==================== TERMINAL UI ====================
def print_banner():
    """Print banner"""
    banner = f"""
{Colors.BOLD}{Colors.CYAN}╔══════════════════════════════════════════════╗
║       📱 Telegram Multi-Account Manager      ║
║       ────────────────────────────────        ║
║    Manage Unlimited Telegram Accounts         ║
║    Bot: @YourBotUsername                      ║
╚══════════════════════════════════════════════╝{Colors.RESET}
    """
    print(banner)

def print_menu():
    """Print menu"""
    menu = f"""
{Colors.BOLD}{Colors.BLUE}╔══════════════════════════════════════════╗
║              MAIN MENU                    ║
╠══════════════════════════════════════════╣
║  {Colors.GREEN}1.{Colors.RESET}  ➕ Add New Account              ║
║  {Colors.GREEN}2.{Colors.RESET}  📋 List All Accounts            ║
║  {Colors.GREEN}3.{Colors.RESET}  🔄 Switch Active Account        ║
║  {Colors.GREEN}4.{Colors.RESET}  👥 View Chats/Contacts          ║
║  {Colors.GREEN}5.{Colors.RESET}  💬 Send Message                 ║
║  {Colors.GREEN}6.{Colors.RESET}  📢 Broadcast Message            ║
║  {Colors.GREEN}7.{Colors.RESET}  🗑 Remove Account               ║
║  {Colors.GREEN}8.{Colors.RESET}  📊 Dashboard/Stats              ║
║  {Colors.GREEN}9.{Colors.RESET}  ❌ Exit (Bot continues)         ║
╚══════════════════════════════════════════╝{Colors.RESET}
    """
    print(menu)

# ==================== TERMINAL HANDLER ====================
async def terminal_menu(db, manager, bot):
    """Terminal menu handler"""
    while True:
        print("\033[H\033[J")  # Clear screen
        print_banner()
        
        # Show active account
        if manager.active_account and manager.active_account in manager.clients:
            try:
                me = await manager.get_active_me()
                print(f"{Colors.GREEN}✓ Active: {me.first_name} ({manager.active_account}){Colors.RESET}")
            except:
                print(f"{Colors.YELLOW}✓ Active: {manager.active_account}{Colors.RESET}")
            print(f"{Colors.DIM}Total accounts: {len(manager.clients)}{Colors.RESET}\n")
        else:
            print(f"{Colors.YELLOW}⚠ No active account{Colors.RESET}\n")
        
        print_menu()
        
        choice = input(f"{Colors.CYAN}Enter choice: {Colors.RESET}").strip()
        
        if choice == "1":
            phone = input(f"{Colors.CYAN}📞 Phone (+911234567890): {Colors.RESET}").strip()
            if not phone.startswith("+"):
                phone = "+" + phone
            
            await manager.login_new_account(phone)
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "2":
            accounts = db.get_accounts()
            if not accounts:
                print(f"{Colors.YELLOW}⚠ No accounts{Colors.RESET}")
            else:
                print(f"\n{Colors.BOLD}📋 Accounts:{Colors.RESET}")
                for acc in accounts:
                    name = acc.get('first_name') or "Unknown"
                    status = "🟢" if acc['phone'] in manager.clients else "🔴"
                    print(f"  {status} {name} - {acc['phone']}")
                print(f"\nTotal: {len(accounts)}")
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "3":
            phone = input(f"{Colors.CYAN}📞 Switch to phone: {Colors.RESET}").strip()
            if not phone.startswith("+"):
                phone = "+" + phone
            await manager.switch_account(phone)
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "4":
            contacts = await manager.get_dialogs(force_refresh=True)
            if contacts:
                print(f"\n{Colors.BOLD}👥 Chats ({len(contacts)}):{Colors.RESET}")
                for i, c in enumerate(contacts[:20], 1):
                    unread = f" ({c['unread']} new)" if c['unread'] > 0 else ""
                    print(f"  {i}. {c['type']} {c['name']}{unread}")
                if len(contacts) > 20:
                    print(f"  ...and {len(contacts)-20} more")
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "5":
            target = input(f"{Colors.CYAN}Target (@username/phone/id): {Colors.RESET}").strip()
            message = input(f"{Colors.CYAN}Message: {Colors.RESET}").strip()
            if manager.active_account:
                await manager.send_message(manager.active_account, target, message)
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "6":
            message = input(f"{Colors.CYAN}Broadcast message: {Colors.RESET}").strip()
            if message:
                ttype = input(f"{Colors.CYAN}Type (all/users/groups): {Colors.RESET}").strip() or "all"
                await manager.broadcast(message, ttype)
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "7":
            phone = input(f"{Colors.CYAN}📞 Remove phone: {Colors.RESET}").strip()
            if not phone.startswith("+"):
                phone = "+" + phone
            confirm = input(f"{Colors.RED}Remove {phone}? (y/n): {Colors.RESET}").strip()
            if confirm.lower() == 'y':
                await manager.logout(phone)
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "8":
            stats = db.get_stats()
            print(f"\n{Colors.BOLD}📊 Dashboard{Colors.RESET}")
            print(f"  📱 Total Accounts: {stats['accounts']}")
            print(f"  🟢 Active Now: {len(manager.clients)}")
            print(f"  💬 Messages Logged: {stats['messages']}")
            print(f"  🤖 Bot: Running")
            
            if manager.active_account:
                me = await manager.get_active_me()
                if me:
                    print(f"\n  Active: {me.first_name} (@{me.username or 'N/A'})")
            
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")
        
        elif choice == "9":
            print(f"{Colors.YELLOW}👋 Terminal closed. Bot continues in background.{Colors.RESET}")
            print(f"{Colors.YELLOW}Send /help to your bot for commands.{Colors.RESET}")
            break
        
        else:
            print(f"{Colors.RED}✗ Invalid choice{Colors.RESET}")
            input(f"{Colors.YELLOW}Press Enter...{Colors.RESET}")

# ==================== MAIN ====================
async def main():
    """Main function"""
    print(f"{Colors.CLEAR}{Colors.BOLD}{Colors.BLUE}Starting Telegram Multi-Account Manager...{Colors.RESET}")
    
    # Check config
    if BOT_TOKEN == "YOUR_BOT_TOKEN_HERE" or API_ID == 0:
        print(f"{Colors.RED}✗ Please configure BOT_TOKEN, API_ID, and API_HASH in the script{Colors.RESET}")
        print(f"{Colors.YELLOW}  - BOT_TOKEN: @BotFather on Telegram{Colors.RESET}")
        print(f"{Colors.YELLOW}  - API_ID/HASH: https://my.telegram.org/apps{Colors.RESET}")
        print(f"{Colors.YELLOW}  - TURSO_DB_URL: https://turso.tech{Colors.RESET}")
        sys.exit(1)
    
    # Initialize
    db = Database()
    manager = AccountManager(db)
    bot = BotAPI(BOT_TOKEN)
    
    # Check bot
    bot_info = await bot.get_me()
    if bot_info:
        print(f"{Colors.GREEN}✓ Bot @{bot_info['username']} connected{Colors.RESET}")
    else:
        print(f"{Colors.RED}✗ Bot connection failed. Check BOT_TOKEN{Colors.RESET}")
        sys.exit(1)
    
    # Load existing accounts
    loaded = await manager.load_all_accounts()
    print(f"\n{Colors.GREEN}✓ {loaded} account(s) loaded{Colors.RESET}")
    
    # Start bot polling in background
    bot_task = asyncio.create_task(handle_bot_commands(bot, db, manager))
    
    # Start terminal menu
    await terminal_menu(db, manager, bot)
    
    # Cleanup
    print(f"{Colors.YELLOW}Shutting down...{Colors.RESET}")
    bot_task.cancel()
    manager.close_all()
    db.close()
    await bot.close()
    print(f"{Colors.GREEN}✓ Done{Colors.RESET}")

def signal_handler(sig, frame):
    print(f"\n{Colors.YELLOW}Exiting...{Colors.RESET}")
    sys.exit(0)

if __name__ == "__main__":
    signal.signal(signal.SIGINT, signal_handler)
    
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print(f"\n{Colors.YELLOW}Bye!{Colors.RESET}")
    except Exception as e:
        print(f"\n{Colors.RED}Fatal error: {e}{Colors.RESET}")
