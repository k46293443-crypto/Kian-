# main.py - الملف الرئيسي للسورس
import asyncio
import os
import sys
import time
import random
import logging
import sqlite3
import arabic_reshaper
from bidi.algorithm import get_display
from datetime import datetime, timedelta
from PIL import Image, ImageDraw, ImageFont
import numpy as np

from telethon import TelegramClient, events, functions, types
from telethon.tl.types import MessageEntityMentionName, Channel, User
from telethon.tl.functions.channels import JoinChannelRequest, CreateChannelRequest, InviteToChannelRequest
from telethon.tl.functions.messages import CreateChatRequest, ExportChatInviteRequest
from telethon.errors import UserNotParticipantError, ChannelInvalidError
from telethon.tl.functions.contacts import BlockRequest

# إعدادات التسجيل
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger('سايثون')

# معلومات القنوات والمطور
CHANNEL = "k_ian_1"  # قناة السورس
GROUP = "k_ian_2"    # مجموعة السورس  
DEVELOPER = "T_TH8"  # يوزر المطور

class SythonBot:
    def __init__(self, api_id, api_hash, session_name='سايثون'):
        self.client = TelegramClient(session_name, api_id, api_hash)
        self.start_time = time.time()
        self.plugins = {}
        self.setup_complete = False
        self.log_group_id = None
        self.storage_group_id = None
        self.user_warnings = {}
        self.user_message_count = {}
        
        # إعدادات الأمان
        self.security_settings = {
            'auto_block_strangers': True,
            'warn_before_block': True,
            'max_warnings': 3,
            'block_after_timeout': True,
            'timeout_minutes': 5,
            'anti_spam_protection': True,
            'max_messages_per_minute': 5
        }
        
        # إنشاء مجلدات التخزين
        os.makedirs('data', exist_ok=True)
        os.makedirs('downloads', exist_ok=True)
        os.makedirs('data/profile_images', exist_ok=True)
        os.makedirs('data/group_images', exist_ok=True)
        
        self.setup_database()
        self.setup_handlers()
        
    def setup_database(self):
        """إعداد قاعدة البيانات للتخزين"""
        self.conn = sqlite3.connect('data/sython.db')
        self.cursor = self.conn.cursor()
        
        # إنشاء جدول للرسائل
        self.cursor.execute('''
            CREATE TABLE IF NOT EXISTS messages (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                message_text TEXT,
                message_date TIMESTAMP,
                chat_type TEXT
            )
        ''')
        
        # إنشاء جدول للملفات
        self.cursor.execute('''
            CREATE TABLE IF NOT EXISTS files (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                file_type TEXT,
                file_path TEXT,
                saved_date TIMESTAMP,
                auto_delete TIMESTAMP
            )
        ''')
        
        self.conn.commit()
        logger.info("✅ تم إعداد قاعدة البيانات")
    
    def setup_handlers(self):
        """إعداد معالجات الأحداث"""
        # أوامر الحساب
        @self.client.on(events.NewMessage(pattern=r'\.ايدي'))
        async def handler(event):
            await self.cmd_id(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.بروفايل'))
        async def handler(event):
            await self.cmd_profile(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.بينج'))
        async def handler(event):
            await self.cmd_ping(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.الاشتراك'))
        async def handler(event):
            await self.cmd_subscription(event)
        
        # أوامر الأدوات
        @self.client.on(events.NewMessage(pattern=r'\.تحميل (.*)'))
        async def handler(event):
            await self.cmd_download(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.ملصق'))
        async def handler(event):
            await self.cmd_sticker(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.صور (.*)'))
        async def handler(event):
            await self.cmd_images(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.ترجمة (.*)'))
        async def handler(event):
            await self.cmd_translate(event)
        
        # أوامر الترفيه
        @self.client.on(events.NewMessage(pattern=r'\.نكته'))
        async def handler(event):
            await self.cmd_joke(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.قيف'))
        async def handler(event):
            await self.cmd_gif(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.(قفل|فتح) الشات'))
        async def handler(event):
            await self.cmd_chat_control(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.كلمات متقاطعة'))
        async def handler(event):
            await self.cmd_crossword(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.سؤال يومي'))
        async def handler(event):
            await self.cmd_daily_question(event)
        
        # أوامر الإدارة
        @self.client.on(events.NewMessage(pattern=r'\.طرد (@\w+|.*)'))
        async def handler(event):
            await self.cmd_kick(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.حظر (@\w+|.*)'))
        async def handler(event):
            await self.cmd_ban(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.كتم (@\w+|.*)'))
        async def handler(event):
            await self.cmd_mute(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.ترحيب'))
        async def handler(event):
            await self.cmd_welcome(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.مسح (\d+)'))
        async def handler(event):
            await self.cmd_purge(event)
        
        # أوامر المطور
        @self.client.on(events.NewMessage(pattern=r'\.اعادة تشغيل'))
        async def handler(event):
            await self.cmd_restart(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.تحديث'))
        async def handler(event):
            await self.cmd_update(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.بلقنات'))
        async def handler(event):
            await self.cmd_plugins(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.الاوامر'))
        async def handler(event):
            await self.cmd_help(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.السجل'))
        async def handler(event):
            await self.cmd_logs(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.التخزين'))
        async def handler(event):
            await self.cmd_storage(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.صورة البوت'))
        async def handler(event):
            await self.cmd_bot_image(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.صور المجموعات'))
        async def handler(event):
            await self.cmd_group_images(event)
            
        # أوامر الحماية
        @self.client.on(events.NewMessage(pattern=r'\.حماية (تشغيل|ايقاف|اعدادات)'))
        async def handler(event):
            await self.cmd_security(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.حظر'))
        async def handler(event):
            await self.cmd_manual_block(event)
            
        @self.client.on(events.NewMessage(pattern=r'\.الغاء حظر'))
        async def handler(event):
            await self.cmd_unblock(event)
        
        # معالجة الرسائل
        @self.client.on(events.NewMessage(incoming=True, func=lambda e: e.is_private))
        async def handler(event):
            await self.log_private_message(event)
            await self.handle_stranger_message(event)
        
        @self.client.on(events.NewMessage(incoming=True, func=lambda e: e.message.media))
        async def handler(event):
            await self.handle_self_destruct_media(event)
    
    # --------------------------------------------------
    # نظام الحماية المتقدم
    # --------------------------------------------------
    
    async def cmd_security(self, event):
        """معالجة أوامر الحماية"""
        command = event.pattern_match.group(1).lower()
        
        if command == 'تشغيل':
            await self.enable_protection(event)
        elif command == 'ايقاف':
            await self.disable_protection(event)
        elif command == 'اعدادات':
            await self.show_security_settings(event)
    
    async def enable_protection(self, event):
        """تشغيل نظام الحماية"""
        self.security_settings['auto_block_strangers'] = True
        await event.reply('✅ **تم تشغيل نظام الحماية المتقدم**\n\nسأقوم تلقائياً بحظر أي شخص غريب يرسل رسالة!')
    
    async def disable_protection(self, event):
        """إيقاف نظام الحماية"""
        self.security_settings['auto_block_strangers'] = False
        await event.reply('❌ **تم إيقاف نظام الحماية**\n\nلن يتم حظر أي شخص تلقائياً.')
    
    async def show_security_settings(self, event):
        """عرض إعدادات الحماية"""
        settings_text = "⚙️ **إعدادات الحماية الحالية:**\n\n"
        translations = {
            'auto_block_strangers': 'الحظر التلقائي للغرباء',
            'warn_before_block': 'التحذير قبل الحظر',
            'max_warnings': 'الحد الأقصى للتحذيرات',
            'block_after_timeout': 'الحظر بعد انتهاء الوقت',
            'timeout_minutes': 'دقائق الانتظار',
            'anti_spam_protection': 'الحماية من السبام',
            'max_messages_per_minute': 'الحد الأقصى للرسائل'
        }
        
        for key, value in self.security_settings.items():
            arabic_key = translations.get(key, key)
            status = "✅ مفعل" if value else "❌ معطل"
            if isinstance(value, int):
                status = f"📊 {value}"
            settings_text += f"• {arabic_key}: {status}\n"
        
        await event.reply(settings_text)
    
    async def cmd_manual_block(self, event):
        """حظر يدوي"""
        if event.reply_to_msg_id:
            reply_msg = await event.get_reply_message()
            user_id = reply_msg.sender_id
            await self.block_user(user_id)
            await event.reply(f'✅ **تم حظر المستخدم بنجاح**\n\nتم حظر الشخص الذي رديت عليه.')
        else:
            await event.reply('❌ **يرجى الرد على رسالة المستخدم الذي تريد حظره**')
    
    async def cmd_unblock(self, event):
        """إلغاء حظر"""
        if event.reply_to_msg_id:
            reply_msg = await event.get_reply_message()
            user_id = reply_msg.sender_id
            try:
                await self.client(BlockRequest(id=user_id, unblock=True))
                await event.reply('✅ **تم إلغاء حظر المستخدم**')
            except Exception as e:
                await event.reply(f'❌ **خطأ في إلغاء الحظر:** {e}')
        else:
            await event.reply('❌ **يرجى الرد على رسالة المستخدم**')
    
    async def block_user(self, user_id):
        """حظر مستخدم"""
        try:
            await self.client(BlockRequest(user_id))
            if user_id in self.user_warnings:
                del self.user_warnings[user_id]
            return True
        except Exception as e:
            logger.error(f"خطأ في حظر المستخدم {user_id}: {e}")
            return False
    
    async def handle_stranger_message(self, event):
        """معالجة رسائل الغرباء"""
        if not self.security_settings['auto_block_strangers']:
            return
        
        user = await event.get_sender()
        chat = await event.get_chat()
        
        if not isinstance(chat, User):
            return
        
        is_contact = await self.is_user_contact(user.id)
        
        if not is_contact:
            await self.process_stranger(user, event)
    
    async def process_stranger(self, user, event):
        """معالجة المستخدم الغريب"""
        user_id = user.id
        
        if self.security_settings['warn_before_block']:
            warnings = self.user_warnings.get(user_id, 0)
            
            if warnings < self.security_settings['max_warnings']:
                warning_msg = f"""
                ⚠️ **تحذير أمني**
                
                أنت تتحدث مع {user.first_name} وهو ليس في قائمة معارفك!
                
                ❗ **التحذير:** {warnings + 1}/{self.security_settings['max_warnings']}
                
                إذا لم تقبل محادثته خلال 5 دقائق، سيتم حظره تلقائياً.
                """
                
                await event.reply(warning_msg)
                self.user_warnings[user_id] = warnings + 1
                
                if self.security_settings['block_after_timeout']:
                    await asyncio.sleep(self.security_settings['timeout_minutes'] * 60)
                    if user_id in self.user_warnings:
                        await self.block_user(user_id)
                        await event.reply(f'🚫 **تم حظر {user.first_name} تلقائياً**\n\nلم يتم قبول محادثته خلال الوقت المحدد.')
            else:
                await self.block_user(user_id)
                await event.reply(f'🚫 **تم حظر {user.first_name}**\n\nتجاوز الحد الأقصى للتحذيرات.')
        else:
            await self.block_user(user_id)
            await event.reply(f'🚫 **تم حظر {user.first_name} تلقائياً**\n\nتم اكتشاف شخص غريب في محادثتك.')
    
    async def is_user_contact(self, user_id):
        """التحقق إذا كان المستخدم من المعارف"""
        try:
            contacts = await self.client.get_contacts()
            return any(contact.id == user_id for contact in contacts)
        except Exception as e:
            logger.error(f"خطأ في التحقق من المعارف: {e}")
            return False

    # --------------------------------------------------
    # نظام الصور والتصميم
    # --------------------------------------------------
    
    def create_sython_logo(self, user_id, username=None):
        """إنشاء صورة شخصية لسايثون"""
        width, height = 1000, 1000
        image = Image.new('RGB', (width, height), '#1a1a2e')
        draw = ImageDraw.Draw(image)
        
        for i in range(50):
            x = random.randint(0, width)
            y = random.randint(0, height)
            r = random.randint(2, 8)
            draw.ellipse([x-r, y-r, x+r, y+r], fill='#16213e')
        
        draw.ellipse([300, 200, 700, 600], outline='#00ff88', width=15)
        draw.ellipse([350, 250, 650, 550], outline='#00ccff', width=12)
        draw.ellipse([450, 350, 470, 370], fill='#ffffff')
        draw.ellipse([530, 350, 550, 370], fill='#ffffff')
        
        try:
            font_large = ImageFont.truetype("arial.ttf", 60)
            font_small = ImageFont.truetype("arial.ttf", 30)
        except:
            font_large = ImageFont.load_default()
            font_small = ImageFont.load_default()
        
        bot_name = "بوت سايثون"
        reshaped_text = arabic_reshaper.reshape(bot_name)
        bidi_text = get_display(reshaped_text)
        draw.text((500, 650), bidi_text, font=font_large, fill='#ffffff', anchor='mm')
        
        if username:
            user_text = f"@{username}"
            draw.text((500, 720), user_text, font=font_small, fill='#00ccff', anchor='mm')
        
        for i in range(10):
            x = random.randint(100, 900)
            y = random.randint(100, 900)
            draw.ellipse([x-3, y-3, x+3, y+3], fill='#00ff88')
        
        filename = f'data/profile_images/sython_{user_id}.png'
        image.save(filename, 'PNG')
        return filename
    
    def create_storage_group_image(self):
        """إنشاء صورة لقروب التخزين"""
        width, height = 1500, 1500
        image = Image.new('RGB', (width, height), '#0a0a1a')
        draw = ImageDraw.Draw(image)
        
        center_x, center_y = width // 2, height // 2
        for i in range(300, 0, -10):
            alpha = i / 300
            color = (0, int(200 * alpha), int(255 * alpha))
            draw.ellipse([center_x-i, center_y-i, center_x+i, center_y+i], 
                        outline=color, width=2)
        
        draw.ellipse([500, 500, 1000, 1000], outline='#00ffff', width=10)
        
        for i in range(8):
            angle = i * 45
            rad = np.deg2rad(angle)
            x1 = center_x + 200 * np.cos(rad)
            y1 = center_y + 200 * np.sin(rad)
            x2 = center_x + 450 * np.cos(rad)
            y2 = center_y + 450 * np.sin(rad)
            draw.line([(x1, y1), (x2, y2)], fill='#00ff88', width=3)
        
        for i in range(50):
            angle = random.uniform(0, 360)
            radius = random.uniform(100, 450)
            x = center_x + radius * np.cos(np.deg2rad(angle))
            y = center_y + radius * np.sin(np.deg2rad(angle))
            size = random.randint(5, 15)
            color = random.choice(['#ff0066', '#00ff88', '#0088ff', '#ffcc00'])
            draw.ellipse([x-size, y-size, x+size, y+size], fill=color)
        
        try:
            font_arabic = ImageFont.truetype("arial.ttf", 80)
            font_english = ImageFont.truetype("arial.ttf", 60)
        except:
            font_arabic = ImageFont.load_default()
            font_english = ImageFont.load_default()
        
        arabic_text = "قروب التخزين"
        reshaped_arabic = arabic_reshaper.reshape(arabic_text)
        bidi_arabic = get_display(reshaped_arabic)
        draw.text((center_x, 1200), bidi_arabic, font=font_arabic, 
                  fill='#ffffff', anchor='mm', stroke_width=2, stroke_fill='#000000')
        
        copyright_text = "SYTHON STORAGE SYSTEM v2.0"
        draw.text((center_x, 1300), copyright_text, font=font_english, 
                  fill='#00ffff', anchor='mm', stroke_width=1, stroke_fill='#0088ff')
        
        filename = 'data/group_images/storage_group.png'
        image.save(filename, 'PNG')
        return filename
    
    def create_log_group_image(self):
        """إنشاء صورة لقروب السجل"""
        width, height = 1500, 1500
        image = Image.new('RGB', (width, height), '#1a0a1a')
        draw = ImageDraw.Draw(image)
        
        for i in range(1000):
            x = random.randint(0, width)
            y = random.randint(0, height)
            if random.random() < 0.7:
                length = random.randint(50, 200)
                draw.line([(x, y), (x+length, y)], 
                         fill=random.choice(['#ff0066', '#00ff88', '#0088ff']), 
                         width=2)
            else:
                size = random.randint(3, 8)
                draw.ellipse([x-size, y-size, x+size, y+size], 
                            fill=random.choice(['#ff0066', '#00ff88', '#0088ff']))
        
        draw.rectangle([400, 400, 1100, 1100], outline='#ff0066', width=8)
        
        for i in range(20):
            x1 = 450 + random.randint(0, 600)
            y1 = 450 + random.randint(0, 600)
            x2 = 450 + random.randint(0, 600)
            y2 = 450 + random.randint(0, 600)
            draw.line([(x1, y1), (x2, y2)], 
                     fill=random.choice(['#00ff88', '#0088ff', '#ffcc00']), 
                     width=3)
        
        for i in range(10):
            x = 500 + i * 60
            height_col = random.randint(100, 400)
            color = random.choice(['#00ff88', '#0088ff', '#ff0066'])
            draw.rectangle([x, 1000-height_col, x+40, 1000], 
                          fill=color, outline='#ffffff', width=2)
        
        try:
            font_arabic = ImageFont.truetype("arial.ttf", 80)
            font_english = ImageFont.truetype("arial.ttf", 60)
        except:
            font_arabic = ImageFont.load_default()
            font_english = ImageFont.load_default()
        
        arabic_text = "قروب السجل"
        reshaped_ara
