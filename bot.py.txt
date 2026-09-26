================================================================================
FAYL: config.py
================================================================================
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import List
import os


class Settings(BaseSettings):
    BOT_TOKEN: str
    ADMIN_IDS: List[int] = []
    CHANNEL_ID: int = 0  # Kanal ID (masalan: -1001234567890)
    CHANNEL_USERNAME: str = ""  # @kanal_username
    DB_PATH: str = "data/bot.db"
    
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore"
    )


settings = Settings()

================================================================================
FAYL: database.py
================================================================================
import aiosqlite
from typing import Optional, List, Dict, Any
from config import settings
import os


async def init_db():
    os.makedirs(os.path.dirname(settings.DB_PATH) if os.path.dirname(settings.DB_PATH) else ".", exist_ok=True)
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute("PRAGMA foreign_keys = ON")
        
        # Users
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                full_name TEXT,
                is_vip INTEGER DEFAULT 0,
                is_banned INTEGER DEFAULT 0,
                joined_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Admins
        await db.execute("""
            CREATE TABLE IF NOT EXISTS admins (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                added_by INTEGER,
                added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Sections (categories)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS sections (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL UNIQUE,
                emoji TEXT DEFAULT '📁',
                description TEXT,
                position INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
        
        # Animes
        await db.execute("""
            CREATE TABLE IF NOT EXISTS animes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                section_id INTEGER NOT NULL,
                title TEXT NOT NULL,
                description TEXT,
                cover_file_id TEXT,
                download_links TEXT,  -- JSON yoki oddiy matn (bir nechta link)
                is_vip INTEGER DEFAULT 0,
                episodes_count INTEGER DEFAULT 1,
                status TEXT DEFAULT 'ongoing',  -- ongoing, completed
                views INTEGER DEFAULT 0,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (section_id) REFERENCES sections(id) ON DELETE CASCADE
            )
        """)
        
        # Episodes (agar kerak bo'lsa, keyinroq kengaytirish uchun)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS episodes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                anime_id INTEGER NOT NULL,
                episode_number INTEGER NOT NULL,
                title TEXT,
                file_id TEXT,
                file_type TEXT DEFAULT 'video',  -- video, document
                size TEXT,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (anime_id) REFERENCES animes(id) ON DELETE CASCADE
            )
        """)
        
        await db.commit()
        
        # Default adminlarni qo'shish
        for admin_id in settings.ADMIN_IDS:
            await db.execute(
                "INSERT OR IGNORE INTO admins (user_id, username) VALUES (?, ?)",
                (admin_id, "config_admin")
            )
        await db.commit()


# ========== USERS ==========
async def add_user(user_id: int, username: str = None, full_name: str = None):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute(
            """INSERT INTO users (user_id, username, full_name) 
               VALUES (?, ?, ?) 
               ON CONFLICT(user_id) DO UPDATE SET 
               username=excluded.username, full_name=excluded.full_name""",
            (user_id, username, full_name)
        )
        await db.commit()


async def get_user(user_id: int) -> Optional[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT * FROM users WHERE user_id = ?", (user_id,)) as cursor:
            row = await cursor.fetchone()
            return dict(row) if row else None


async def is_vip(user_id: int) -> bool:
    user = await get_user(user_id)
    return bool(user and user.get("is_vip"))


async def set_vip(user_id: int, status: bool = True):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute(
            "UPDATE users SET is_vip = ? WHERE user_id = ?",
            (1 if status else 0, user_id)
        )
        # Agar user yo'q bo'lsa qo'shamiz
        await db.execute(
            "INSERT OR IGNORE INTO users (user_id, is_vip) VALUES (?, ?)",
            (user_id, 1 if status else 0)
        )
        await db.commit()


async def get_all_vips() -> List[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT * FROM users WHERE is_vip = 1") as cursor:
            rows = await cursor.fetchall()
            return [dict(r) for r in rows]


async def get_users_count() -> int:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        async with db.execute("SELECT COUNT(*) FROM users") as cursor:
            return (await cursor.fetchone())[0]


# ========== ADMINS ==========
async def is_admin(user_id: int) -> bool:
    if user_id in settings.ADMIN_IDS:
        return True
    async with aiosqlite.connect(settings.DB_PATH) as db:
        async with db.execute("SELECT 1 FROM admins WHERE user_id = ?", (user_id,)) as cursor:
            return await cursor.fetchone() is not None


async def add_admin(user_id: int, username: str = None, added_by: int = None):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute(
            "INSERT OR REPLACE INTO admins (user_id, username, added_by) VALUES (?, ?, ?)",
            (user_id, username, added_by)
        )
        await db.commit()


async def remove_admin(user_id: int):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute("DELETE FROM admins WHERE user_id = ?", (user_id,))
        await db.commit()


async def get_all_admins() -> List[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT * FROM admins") as cursor:
            rows = await cursor.fetchall()
            return [dict(r) for r in rows]


# ========== SECTIONS ==========
async def add_section(name: str, emoji: str = "📁", description: str = None) -> int:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        cursor = await db.execute(
            "INSERT INTO sections (name, emoji, description) VALUES (?, ?, ?)",
            (name, emoji, description)
        )
        await db.commit()
        return cursor.lastrowid


async def get_sections() -> List[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT * FROM sections ORDER BY position, id") as cursor:
            rows = await cursor.fetchall()
            return [dict(r) for r in rows]


async def get_section(section_id: int) -> Optional[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT * FROM sections WHERE id = ?", (section_id,)) as cursor:
            row = await cursor.fetchone()
            return dict(row) if row else None


async def update_section(section_id: int, name: str = None, emoji: str = None, description: str = None):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        updates = []
        params = []
        if name is not None:
            updates.append("name = ?")
            params.append(name)
        if emoji is not None:
            updates.append("emoji = ?")
            params.append(emoji)
        if description is not None:
            updates.append("description = ?")
            params.append(description)
        if updates:
            params.append(section_id)
            await db.execute(f"UPDATE sections SET {', '.join(updates)} WHERE id = ?", params)
            await db.commit()


async def delete_section(section_id: int):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute("DELETE FROM sections WHERE id = ?", (section_id,))
        await db.commit()


# ========== ANIMES ==========
async def add_anime(
    section_id: int,
    title: str,
    description: str = None,
    cover_file_id: str = None,
    download_links: str = None,
    is_vip: bool = False,
    episodes_count: int = 1,
    status: str = "ongoing"
) -> int:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        cursor = await db.execute(
            """INSERT INTO animes 
               (section_id, title, description, cover_file_id, download_links, is_vip, episodes_count, status)
               VALUES (?, ?, ?, ?, ?, ?, ?, ?)""",
            (section_id, title, description, cover_file_id, download_links, 1 if is_vip else 0, episodes_count, status)
        )
        await db.commit()
        return cursor.lastrowid


async def get_animes_by_section(section_id: int) -> List[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute(
            "SELECT * FROM animes WHERE section_id = ? ORDER BY id DESC", (section_id,)
        ) as cursor:
            rows = await cursor.fetchall()
            return [dict(r) for r in rows]


async def get_anime(anime_id: int) -> Optional[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute("SELECT * FROM animes WHERE id = ?", (anime_id,)) as cursor:
            row = await cursor.fetchone()
            return dict(row) if row else None


async def update_anime(anime_id: int, **kwargs):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        allowed = ["title", "description", "cover_file_id", "download_links", "is_vip", "episodes_count", "status", "section_id"]
        updates = []
        params = []
        for k, v in kwargs.items():
            if k in allowed and v is not None:
                updates.append(f"{k} = ?")
                params.append(v)
        if updates:
            updates.append("updated_at = CURRENT_TIMESTAMP")
            params.append(anime_id)
            await db.execute(f"UPDATE animes SET {', '.join(updates)} WHERE id = ?", params)
            await db.commit()


async def delete_anime(anime_id: int):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute("DELETE FROM animes WHERE id = ?", (anime_id,))
        await db.commit()


async def increment_views(anime_id: int):
    async with aiosqlite.connect(settings.DB_PATH) as db:
        await db.execute("UPDATE animes SET views = views + 1 WHERE id = ?", (anime_id,))
        await db.commit()


async def get_animes_count() -> int:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        async with db.execute("SELECT COUNT(*) FROM animes") as cursor:
            return (await cursor.fetchone())[0]


async def get_sections_count() -> int:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        async with db.execute("SELECT COUNT(*) FROM sections") as cursor:
            return (await cursor.fetchone())[0]


async def search_animes(query: str) -> List[Dict]:
    async with aiosqlite.connect(settings.DB_PATH) as db:
        db.row_factory = aiosqlite.Row
        async with db.execute(
            "SELECT * FROM animes WHERE title LIKE ? ORDER BY views DESC LIMIT 20",
            (f"%{query}%",)
        ) as cursor:
            rows = await cursor.fetchall()
            return [dict(r) for r in rows]

================================================================================
FAYL: states.py
================================================================================
from aiogram.fsm.state import State, StatesGroup


class AdminStates(StatesGroup):
    # Section
    add_section_name = State()
    add_section_emoji = State()
    edit_section_name = State()
    edit_section_emoji = State()
    
    # Anime
    add_anime_section = State()
    add_anime_title = State()
    add_anime_desc = State()
    add_anime_cover = State()
    add_anime_links = State()
    add_anime_vip = State()
    add_anime_episodes = State()
    
    edit_anime_field = State()
    edit_anime_value = State()
    
    # VIP
    add_vip_id = State()
    
    # Admin
    add_admin_id = State()
    
    # Broadcast
    broadcast_message = State()
    
    # Post
    post_confirm = State()


class UserStates(StatesGroup):
    search_query = State()

================================================================================
FAYL: utils/subscription.py
================================================================================
from aiogram import Bot
from aiogram.enums import ChatMemberStatus
from config import settings


async def check_subscription(bot: Bot, user_id: int) -> bool:
    """Foydalanuvchi kanalga obuna bo'lganligini tekshiradi"""
    if not settings.CHANNEL_ID:
        return True  # Agar kanal sozlanmagan bo'lsa, o'tkazib yuboramiz
    
    try:
        member = await bot.get_chat_member(chat_id=settings.CHANNEL_ID, user_id=user_id)
        return member.status in [
            ChatMemberStatus.MEMBER,
            ChatMemberStatus.ADMINISTRATOR,
            ChatMemberStatus.CREATOR,
            ChatMemberStatus.RESTRICTED  # ba'zi hollarda restricted ham member hisoblanadi
        ]
    except Exception:
        # Bot kanalda admin emas yoki xato
        return False


def get_subscribe_keyboard():
    """Majburiy obuna klaviaturasi"""
    from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
    from aiogram.utils.keyboard import InlineKeyboardBuilder
    
    builder = InlineKeyboardBuilder()
    
    channel_link = f"https://t.me/{settings.CHANNEL_USERNAME.lstrip('@')}" if settings.CHANNEL_USERNAME else None
    
    if channel_link:
        builder.row(
            InlineKeyboardButton(text="📢 Kanalga obuna bo'lish", url=channel_link)
        )
    
    builder.row(
        InlineKeyboardButton(text="✅ Obunani tekshirish", callback_data="check_sub")
    )
    
    return builder.as_markup()

================================================================================
FAYL: keyboards/user_kb.py
================================================================================
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton, ReplyKeyboardMarkup, KeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder, ReplyKeyboardBuilder
from typing import List, Dict


def main_menu_kb(is_admin: bool = False) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="🎬 Anime bo'limlari", callback_data="sections"),
        InlineKeyboardButton(text="🔍 Qidiruv", callback_data="search")
    )
    builder.row(
        InlineKeyboardButton(text="⭐ VIP status", callback_data="vip_info"),
        InlineKeyboardButton(text="📢 Kanalimiz", url="https://t.me/" + "your_channel")  # config dan olinadi
    )
    builder.row(
        InlineKeyboardButton(text="ℹ️ Yordam", callback_data="help")
    )
    if is_admin:
        builder.row(
            InlineKeyboardButton(text="🛠 Admin panel", callback_data="admin_panel")
        )
    return builder.as_markup()


def sections_kb(sections: List[Dict]) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    for s in sections:
        emoji = s.get("emoji") or "📁"
        builder.row(
            InlineKeyboardButton(
                text=f"{emoji} {s['name']}",
                callback_data=f"section_{s['id']}"
            )
        )
    builder.row(InlineKeyboardButton(text="◀️ Orqaga", callback_data="back_main"))
    return builder.as_markup()


def animes_kb(animes: List[Dict], section_id: int, page: int = 0, per_page: int = 8) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    start = page * per_page
    end = start + per_page
    page_animes = animes[start:end]
    
    for a in page_animes:
        vip_mark = "👑 " if a.get("is_vip") else ""
        status = "🟢" if a.get("status") == "completed" else "🔴"
        builder.row(
            InlineKeyboardButton(
                text=f"{vip_mark}{status} {a['title']}",
                callback_data=f"anime_{a['id']}"
            )
        )
    
    # Pagination
    nav = []
    if page > 0:
        nav.append(InlineKeyboardButton(text="⬅️", callback_data=f"animes_page_{section_id}_{page-1}"))
    if end < len(animes):
        nav.append(InlineKeyboardButton(text="➡️", callback_data=f"animes_page_{section_id}_{page+1}"))
    if nav:
        builder.row(*nav)
    
    builder.row(InlineKeyboardButton(text="◀️ Bo'limlarga", callback_data="sections"))
    return builder.as_markup()


def anime_detail_kb(anime_id: int, has_links: bool = True, is_vip_content: bool = False, user_is_vip: bool = False) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    
    if is_vip_content and not user_is_vip:
        builder.row(
            InlineKeyboardButton(text="👑 VIP bo'lish", callback_data="vip_info")
        )
    else:
        if has_links:
            builder.row(
                InlineKeyboardButton(text="📥 Yuklab olish", callback_data=f"download_{anime_id}")
            )
    
    builder.row(
        InlineKeyboardButton(text="◀️ Orqaga", callback_data="back_to_section")
    )
    return builder.as_markup()


def back_main_kb() -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(InlineKeyboardButton(text="◀️ Asosiy menyu", callback_data="back_main"))
    return builder.as_markup()


def cancel_kb() -> ReplyKeyboardMarkup:
    builder = ReplyKeyboardBuilder()
    builder.add(KeyboardButton(text="❌ Bekor qilish"))
    return builder.as_markup(resize_keyboard=True)

================================================================================
FAYL: keyboards/admin_kb.py
================================================================================
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.utils.keyboard import InlineKeyboardBuilder
from typing import List, Dict


def admin_panel_kb() -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="📊 Statistika", callback_data="admin_stats"),
        InlineKeyboardButton(text="📁 Bo'limlar", callback_data="admin_sections")
    )
    builder.row(
        InlineKeyboardButton(text="🎬 Animelar", callback_data="admin_animes"),
        InlineKeyboardButton(text="👑 VIP boshqaruv", callback_data="admin_vips")
    )
    builder.row(
        InlineKeyboardButton(text="👮 Adminlar", callback_data="admin_admins"),
        InlineKeyboardButton(text="📢 Xabar yuborish", callback_data="admin_broadcast")
    )
    builder.row(
        InlineKeyboardButton(text="📤 Kanalga post", callback_data="admin_post_channel")
    )
    builder.row(
        InlineKeyboardButton(text="◀️ Asosiy menyu", callback_data="back_main")
    )
    return builder.as_markup()


def admin_sections_kb(sections: List[Dict]) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    for s in sections:
        emoji = s.get("emoji") or "📁"
        builder.row(
            InlineKeyboardButton(
                text=f"{emoji} {s['name']}",
                callback_data=f"admin_section_{s['id']}"
            )
        )
    builder.row(
        InlineKeyboardButton(text="➕ Yangi bo'lim", callback_data="admin_add_section")
    )
    builder.row(
        InlineKeyboardButton(text="◀️ Admin panel", callback_data="admin_panel")
    )
    return builder.as_markup()


def admin_section_actions_kb(section_id: int) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="✏️ Tahrirlash", callback_data=f"admin_edit_section_{section_id}"),
        InlineKeyboardButton(text="🗑 O'chirish", callback_data=f"admin_del_section_{section_id}")
    )
    builder.row(
        InlineKeyboardButton(text="🎬 Animelari", callback_data=f"admin_section_animes_{section_id}")
    )
    builder.row(
        InlineKeyboardButton(text="◀️ Bo'limlar", callback_data="admin_sections")
    )
    return builder.as_markup()


def admin_animes_list_kb(animes: List[Dict], section_id: int = None) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    for a in animes[:15]:  # limit
        vip = "👑" if a.get("is_vip") else ""
        builder.row(
            InlineKeyboardButton(
                text=f"{vip} {a['title'][:40]}",
                callback_data=f"admin_anime_{a['id']}"
            )
        )
    builder.row(
        InlineKeyboardButton(text="➕ Yangi anime", callback_data=f"admin_add_anime_{section_id or 0}")
    )
    builder.row(
        InlineKeyboardButton(text="◀️ Orqaga", callback_data="admin_animes" if not section_id else f"admin_section_{section_id}")
    )
    return builder.as_markup()


def admin_anime_actions_kb(anime_id: int) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="✏️ Tahrirlash", callback_data=f"admin_edit_anime_{anime_id}"),
        InlineKeyboardButton(text="🗑 O'chirish", callback_data=f"admin_del_anime_{anime_id}")
    )
    builder.row(
        InlineKeyboardButton(text="📤 Kanalga joylash", callback_data=f"admin_post_anime_{anime_id}")
    )
    builder.row(
        InlineKeyboardButton(text="◀️ Orqaga", callback_data="admin_animes")
    )
    return builder.as_markup()


def confirm_kb(action: str, item_id: int) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="✅ Ha", callback_data=f"confirm_{action}_{item_id}"),
        InlineKeyboardButton(text="❌ Yo'q", callback_data=f"cancel_{action}")
    )
    return builder.as_markup()


def admin_vips_kb(vips: List[Dict]) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    for v in vips[:20]:
        name = v.get("full_name") or v.get("username") or str(v["user_id"])
        builder.row(
            InlineKeyboardButton(
                text=f"👑 {name}",
                callback_data=f"admin_vip_{v['user_id']}"
            )
        )
    builder.row(
        InlineKeyboardButton(text="➕ VIP qo'shish", callback_data="admin_add_vip")
    )
    builder.row(
        InlineKeyboardButton(text="◀️ Admin panel", callback_data="admin_panel")
    )
    return builder.as_markup()


def admin_admins_kb(admins: List[Dict]) -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    for a in admins:
        name = a.get("username") or str(a["user_id"])
        builder.row(
            InlineKeyboardButton(
                text=f"👮 {name}",
                callback_data=f"admin_admin_{a['user_id']}"
            )
        )
    builder.row(
        InlineKeyboardButton(text="➕ Admin qo'shish", callback_data="admin_add_admin")
    )
    builder.row(
        InlineKeyboardButton(text="◀️ Admin panel", callback_data="admin_panel")
    )
    return builder.as_markup()


def cancel_inline_kb() -> InlineKeyboardMarkup:
    builder = InlineKeyboardBuilder()
    builder.row(InlineKeyboardButton(text="❌ Bekor qilish", callback_data="admin_cancel"))
    return builder.as_markup()

================================================================================
FAYL: handlers/user.py
================================================================================
from aiogram import Router, F, Bot
from aiogram.types import Message, CallbackQuery
from aiogram.filters import CommandStart, Command
from aiogram.fsm.context import FSMContext
from aiogram.enums import ParseMode

import database as db
from keyboards.user_kb import (
    main_menu_kb, sections_kb, animes_kb, anime_detail_kb, back_main_kb
)
from states import UserStates
from config import settings
from utils.subscription import check_subscription, get_subscribe_keyboard

router = Router()


async def get_main_keyboard(user_id: int):
    """Asosiy menyu klaviaturasini yaratadi"""
    is_adm = await db.is_admin(user_id)
    from aiogram.types import InlineKeyboardButton
    from aiogram.utils.keyboard import InlineKeyboardBuilder
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="🎬 Anime bo'limlari", callback_data="sections"),
        InlineKeyboardButton(text="🔍 Qidiruv", callback_data="search")
    )
    builder.row(
        InlineKeyboardButton(text="⭐ VIP status", callback_data="vip_info"),
    )
    if settings.CHANNEL_USERNAME:
        builder.row(
            InlineKeyboardButton(text="📢 Kanalimiz", url=f"https://t.me/{settings.CHANNEL_USERNAME.lstrip('@')}")
        )
    builder.row(InlineKeyboardButton(text="ℹ️ Yordam", callback_data="help"))
    if is_adm:
        builder.row(InlineKeyboardButton(text="🛠 Admin panel", callback_data="admin_panel"))
    return builder.as_markup()


@router.message(CommandStart())
async def cmd_start(message: Message, state: FSMContext, bot: Bot):
    await state.clear()
    await db.add_user(
        message.from_user.id,
        message.from_user.username,
        message.from_user.full_name
    )
    
    # ===== MAJBURIY OBUNA TEKSHIRUVI =====
    is_subscribed = await check_subscription(bot, message.from_user.id)
    if not is_subscribed:
        text = (
            f"<b>🔒 Majburiy obuna</b>\n\n"
            f"Botdan foydalanish uchun avval kanalimizga obuna bo'lishingiz shart!\n\n"
            f"👇 Pastdagi tugma orqali obuna bo'ling, keyin «✅ Obunani tekshirish» ni bosing."
        )
        await message.answer(text, reply_markup=get_subscribe_keyboard(), parse_mode=ParseMode.HTML)
        return
    
    # Obuna bo'lgan — asosiy menyu
    user = await db.get_user(message.from_user.id)
    vip_status = "👑 VIP a'zo" if user and user.get("is_vip") else "Oddiy foydalanuvchi"
    
    text = (
        f"<b>✨ Assalomu alaykum, {message.from_user.first_name}!</b>\n\n"
        f"🎬 <b>Anime Kanal Botiga</b> xush kelibsiz!\n\n"
        f"📊 Sizning statusingiz: <b>{vip_status}</b>\n\n"
        f"Bu yerdan eng so'nggi animelarni ko'rishingiz, yuklab olishingiz mumkin.\n"
        f"Bo'limlarni tanlang yoki qidiruvdan foydalaning 👇"
    )
    
    kb = await get_main_keyboard(message.from_user.id)
    await message.answer(text, reply_markup=kb, parse_mode=ParseMode.HTML)


@router.callback_query(F.data == "check_sub")
async def check_subscription_callback(callback: CallbackQuery, bot: Bot):
    """Obunani qayta tekshirish"""
    is_subscribed = await check_subscription(bot, callback.from_user.id)
    
    if is_subscribed:
        user = await db.get_user(callback.from_user.id)
        vip_status = "👑 VIP a'zo" if user and user.get("is_vip") else "Oddiy foydalanuvchi"
        
        text = (
            f"<b>✅ Rahmat! Obuna tasdiqlandi.</b>\n\n"
            f"✨ Assalomu alaykum, {callback.from_user.first_name}!\n\n"
            f"🎬 <b>Anime Kanal Botiga</b> xush kelibsiz!\n\n"
            f"📊 Sizning statusingiz: <b>{vip_status}</b>\n\n"
            f"Bo'limlarni tanlang yoki qidiruvdan foydalaning 👇"
        )
        kb = await get_main_keyboard(callback.from_user.id)
        await callback.message.edit_text(text, reply_markup=kb, parse_mode=ParseMode.HTML)
        await callback.answer("✅ Obuna tasdiqlandi!", show_alert=True)
    else:
        await callback.answer("❌ Siz hali kanalga obuna bo'lmagansiz!", show_alert=True)


@router.callback_query(F.data == "back_main")
async def back_to_main(callback: CallbackQuery, state: FSMContext, bot: Bot):
    await state.clear()
    
    # Majburiy obuna tekshiruvi
    if not await check_subscription(bot, callback.from_user.id):
        text = (
            f"<b>🔒 Majburiy obuna</b>\n\n"
            f"Botdan foydalanish uchun avval kanalimizga obuna bo'lishingiz shart!"
        )
        await callback.message.edit_text(text, reply_markup=get_subscribe_keyboard(), parse_mode=ParseMode.HTML)
        await callback.answer()
        return
    
    user = await db.get_user(callback.from_user.id)
    vip_status = "👑 VIP a'zo" if user and user.get("is_vip") else "Oddiy foydalanuvchi"
    
    text = (
        f"<b>🏠 Asosiy menyu</b>\n\n"
        f"📊 Status: <b>{vip_status}</b>\n\n"
        f"Quyidagi bo'limlardan birini tanlang:"
    )
    
    kb = await get_main_keyboard(callback.from_user.id)
    await callback.message.edit_text(text, reply_markup=kb, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "sections")
async def show_sections(callback: CallbackQuery, bot: Bot):
    # Majburiy obuna
    if not await check_subscription(bot, callback.from_user.id):
        text = (
            f"<b>🔒 Majburiy obuna</b>\n\n"
            f"Botdan foydalanish uchun avval kanalimizga obuna bo'lishingiz shart!"
        )
        await callback.message.edit_text(text, reply_markup=get_subscribe_keyboard(), parse_mode=ParseMode.HTML)
        await callback.answer()
        return
    
    sections = await db.get_sections()
    if not sections:
        await callback.message.edit_text(
            "📭 Hozircha bo'limlar yo'q.\nAdmin tez orada qo'shadi!",
            reply_markup=back_main_kb()
        )
    else:
        text = "<b>📁 Anime bo'limlari</b>\n\nQuyidagi bo'limlardan birini tanlang:"
        await callback.message.edit_text(text, reply_markup=sections_kb(sections), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("section_"))
async def show_section_animes(callback: CallbackQuery, state: FSMContext):
    section_id = int(callback.data.split("_")[1])
    section = await db.get_section(section_id)
    if not section:
        await callback.answer("Bo'lim topilmadi!", show_alert=True)
        return
    
    animes = await db.get_animes_by_section(section_id)
    await state.update_data(current_section=section_id)
    
    if not animes:
        text = f"<b>{section.get('emoji', '📁')} {section['name']}</b>\n\n📭 Bu bo'limda hozircha anime yo'q."
        await callback.message.edit_text(text, reply_markup=sections_kb(await db.get_sections()), parse_mode=ParseMode.HTML)
    else:
        text = (
            f"<b>{section.get('emoji', '📁')} {section['name']}</b>\n"
            f"📊 Jami: <b>{len(animes)}</b> ta anime\n\n"
            f"Tanlang:"
        )
        await callback.message.edit_text(text, reply_markup=animes_kb(animes, section_id), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("animes_page_"))
async def animes_pagination(callback: CallbackQuery):
    parts = callback.data.split("_")
    section_id = int(parts[2])
    page = int(parts[3])
    section = await db.get_section(section_id)
    animes = await db.get_animes_by_section(section_id)
    
    text = (
        f"<b>{section.get('emoji', '📁')} {section['name']}</b>\n"
        f"📊 Jami: <b>{len(animes)}</b> ta anime\n\n"
        f"Sahifa: {page + 1}"
    )
    await callback.message.edit_text(text, reply_markup=animes_kb(animes, section_id, page), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("anime_"))
async def show_anime_detail(callback: CallbackQuery, bot: Bot):
    # Majburiy obuna
    if not await check_subscription(bot, callback.from_user.id):
        text = (
            f"<b>🔒 Majburiy obuna</b>\n\n"
            f"Botdan foydalanish uchun avval kanalimizga obuna bo'lishingiz shart!"
        )
        await callback.message.edit_text(text, reply_markup=get_subscribe_keyboard(), parse_mode=ParseMode.HTML)
        await callback.answer()
        return
    
    anime_id = int(callback.data.split("_")[1])
    anime = await db.get_anime(anime_id)
    if not anime:
        await callback.answer("Anime topilmadi!", show_alert=True)
        return
    
    await db.increment_views(anime_id)
    user_is_vip = await db.is_vip(callback.from_user.id)
    is_vip_content = bool(anime.get("is_vip"))
    
    status_emoji = "✅ Tugagan" if anime.get("status") == "completed" else "🔄 Davom etmoqda"
    vip_mark = "\n👑 <b>VIP kontent</b>" if is_vip_content else ""
    
    text = (
        f"<b>🎬 {anime['title']}</b>{vip_mark}\n\n"
        f"📝 <b>Tavsif:</b>\n{anime.get('description') or 'Tavsif yoʻq'}\n\n"
        f"📊 Holat: {status_emoji}\n"
        f"🎞 Qismlar: <b>{anime.get('episodes_count', 1)}</b>\n"
        f"👁 Koʻrilgan: <b>{anime.get('views', 0) + 1}</b>\n"
    )
    
    has_links = bool(anime.get("download_links"))
    
    # Cover bo'lsa photo yuborish
    if anime.get("cover_file_id"):
        try:
            await callback.message.delete()
            await callback.message.answer_photo(
                photo=anime["cover_file_id"],
                caption=text,
                reply_markup=anime_detail_kb(anime_id, has_links, is_vip_content, user_is_vip),
                parse_mode=ParseMode.HTML
            )
            await callback.answer()
            return
        except Exception:
            pass
    
    await callback.message.edit_text(
        text,
        reply_markup=anime_detail_kb(anime_id, has_links, is_vip_content, user_is_vip),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("download_"))
async def download_anime(callback: CallbackQuery, bot: Bot):
    # Majburiy obuna
    if not await check_subscription(bot, callback.from_user.id):
        await callback.answer("🔒 Avval kanalga obuna bo'ling!", show_alert=True)
        return
    
    anime_id = int(callback.data.split("_")[1])
    anime = await db.get_anime(anime_id)
    if not anime:
        await callback.answer("Anime topilmadi!", show_alert=True)
        return
    
    if anime.get("is_vip") and not await db.is_vip(callback.from_user.id):
        await callback.answer("👑 Bu VIP kontent! Avval VIP bo'ling.", show_alert=True)
        return
    
    links = anime.get("download_links") or "Hozircha yuklab olish havolalari yo'q."
    text = (
        f"<b>📥 {anime['title']} — Yuklab olish</b>\n\n"
        f"{links}\n\n"
        f"<i>Havolalarni nusxalab oling.</i>"
    )
    await callback.message.answer(text, parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "vip_info")
async def vip_info(callback: CallbackQuery):
    user = await db.get_user(callback.from_user.id)
    is_vip = user and user.get("is_vip")
    
    if is_vip:
        text = (
            "<b>👑 Siz VIP a'zosiz!</b>\n\n"
            "Sizga maxsus kontentlar va imtiyozlar ochiq.\n"
            "Rahmat qo'llab-quvvatlaganingiz uchun! ❤️"
        )
    else:
        text = (
            "<b>⭐ VIP status nima?</b>\n\n"
            "VIP a'zolar:\n"
            "• Maxsus (VIP) animelarga kirish\n"
            "• Erta chiqish\n"
            "• Reklamasiz foydalanish\n"
            "• Maxsus yordam\n\n"
            "<i>VIP bo'lish uchun admin bilan bog'laning yoki kanalimizdagi e'lonlarni kuzating.</i>"
        )
    
    await callback.message.edit_text(text, reply_markup=back_main_kb(), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "help")
async def help_cmd(callback: CallbackQuery):
    text = (
        "<b>ℹ️ Yordam</b>\n\n"
        "🎬 <b>Anime bo'limlari</b> — kategoriyalar bo'yicha ko'rish\n"
        "🔍 <b>Qidiruv</b> — anime nomi bo'yicha qidirish\n"
        "⭐ <b>VIP</b> — maxsus kontentlar\n"
        "📢 <b>Kanal</b> — asosiy anime kanalimiz\n\n"
        "Savollar bo'lsa adminlarga yozing."
    )
    await callback.message.edit_text(text, reply_markup=back_main_kb(), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "search")
async def start_search(callback: CallbackQuery, state: FSMContext, bot: Bot):
    # Majburiy obuna
    if not await check_subscription(bot, callback.from_user.id):
        text = (
            f"<b>🔒 Majburiy obuna</b>\n\n"
            f"Botdan foydalanish uchun avval kanalimizga obuna bo'lishingiz shart!"
        )
        await callback.message.edit_text(text, reply_markup=get_subscribe_keyboard(), parse_mode=ParseMode.HTML)
        await callback.answer()
        return
    
    await state.set_state(UserStates.search_query)
    await callback.message.edit_text(
        "🔍 <b>Qidiruv</b>\n\nAnime nomini yozing:",
        reply_markup=back_main_kb(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.message(UserStates.search_query)
async def process_search(message: Message, state: FSMContext):
    query = message.text.strip()
    if len(query) < 2:
        await message.answer("Kamida 2 ta belgi kiriting.")
        return
    
    results = await db.search_animes(query)
    await state.clear()
    
    if not results:
        await message.answer(
            f"🔍 «{query}» bo'yicha hech narsa topilmadi.",
            reply_markup=back_main_kb()
        )
        return
    
    text = f"<b>🔍 Qidiruv natijalari: «{query}»</b>\n\nTopildi: {len(results)} ta\n\n"
    from aiogram.utils.keyboard import InlineKeyboardBuilder
    from aiogram.types import InlineKeyboardButton
    builder = InlineKeyboardBuilder()
    for a in results[:10]:
        vip = "👑 " if a.get("is_vip") else ""
        builder.row(
            InlineKeyboardButton(text=f"{vip}{a['title']}", callback_data=f"anime_{a['id']}")
        )
    builder.row(InlineKeyboardButton(text="◀️ Asosiy menyu", callback_data="back_main"))
    
    await message.answer(text, reply_markup=builder.as_markup(), parse_mode=ParseMode.HTML)


@router.callback_query(F.data == "back_to_section")
async def back_to_section(callback: CallbackQuery, state: FSMContext):
    data = await state.get_data()
    section_id = data.get("current_section")
    if section_id:
        # Reuse the section handler logic
        section = await db.get_section(section_id)
        animes = await db.get_animes_by_section(section_id)
        text = (
            f"<b>{section.get('emoji', '📁')} {section['name']}</b>\n"
            f"📊 Jami: <b>{len(animes)}</b> ta anime\n\n"
            f"Tanlang:"
        )
        await callback.message.edit_text(text, reply_markup=animes_kb(animes, section_id), parse_mode=ParseMode.HTML)
    else:
        await show_sections(callback)
    await callback.answer()

================================================================================
FAYL: handlers/admin.py
================================================================================
from aiogram import Router, F, Bot
from aiogram.types import Message, CallbackQuery
from aiogram.filters import Command
from aiogram.fsm.context import FSMContext
from aiogram.enums import ParseMode

import database as db
from keyboards.admin_kb import (
    admin_panel_kb, admin_sections_kb, admin_section_actions_kb,
    admin_animes_list_kb, admin_anime_actions_kb, confirm_kb,
    admin_vips_kb, admin_admins_kb, cancel_inline_kb
)
from keyboards.user_kb import back_main_kb
from states import AdminStates
from config import settings

router = Router()


# ========== FILTER ==========
async def admin_filter(user_id: int) -> bool:
    return await db.is_admin(user_id)


# ========== PANEL ==========
@router.callback_query(F.data == "admin_panel")
async def admin_panel(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        await callback.answer("⛔ Siz admin emassiz!", show_alert=True)
        return
    
    text = (
        "<b>🛠 Admin Panel</b>\n\n"
        "200K+ kanal uchun professional boshqaruv paneli.\n"
        "Quyidagi bo'limlardan birini tanlang:"
    )
    await callback.message.edit_text(text, reply_markup=admin_panel_kb(), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.message(Command("admin"))
async def cmd_admin(message: Message):
    if not await admin_filter(message.from_user.id):
        await message.answer("⛔ Siz admin emassiz!")
        return
    text = (
        "<b>🛠 Admin Panel</b>\n\n"
        "200K+ kanal uchun professional boshqaruv paneli."
    )
    await message.answer(text, reply_markup=admin_panel_kb(), parse_mode=ParseMode.HTML)


# ========== STATS ==========
@router.callback_query(F.data == "admin_stats")
async def admin_stats(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    users = await db.get_users_count()
    animes = await db.get_animes_count()
    sections = await db.get_sections_count()
    vips = len(await db.get_all_vips())
    admins = len(await db.get_all_admins())
    
    text = (
        "<b>📊 Statistika</b>\n\n"
        f"👥 Foydalanuvchilar: <b>{users}</b>\n"
        f"🎬 Animelar: <b>{animes}</b>\n"
        f"📁 Bo'limlar: <b>{sections}</b>\n"
        f"👑 VIP a'zolar: <b>{vips}</b>\n"
        f"👮 Adminlar: <b>{admins}</b>\n"
    )
    await callback.message.edit_text(text, reply_markup=admin_panel_kb(), parse_mode=ParseMode.HTML)
    await callback.answer()


# ========== SECTIONS ==========
@router.callback_query(F.data == "admin_sections")
async def admin_sections(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    sections = await db.get_sections()
    text = f"<b>📁 Bo'limlar boshqaruvi</b>\n\nJami: {len(sections)} ta"
    await callback.message.edit_text(text, reply_markup=admin_sections_kb(sections), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "admin_add_section")
async def admin_add_section_start(callback: CallbackQuery, state: FSMContext):
    if not await admin_filter(callback.from_user.id):
        return
    await state.set_state(AdminStates.add_section_name)
    await callback.message.edit_text(
        "📁 <b>Yangi bo'lim qo'shish</b>\n\nBo'lim nomini yuboring:",
        reply_markup=cancel_inline_kb(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.message(AdminStates.add_section_name)
async def admin_add_section_name(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    await state.update_data(section_name=message.text.strip())
    await state.set_state(AdminStates.add_section_emoji)
    await message.answer(
        "Endi bo'lim uchun emoji yuboring (masalan: 🔥 yoki ⚔️):\nYoki /skip deb o'tkazib yuboring.",
        reply_markup=cancel_inline_kb()
    )


@router.message(AdminStates.add_section_emoji)
async def admin_add_section_emoji(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    data = await state.get_data()
    name = data["section_name"]
    emoji = message.text.strip() if message.text != "/skip" else "📁"
    
    section_id = await db.add_section(name, emoji)
    await state.clear()
    await message.answer(
        f"✅ Bo'lim qo'shildi!\n\n{emoji} <b>{name}</b> (ID: {section_id})",
        reply_markup=admin_panel_kb(),
        parse_mode=ParseMode.HTML
    )


@router.callback_query(F.data.startswith("admin_section_"))
async def admin_section_detail(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    if callback.data.startswith("admin_section_animes_"):
        return  # boshqa handler
    section_id = int(callback.data.split("_")[2])
    section = await db.get_section(section_id)
    if not section:
        await callback.answer("Topilmadi", show_alert=True)
        return
    animes_count = len(await db.get_animes_by_section(section_id))
    text = (
        f"<b>{section.get('emoji', '📁')} {section['name']}</b>\n\n"
        f"📝 {section.get('description') or 'Tavsif yoʻq'}\n"
        f"🎬 Animelar soni: <b>{animes_count}</b>"
    )
    await callback.message.edit_text(text, reply_markup=admin_section_actions_kb(section_id), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("admin_edit_section_"))
async def admin_edit_section(callback: CallbackQuery, state: FSMContext):
    if not await admin_filter(callback.from_user.id):
        return
    section_id = int(callback.data.split("_")[3])
    await state.update_data(edit_section_id=section_id)
    await state.set_state(AdminStates.edit_section_name)
    await callback.message.edit_text(
        "✏️ Yangi nomni yuboring (yoki /skip):",
        reply_markup=cancel_inline_kb()
    )
    await callback.answer()


@router.message(AdminStates.edit_section_name)
async def admin_edit_section_name(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    data = await state.get_data()
    section_id = data["edit_section_id"]
    if message.text != "/skip":
        await db.update_section(section_id, name=message.text.strip())
    await state.set_state(AdminStates.edit_section_emoji)
    await message.answer("Yangi emoji yuboring (yoki /skip):")


@router.message(AdminStates.edit_section_emoji)
async def admin_edit_section_emoji(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    data = await state.get_data()
    section_id = data["edit_section_id"]
    if message.text != "/skip":
        await db.update_section(section_id, emoji=message.text.strip())
    await state.clear()
    section = await db.get_section(section_id)
    await message.answer(
        f"✅ Bo'lim yangilandi!\n{section.get('emoji')} {section['name']}",
        reply_markup=admin_panel_kb(),
        parse_mode=ParseMode.HTML
    )


@router.callback_query(F.data.startswith("admin_del_section_"))
async def admin_del_section(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    section_id = int(callback.data.split("_")[3])
    section = await db.get_section(section_id)
    text = f"🗑 <b>{section['name']}</b> bo'limini o'chirishni tasdiqlaysizmi?\n\n⚠️ Ichidagi barcha animelar ham o'chadi!"
    await callback.message.edit_text(text, reply_markup=confirm_kb("del_section", section_id), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("confirm_del_section_"))
async def confirm_del_section(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    section_id = int(callback.data.split("_")[3])
    await db.delete_section(section_id)
    await callback.message.edit_text("✅ Bo'lim o'chirildi!", reply_markup=admin_panel_kb())
    await callback.answer()


# ========== ANIMES ==========
@router.callback_query(F.data == "admin_animes")
async def admin_animes(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    # Barcha animelarni ko'rsatish qiyin, bo'lim tanlash
    sections = await db.get_sections()
    if not sections:
        await callback.message.edit_text(
            "Avval bo'lim qo'shing!",
            reply_markup=admin_panel_kb()
        )
        return
    text = "<b>🎬 Animelar boshqaruvi</b>\n\nBo'limni tanlang:"
    from aiogram.utils.keyboard import InlineKeyboardBuilder
    from aiogram.types import InlineKeyboardButton
    builder = InlineKeyboardBuilder()
    for s in sections:
        builder.row(
            InlineKeyboardButton(
                text=f"{s.get('emoji', '📁')} {s['name']}",
                callback_data=f"admin_section_animes_{s['id']}"
            )
        )
    builder.row(InlineKeyboardButton(text="◀️ Admin panel", callback_data="admin_panel"))
    await callback.message.edit_text(text, reply_markup=builder.as_markup(), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("admin_section_animes_"))
async def admin_section_animes(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    section_id = int(callback.data.split("_")[3])
    section = await db.get_section(section_id)
    animes = await db.get_animes_by_section(section_id)
    text = f"<b>{section.get('emoji')} {section['name']}</b> — animelar\nJami: {len(animes)}"
    await callback.message.edit_text(text, reply_markup=admin_animes_list_kb(animes, section_id), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("admin_add_anime_"))
async def admin_add_anime_start(callback: CallbackQuery, state: FSMContext):
    if not await admin_filter(callback.from_user.id):
        return
    section_id = int(callback.data.split("_")[3])
    if section_id == 0:
        # Bo'lim tanlash kerak
        sections = await db.get_sections()
        from aiogram.utils.keyboard import InlineKeyboardBuilder
        from aiogram.types import InlineKeyboardButton
        builder = InlineKeyboardBuilder()
        for s in sections:
            builder.row(
                InlineKeyboardButton(
                    text=f"{s.get('emoji')} {s['name']}",
                    callback_data=f"admin_add_anime_{s['id']}"
                )
            )
        builder.row(InlineKeyboardButton(text="❌ Bekor", callback_data="admin_cancel"))
        await callback.message.edit_text("Anime qaysi bo'limga?", reply_markup=builder.as_markup())
        await callback.answer()
        return
    
    await state.update_data(anime_section_id=section_id)
    await state.set_state(AdminStates.add_anime_title)
    await callback.message.edit_text(
        "🎬 <b>Yangi anime qo'shish</b>\n\nAnime nomini yuboring:",
        reply_markup=cancel_inline_kb(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.message(AdminStates.add_anime_title)
async def admin_add_anime_title(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    await state.update_data(anime_title=message.text.strip())
    await state.set_state(AdminStates.add_anime_desc)
    await message.answer("📝 Tavsifni yuboring (yoki /skip):")


@router.message(AdminStates.add_anime_desc)
async def admin_add_anime_desc(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    desc = None if message.text == "/skip" else message.text.strip()
    await state.update_data(anime_desc=desc)
    await state.set_state(AdminStates.add_anime_cover)
    await message.answer("🖼 Cover rasmni yuboring (photo) yoki /skip:")


@router.message(AdminStates.add_anime_cover)
async def admin_add_anime_cover(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    cover = None
    if message.photo:
        cover = message.photo[-1].file_id
    elif message.text != "/skip":
        await message.answer("Rasm yuboring yoki /skip deb yozing.")
        return
    await state.update_data(anime_cover=cover)
    await state.set_state(AdminStates.add_anime_links)
    await message.answer(
        "📥 Yuklab olish havolalarini yuboring (bir nechta bo'lsa har birini yangi qatordan):\n"
        "Masalan:\n360p: https://...\n720p: https://...\nYoki /skip:"
    )


@router.message(AdminStates.add_anime_links)
async def admin_add_anime_links(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    links = None if message.text == "/skip" else message.text.strip()
    await state.update_data(anime_links=links)
    await state.set_state(AdminStates.add_anime_vip)
    from aiogram.utils.keyboard import InlineKeyboardBuilder
    from aiogram.types import InlineKeyboardButton
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="👑 Ha, VIP", callback_data="anime_vip_yes"),
        InlineKeyboardButton(text="Oddiy", callback_data="anime_vip_no")
    )
    await message.answer("Bu anime VIP kontentmi?", reply_markup=builder.as_markup())


@router.callback_query(F.data.in_({"anime_vip_yes", "anime_vip_no"}))
async def admin_add_anime_vip(callback: CallbackQuery, state: FSMContext):
    if not await admin_filter(callback.from_user.id):
        return
    is_vip = callback.data == "anime_vip_yes"
    await state.update_data(anime_vip=is_vip)
    await state.set_state(AdminStates.add_anime_episodes)
    await callback.message.edit_text("🎞 Qismlar sonini yuboring (raqam, masalan 12):")
    await callback.answer()


@router.message(AdminStates.add_anime_episodes)
async def admin_add_anime_episodes(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    try:
        episodes = int(message.text.strip())
    except ValueError:
        await message.answer("Faqat raqam kiriting!")
        return
    
    data = await state.get_data()
    anime_id = await db.add_anime(
        section_id=data["anime_section_id"],
        title=data["anime_title"],
        description=data.get("anime_desc"),
        cover_file_id=data.get("anime_cover"),
        download_links=data.get("anime_links"),
        is_vip=data.get("anime_vip", False),
        episodes_count=episodes
    )
    await state.clear()
    
    vip_text = "👑 VIP" if data.get("anime_vip") else "Oddiy"
    await message.answer(
        f"✅ Anime muvaffaqiyatli qo'shildi!\n\n"
        f"🎬 <b>{data['anime_title']}</b>\n"
        f"ID: {anime_id}\n"
        f"Status: {vip_text}\n"
        f"Qismlar: {episodes}",
        reply_markup=admin_panel_kb(),
        parse_mode=ParseMode.HTML
    )


@router.callback_query(F.data.startswith("admin_anime_"))
async def admin_anime_detail(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    anime_id = int(callback.data.split("_")[2])
    anime = await db.get_anime(anime_id)
    if not anime:
        await callback.answer("Topilmadi", show_alert=True)
        return
    section = await db.get_section(anime["section_id"])
    text = (
        f"<b>🎬 {anime['title']}</b>\n\n"
        f"📁 Bo'lim: {section.get('emoji')} {section['name'] if section else '?'}\n"
        f"📝 {anime.get('description') or 'Yoʻq'}\n"
        f"👑 VIP: {'Ha' if anime.get('is_vip') else 'Yoʻq'}\n"
        f"🎞 Qismlar: {anime.get('episodes_count')}\n"
        f"👁 Views: {anime.get('views')}\n"
        f"📥 Links: {bool(anime.get('download_links'))}"
    )
    await callback.message.edit_text(text, reply_markup=admin_anime_actions_kb(anime_id), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("admin_del_anime_"))
async def admin_del_anime(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    anime_id = int(callback.data.split("_")[3])
    anime = await db.get_anime(anime_id)
    text = f"🗑 <b>{anime['title']}</b> ni o'chirishni tasdiqlaysizmi?"
    await callback.message.edit_text(text, reply_markup=confirm_kb("del_anime", anime_id), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data.startswith("confirm_del_anime_"))
async def confirm_del_anime(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    anime_id = int(callback.data.split("_")[3])
    await db.delete_anime(anime_id)
    await callback.message.edit_text("✅ Anime o'chirildi!", reply_markup=admin_panel_kb())
    await callback.answer()


# ========== VIP ==========
@router.callback_query(F.data == "admin_vips")
async def admin_vips(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    vips = await db.get_all_vips()
    text = f"<b>👑 VIP boshqaruvi</b>\n\nJami VIP: {len(vips)}"
    await callback.message.edit_text(text, reply_markup=admin_vips_kb(vips), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "admin_add_vip")
async def admin_add_vip_start(callback: CallbackQuery, state: FSMContext):
    if not await admin_filter(callback.from_user.id):
        return
    await state.set_state(AdminStates.add_vip_id)
    await callback.message.edit_text(
        "👑 VIP qo'shish\n\nFoydalanuvchi ID sini yuboring (raqam):",
        reply_markup=cancel_inline_kb()
    )
    await callback.answer()


@router.message(AdminStates.add_vip_id)
async def admin_add_vip_process(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    try:
        user_id = int(message.text.strip())
    except ValueError:
        await message.answer("Faqat raqam kiriting!")
        return
    await db.set_vip(user_id, True)
    await state.clear()
    await message.answer(
        f"✅ <code>{user_id}</code> VIP qilindi!",
        reply_markup=admin_panel_kb(),
        parse_mode=ParseMode.HTML
    )


@router.callback_query(F.data.startswith("admin_vip_"))
async def admin_vip_detail(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    user_id = int(callback.data.split("_")[2])
    user = await db.get_user(user_id)
    name = user.get("full_name") or user.get("username") or str(user_id) if user else str(user_id)
    
    from aiogram.utils.keyboard import InlineKeyboardBuilder
    from aiogram.types import InlineKeyboardButton
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="🗑 VIP dan olib tashlash", callback_data=f"admin_remove_vip_{user_id}")
    )
    builder.row(InlineKeyboardButton(text="◀️ Orqaga", callback_data="admin_vips"))
    
    await callback.message.edit_text(
        f"<b>👑 {name}</b>\nID: <code>{user_id}</code>",
        reply_markup=builder.as_markup(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("admin_remove_vip_"))
async def admin_remove_vip(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    user_id = int(callback.data.split("_")[3])
    await db.set_vip(user_id, False)
    await callback.message.edit_text("✅ VIP olib tashlandi!", reply_markup=admin_panel_kb())
    await callback.answer()


# ========== ADMINS ==========
@router.callback_query(F.data == "admin_admins")
async def admin_admins(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    admins = await db.get_all_admins()
    # Config dagi adminlarni ham qo'shish
    text = f"<b>👮 Adminlar boshqaruvi</b>\n\nJami: {len(admins) + len(settings.ADMIN_IDS)}"
    await callback.message.edit_text(text, reply_markup=admin_admins_kb(admins), parse_mode=ParseMode.HTML)
    await callback.answer()


@router.callback_query(F.data == "admin_add_admin")
async def admin_add_admin_start(callback: CallbackQuery, state: FSMContext):
    if not await admin_filter(callback.from_user.id):
        return
    await state.set_state(AdminStates.add_admin_id)
    await callback.message.edit_text(
        "👮 Yangi admin ID sini yuboring:",
        reply_markup=cancel_inline_kb()
    )
    await callback.answer()


@router.message(AdminStates.add_admin_id)
async def admin_add_admin_process(message: Message, state: FSMContext):
    if not await admin_filter(message.from_user.id):
        return
    try:
        user_id = int(message.text.strip())
    except ValueError:
        await message.answer("Faqat raqam!")
        return
    await db.add_admin(user_id, added_by=message.from_user.id)
    await state.clear()
    await message.answer(
        f"✅ <code>{user_id}</code> admin qilindi!",
        reply_markup=admin_panel_kb(),
        parse_mode=ParseMode.HTML
    )


@router.callback_query(F.data.startswith("admin_admin_"))
async def admin_admin_detail(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    user_id = int(callback.data.split("_")[2])
    if user_id in settings.ADMIN_IDS:
        await callback.answer("Asosiy adminni o'chirib bo'lmaydi!", show_alert=True)
        return
    
    from aiogram.utils.keyboard import InlineKeyboardBuilder
    from aiogram.types import InlineKeyboardButton
    builder = InlineKeyboardBuilder()
    builder.row(
        InlineKeyboardButton(text="🗑 Adminlikdan olish", callback_data=f"admin_remove_admin_{user_id}")
    )
    builder.row(InlineKeyboardButton(text="◀️ Orqaga", callback_data="admin_admins"))
    
    await callback.message.edit_text(
        f"<b>👮 Admin</b>\nID: <code>{user_id}</code>",
        reply_markup=builder.as_markup(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.callback_query(F.data.startswith("admin_remove_admin_"))
async def admin_remove_admin(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    user_id = int(callback.data.split("_")[3])
    await db.remove_admin(user_id)
    await callback.message.edit_text("✅ Admin olib tashlandi!", reply_markup=admin_panel_kb())
    await callback.answer()


# ========== BROADCAST ==========
@router.callback_query(F.data == "admin_broadcast")
async def admin_broadcast_start(callback: CallbackQuery, state: FSMContext):
    if not await admin_filter(callback.from_user.id):
        return
    await state.set_state(AdminStates.broadcast_message)
    await callback.message.edit_text(
        "📢 <b>Xabar yuborish</b>\n\nBarcha foydalanuvchilarga yuboriladigan xabarni yuboring (matn, rasm, video...):",
        reply_markup=cancel_inline_kb(),
        parse_mode=ParseMode.HTML
    )
    await callback.answer()


@router.message(AdminStates.broadcast_message)
async def admin_broadcast_process(message: Message, state: FSMContext, bot: Bot):
    if not await admin_filter(message.from_user.id):
        return
    await state.clear()
    await message.answer("⏳ Xabar yuborilmoqda...")
    
    # Oddiy implementatsiya — barcha userlarni olish
    async with __import__("aiosqlite").connect(settings.DB_PATH) as conn:
        conn.row_factory = __import__("aiosqlite").Row
        async with conn.execute("SELECT user_id FROM users") as cursor:
            users = await cursor.fetchall()
    
    success = 0
    fail = 0
    for u in users:
        try:
            await message.copy_to(u["user_id"])
            success += 1
        except Exception:
            fail += 1
    
    await message.answer(
        f"✅ Yuborildi: {success}\n❌ Xato: {fail}",
        reply_markup=admin_panel_kb()
    )


# ========== POST TO CHANNEL ==========
@router.callback_query(F.data == "admin_post_channel")
async def admin_post_channel(callback: CallbackQuery):
    if not await admin_filter(callback.from_user.id):
        return
    if not settings.CHANNEL_ID:
        await callback.answer("CHANNEL_ID sozlanmagan!", show_alert=True)
        return
    await callback.message.edit_text(
        "📤 Kanalga post qilish uchun avval anime tanlang (Animelar bo'limidan).",
        reply_markup=admin_panel_kb()
    )
    await callback.answer()


@router.callback_query(F.data.startswith("admin_post_anime_"))
async def admin_post_anime(callback: CallbackQuery, bot: Bot):
    if not await admin_filter(callback.from_user.id):
        return
    if not settings.CHANNEL_ID:
        await callback.answer("CHANNEL_ID .env da sozlanmagan!", show_alert=True)
        return
    
    anime_id = int(callback.data.split("_")[3])
    anime = await db.get_anime(anime_id)
    if not anime:
        await callback.answer("Topilmadi", show_alert=True)
        return
    
    section = await db.get_section(anime["section_id"])
    vip = "👑 VIP | " if anime.get("is_vip") else ""
    text = (
        f"{vip}<b>🎬 {anime['title']}</b>\n\n"
        f"{anime.get('description') or ''}\n\n"
        f"📁 {section.get('emoji', '')} {section['name'] if section else ''}\n"
        f"🎞 Qismlar: {anime.get('episodes_count')}\n\n"
        f"👉 Bot orqali yuklab oling: @{ (await bot.get_me()).username }"
    )
    
    try:
        if anime.get("cover_file_id"):
            await bot.send_photo(
                settings.CHANNEL_ID,
                photo=anime["cover_file_id"],
                caption=text,
                parse_mode=ParseMode.HTML
            )
        else:
            await bot.send_message(
                settings.CHANNEL_ID,
                text,
                parse_mode=ParseMode.HTML
            )
        await callback.answer("✅ Kanalga joylandi!", show_alert=True)
    except Exception as e:
        await callback.answer(f"Xato: {str(e)[:50]}", show_alert=True)


# ========== CANCEL ==========
@router.callback_query(F.data == "admin_cancel")
async def admin_cancel(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await callback.message.edit_text(
        "❌ Bekor qilindi.",
        reply_markup=admin_panel_kb()
    )
    await callback.answer()


@router.callback_query(F.data.startswith("cancel_"))
async def cancel_action(callback: CallbackQuery, state: FSMContext):
    await state.clear()
    await callback.message.edit_text(
        "❌ Bekor qilindi.",
        reply_markup=admin_panel_kb()
    )
    await callback.answer()

================================================================================
FAYL: handlers/__init__.py
================================================================================
from .user import router as user_router
from .admin import router as admin_router

__all__ = ["user_router", "admin_router"]

================================================================================
FAYL: keyboards/__init__.py
================================================================================
from .user_kb import *
from .admin_kb import *

================================================================================
FAYL: bot.py
================================================================================
import asyncio
import logging
from aiogram import Bot, Dispatcher
from aiogram.client.default import DefaultBotProperties
from aiogram.enums import ParseMode
from aiogram.fsm.storage.memory import MemoryStorage
from loguru import logger

from config import settings
from database import init_db
from handlers import user_router, admin_router


async def main():
    # Logging
    logging.basicConfig(level=logging.INFO)
    logger.info("Bot ishga tushmoqda...")
    
    # DB
    await init_db()
    logger.info("Database tayyor.")
    
    # Bot & DP
    bot = Bot(
        token=settings.BOT_TOKEN,
        default=DefaultBotProperties(parse_mode=ParseMode.HTML)
    )
    dp = Dispatcher(storage=MemoryStorage())
    
    # Routers
    dp.include_router(user_router)
    dp.include_router(admin_router)
    
    # Start
    logger.success(f"Bot @{ (await bot.get_me()).username } ishga tushdi!")
    await dp.start_polling(bot)


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        logger.info("Bot to'xtatildi.")

================================================================================
FAYL: requirements.txt
================================================================================
aiogram==3.15.0
aiosqlite==0.20.0
python-dotenv==1.0.1
pydantic-settings==2.7.0
loguru==0.7.2

================================================================================
FAYL: .env  (muhim!)
================================================================================
BOT_TOKEN=8630962311:AAFT6AtIDoSSVB7ek95wHg55qD4Yq9yemSU
ADMIN_IDS=[8318241400]
CHANNEL_ID=-1003940243822
CHANNEL_USERNAME=
DB_PATH=data/bot.db

================================================================================
TUGADI. Barcha kodlar tayyor.
================================================================================
