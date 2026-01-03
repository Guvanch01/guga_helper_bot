import logging
import os
import re
import random
import urllib.parse
import requests
import string
import asyncio  
from docx.oxml import parse_xml
from docx.oxml.ns import nsdecls
from datetime import datetime
from typing import Optional, Dict, Any
from io import BytesIO
import requests
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, InputFile
from telegram.ext import (
    Application, CommandHandler, CallbackQueryHandler,
    MessageHandler, filters, ContextTypes, ConversationHandler
)
from docx import Document
from docx.shared import Pt, Cm, Inches
from docx.enum.text import WD_ALIGN_PARAGRAPH
from docx.enum.style import WD_STYLE_TYPE
from pptx import Presentation
from pptx.util import Inches as PptxInches, Pt as PptxPt
from pptx.dml.color import RGBColor
from io import BytesIO
from duckduckgo_search import DDGS
from pptx.enum.text import PP_ALIGN
import openpyxl
from openpyxl.styles import Font, Alignment, Border, Side, PatternFill

USED_IMAGE_URLS = set()

logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

BOT_TOKEN = "8536135266:AAGz9vM4M6-LiJJdxXJDduUGXZOh6O2w0N4"
ADMIN_ID = 6581335835
GEMINI_API_KEY = "AIzaSyDobeNv2ai8c0v2n32gwi4bj1FCbMZhrI4"
GROQ_API_KEY = "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"
PIXABAY_API_KEY = "54003630-714e9f86777060ab07858940b"

# BAŞDA goşuň:
logger.info(f"🔑 Gemini API Key: {GEMINI_API_KEY[:10]}...{GEMINI_API_KEY[-5:]}")
logger.info(f"🔑 Groq API Key: {GROQ_API_KEY[:10]}...{GROQ_API_KEY[-5:]}")

HOLIDAY_PROMOS = {
    "NEWYEAR25": {"date": "01-01", "discount": 30, "name": "Новый Год"},
    "WOMEN8": {"date": "03-08", "discount": 20, "name": "8 Марта"},
    "NOWRUZ": {"date": "03-21", "discount": 25, "name": "Новруз Байрам"},
    "NEUTRALITY": {"date": "12-12", "discount": 20, "name": "День Нейтралитета"},
    "STUDENT_DAY": {"date": "11-17", "discount": 15, "name": "День Студента"},
    "JAN2TEST": {"date": "01-02", "discount": 50, "name": "Тестовый День"}
}

PAYMENTS = {
    "BY": {"card": "1234 5678 9012 3456", "name": "IVANOV IVAN", "bank": "Беларусбанк", "currency": "BYN"},
    "RU": {"card": "9876 5432 1098 7654", "name": "ИВАНОВ ИВАН", "bank": "Сбербанк", "currency": "RUB"}
}

PRICES = {
    "BY": {
        "referat": {"min": 5, "max": 25, "price_per_page": 0.85},
        "doklad": {"min": 1, "max": 4, "price_per_page": 0.85},
        "esse": {"min": 1, "max": 6, "price_per_page": 0.85},
        "kursovaya": {"min": 25, "max": 50, "price_per_page": 0.95},
        "presentation": {"min": 5, "max": 20, "price_per_page": 0.85}
        # ❌ "table" AÝRYLDY!
    },
    "RU": {
        "referat": {"min": 5, "max": 25, "price_per_page": 18},
        "doklad": {"min": 1, "max": 4, "price_per_page": 18},
        "esse": {"min": 1, "max": 6, "price_per_page": 18},
        "kursovaya": {"min": 25, "max": 50, "price_per_page": 25},
        "presentation": {"min": 5, "max": 20, "price_per_page": 23}
        # ❌ "table" AÝRYLDY!
    }
}

WORK_TYPES = {
    "referat": {"ru": "Реферат", "en": "Abstract/Report"},
    "doklad": {"ru": "Доклад", "en": "Report/Presentation"},
    "esse": {"ru": "Эссе", "en": "Essay"},
    "kursovaya": {"ru": "Курсовая работа", "en": "Term Paper"},
    "presentation": {"ru": "Презентация", "en": "Presentation"},
    "table": {"ru": "Таблица", "en": "Table Work"}
}

PROMO_CODES = {"WELCOME": 20, "FRIEND": 20, "VIP2025": 8}

(SELECT_COUNTRY, SELECT_LANG, SELECT_WORK_TYPE, SELECT_PAGES, 
 ENTER_TOPIC, ENTER_UNIVERSITY, ENTER_FACULTY, ENTER_SUBJECT,
 ENTER_FULLNAME, ENTER_COURSE, ENTER_GROUP, ENTER_TEACHER,
 ENTER_CITY, ENTER_PHONE, UPLOAD_ZADANIE, PAYMENT_PHOTO) = range(16)

users_db: Dict[int, dict] = {}
orders_db: Dict[str, dict] = {}
pending_payments: Dict[str, dict] = {}

TEXTS = {
    "ru": {
        "welcome": """🎓 *АКАДЕМИЧЕСКИЙ ПОМОЩНИК*

Добро пожаловать! 👋
━━━━━━━━━━━━━━━━━━━━
🔥 *АКЦИИ:*
🎁 Каждый 8-й заказ БЕСПЛАТНО!
☀️ Утренняя скидка (06:00-07:00): -10%
👥 Приведи друга: -30% ОБОИМ!
🎉 Выходные: -10%
━━━━━━━━━━━━━━━━━━━━
✅ Гарантия качества
✅ Быстрая доставка""",
        "select_country": "🌍 *Выберите вашу страну:*",
        "select_work_type": "📝 *Выберите тип работы:*",
        "select_pages": "📄 *Выберите количество страниц:*",
        "enter_topic": "📝 *Введите тему работы:*",
        "enter_university": "🏛 *Введите название университета:*",
        "enter_faculty": "📚 *Введите название факультета:*",
        "enter_subject": "📖 *Введите название предмета:*",
        "enter_fullname": "👤 *Введите ваше ФИО:*",
        "enter_course": "🎓 *Введите курс обучения:*",
        "enter_group": "👥 *Введите номер группы:*",
        "enter_teacher": "👨‍🏫 *Введите ФИО преподавателя:*",
        "enter_city": "🏙 *Введите город:*",
        "enter_phone": "📱 *Введите номер телефона:*",
        "new_order": "📝 Новый заказ",
        "promotions": "🎁 Акции",
        "promo_code": "🏷️ Промокод",
        "my_account": "📊 Мой аккаунт",
        "referral": "👥 Реферал",
        "help": "❓ Помощь",
        "back": "🔙 Назад",
        "cancel": "❌ Отмена"
    },
    "en": {
        "welcome": """🎓 *ACADEMIC ASSISTANT*

Welcome! 👋
━━━━━━━━━━━━━━━━━━━━
🔥 *PROMOTIONS:*
🎁 Every 8th order FREE!
☀️ Morning (06:00-07:00): -10%
👥 Refer friend: -30% FOR BOTH!
🎉 Weekend: -10%
━━━━━━━━━━━━━━━━━━━━
✅ Quality guaranteed
✅ Fast delivery""",
        "select_country": "🌍 *Select your country:*",
        "select_work_type": "📝 *Select work type:*",
        "select_pages": "📄 *Select pages:*",
        "enter_topic": "📝 *Enter topic:*",
        "enter_university": "🏛 *Enter university:*",
        "enter_faculty": "📚 *Enter faculty:*",
        "enter_subject": "📖 *Enter subject:*",
        "enter_fullname": "👤 *Enter full name:*",
        "enter_course": "🎓 *Enter course:*",
        "enter_group": "👥 *Enter group:*",
        "enter_teacher": "👨‍🏫 *Enter teacher:*",
        "enter_city": "🏙 *Enter city:*",
        "enter_phone": "📱 *Enter phone:*",
        "new_order": "📝 New Order",
        "promotions": "🎁 Promotions",
        "promo_code": "🏷️ Promo",
        "my_account": "📊 Account",
        "referral": "👥 Referral",
        "help": "❓ Help",
        "back": "🔙 Back",
        "cancel": "❌ Cancel"
    }
}

import random

def generate_ai_image_url(prompt: str) -> str:
    """
    ✅ Pollinations AI arkaly täze we üýtgeşik surat döretmek.
    Mugt we hiç hili açar (key) soraýan däl.
    """
    try:
        # Suratyň 100% üýtgeşik bolmagy üçin tötänleýin san (seed)
        random_seed = random.randint(1, 999999)
        
        # Gözleg sözlerini arassalamak we iňlis diline terjime etmek (AI iňlisçe gowy düşünýär)
        # Eger kodyňyza terjimeçi goşmadyk bolsaňyz, iň bolmanda sözleri arassalaň
        clean_prompt = prompt.replace(" ", "%20")
        
        # Pollinations AI URL formaty
        # width=1024, height=768 (Prezentasiýa üçin laýyk ölçeg)
        image_url = f"https://pollinations.ai/p/{clean_prompt}?width=1024&height=768&seed={random_seed}&nologo=true"
        
        logger.info(f"🎨 AI Image Generated: {image_url}")
        return image_url
    except Exception as e:
        logger.error(f"❌ AI Image Generation failed: {e}")
        return "https://images.pexels.com/photos/3183150/pexels-photo-3183150.jpeg" # Fallback

def generate_order_id() -> str:
    return "ORD" + ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))

def get_user(user_id: int) -> dict:
    if user_id not in users_db:
        users_db[user_id] = {
            "orders_count": 0, "total_spent": 0, "bonus": 0,
            "used_promos": [], "referrals": [], "language": "ru",
            "country": None, "created": datetime.now().isoformat()
        }
    return users_db[user_id]

def get_text(user_id: int, key: str) -> str:
    user = get_user(user_id)
    return TEXTS.get(user.get("language", "ru"), TEXTS["ru"]).get(key, key)

def calculate_price(country: str, work_type: str, pages: int) -> float:
    price_info = PRICES[country][work_type]
    return pages * price_info.get("price_per_item" if work_type == "table" else "price_per_page")

def calculate_final_price(user_id: int, base_price: float, promo: str = None) -> tuple:
    user = get_user(user_id)
    discounts = []
    total_discount = 0
    
    if (user["orders_count"] + 1) % 8 == 0 and user["orders_count"] > 0:
        return 0, [("🎁 8-й заказ БЕСПЛАТНО!", 100)]
    
    if user.get("referral_discount") == 30:
        discounts.append(("👥 Реферал", 30))
        total_discount += 30
        user["referral_discount"] = 0
    
    if 6 <= datetime.now().hour < 7:
        discounts.append(("☀️ Утро (06:00-07:00)", 10))
        total_discount += 10
    
    if datetime.now().weekday() >= 5:
        discounts.append(("🎉 Выходные", 10))
        total_discount += 10
    
    if promo and promo.upper() in PROMO_CODES:
        if promo.upper() not in user["used_promos"]:
            disc = PROMO_CODES[promo.upper()]
            discounts.append((f"🏷️ {promo.upper()}", disc))
            total_discount += disc
    
    total_discount = min(total_discount, 50)
    final = base_price * (100 - total_discount) / 100
    return round(final, 2), discounts

def get_currency_symbol(country: str) -> str:
    return "BYN" if country == "BY" else "₽"

# ============== BOT HANDLERS ==============

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    logger.info(f"👤 User started: ID={user.id}, Name={user.full_name}")
    user_data = get_user(user.id)
    
    if context.args and len(context.args) > 0 and context.args[0].isdigit():
        ref_id = int(context.args[0])
        if ref_id != user.id and ref_id in users_db:
            ref_user = get_user(ref_id)
            if user.id not in ref_user["referrals"]:
                ref_user["referrals"].append(user.id)
                user_data["referral_discount"] = 30
                
                try:
                    await context.bot.send_message(user.id, "🎉 Вы получили 30% скидку от реферала!")
                    await context.bot.send_message(ref_id, f"👥 Ваш друг {user.full_name} присоединился! Оба получили 30%!")
                except:
                    pass
    
    text = "🌍 *Выберите язык / Select language:*"
    keyboard = [[InlineKeyboardButton("🇷🇺 Русский", callback_data="lang_ru"), InlineKeyboardButton("🇬🇧 English", callback_data="lang_en")]]
    
    if update.message:
        await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
    elif update.callback_query:
        await update.callback_query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def select_language(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    lang = query.data.split("_")[1]
    user = get_user(query.from_user.id)
    user["language"] = lang
    
    if user.get("referral_discount") == 30:
        bonus_msg = "🎉 У вас 30% скидка!" if lang == "ru" else "🎉 You have 30% discount!"
        try:
            await query.message.reply_text(bonus_msg)
        except:
            pass
    
    await show_main_menu(update, context)

async def show_main_menu(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    user = get_user(user_id)
    lang = user.get("language", "ru")
    
    text = TEXTS[lang]["welcome"]
    keyboard = [
        [InlineKeyboardButton(TEXTS[lang]["new_order"], callback_data="new_order")],
        [InlineKeyboardButton(TEXTS[lang]["promotions"], callback_data="promotions"), InlineKeyboardButton(TEXTS[lang]["promo_code"], callback_data="enter_promo")],
        [InlineKeyboardButton(TEXTS[lang]["my_account"], callback_data="account"), InlineKeyboardButton(TEXTS[lang]["referral"], callback_data="referral")],
        [InlineKeyboardButton(TEXTS[lang]["help"], callback_data="help")],
        [InlineKeyboardButton("🌍 Language", callback_data="change_lang")]
    ]
    
    if user_id == ADMIN_ID:
        keyboard.append([InlineKeyboardButton("🔐 ADMIN", callback_data="admin")])
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def new_order(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    lang = get_user(user_id).get("language", "ru")
    text = TEXTS[lang]["select_country"]
    
    keyboard = [
        [InlineKeyboardButton("🇧🇾 Беларусь", callback_data="country_BY"), InlineKeyboardButton("🇷🇺 Россия", callback_data="country_RU")],
        [InlineKeyboardButton(TEXTS[lang]["back"], callback_data="main_menu")]
    ]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
    return SELECT_COUNTRY

async def select_country(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    country = query.data.split("_")[1]
    context.user_data["country"] = country
    
    user_id = query.from_user.id
    user = get_user(user_id)
    user["country"] = country
    lang = user.get("language", "ru")
    
    currency = get_currency_symbol(country)
    prices = PRICES[country]
    
    text = f"""📝 *{TEXTS[lang]["select_work_type"]}*

━━━━━━━━━━━━━━━━━━━━
💰 *Цены ({currency}):*

📄 Реферат — {prices['referat']['price_per_page']} {currency}/стр.
📋 Доклад — {prices['doklad']['price_per_page']} {currency}/стр.
✍️ Эссе — {prices['esse']['price_per_page']} {currency}/стр.
📚 Курсовая — {prices['kursovaya']['price_per_page']} {currency}/стр.
🎬 Презентация — {prices['presentation']['price_per_page']} {currency}/сл.
━━━━━━━━━━━━━━━━━━━━"""
    
    keyboard = [
        [InlineKeyboardButton("📄 " + WORK_TYPES["referat"]["ru"], callback_data="work_referat"), 
         InlineKeyboardButton("📋 " + WORK_TYPES["doklad"]["ru"], callback_data="work_doklad")],
        [InlineKeyboardButton("✍️ " + WORK_TYPES["esse"]["ru"], callback_data="work_esse"), 
         InlineKeyboardButton("📚 " + WORK_TYPES["kursovaya"]["ru"], callback_data="work_kursovaya")],
        [InlineKeyboardButton("🎬 " + WORK_TYPES["presentation"]["ru"], callback_data="work_presentation")],
        # ❌ TABLE BUTTON AÝRYLDY!
        [InlineKeyboardButton(TEXTS[lang]["back"], callback_data="new_order")]
    ]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
    return SELECT_WORK_TYPE

async def select_work_type(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    # ✅ PARSE WORK TYPE
    work_type = query.data.split("_")[1]
    context.user_data["work_type"] = work_type  # ✅ SAKLA!
    
    user_id = query.from_user.id
    lang = get_user(user_id).get("language", "ru")
    country = context.user_data["country"]
    
    price_info = PRICES[country][work_type]
    min_pages = price_info["min"]
    max_pages = price_info["max"]
    currency = get_currency_symbol(country)
    
    price_key = "price_per_item" if work_type == "table" else "price_per_page"
    unit = "шт." if work_type == "table" else "стр."
    price_per = price_info[price_key]
    
    text = f"📄 *{TEXTS[lang]['select_pages']}*\n\n💰 Цена: {price_per} {currency}/{unit}\n📏 Диапазон: {min_pages}-{max_pages} {unit}"
    
    keyboard = []
    row = []
    
    if work_type == "esse":
        # ESSE: 1-6
        for i in range(1, 7):
            price = i * price_per
            row.append(InlineKeyboardButton(f"{i} ({price} {currency})", callback_data=f"pages_{i}"))
            if len(row) == 3:
                keyboard.append(row)
                row = []
        if row:
            keyboard.append(row)
    
    elif work_type in ["doklad"]:
        # DOKLAD: 1-10
        for i in range(1, 11):
            price = i * price_per
            row.append(InlineKeyboardButton(f"{i} ({price} {currency})", callback_data=f"pages_{i}"))
            if len(row) == 3:
                keyboard.append(row)
                row = []
        if row:
            keyboard.append(row)
    
    else:
        # REFERAT, KURSOVAYA, PRESENTATION: step by 5
        step = 5
        for i in range(min_pages, max_pages + 1, step):
            price = i * price_per
            row.append(InlineKeyboardButton(f"{i} ({price} {currency})", callback_data=f"pages_{i}"))
            if len(row) == 3:
                keyboard.append(row)
                row = []
        if row:
            keyboard.append(row)
    
    keyboard.append([InlineKeyboardButton(TEXTS[lang]["back"], callback_data=f"country_{country}")])
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
    return SELECT_PAGES

async def select_pages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """✅ Handle page selection"""
    query = update.callback_query
    await query.answer()
    
    pages = int(query.data.split("_")[1])
    context.user_data["pages"] = pages
    
    user_id = query.from_user.id
    lang = get_user(user_id).get("language", "ru")
    country = context.user_data["country"]
    work_type = context.user_data["work_type"]
    
    base_price = calculate_price(country, work_type, pages)
    context.user_data["base_price"] = base_price
    
    # ❌ TABLE SPECIAL CASE AÝRYLDY!
    # Göni TOPIC soraýar
    
    text = TEXTS[lang]["enter_topic"]
    await query.edit_message_text(text, parse_mode='Markdown')
    return ENTER_TOPIC

async def receive_topic(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["topic"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_university"], parse_mode='Markdown')
    return ENTER_UNIVERSITY

async def receive_university(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["university"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_faculty"], parse_mode='Markdown')
    return ENTER_FACULTY

async def receive_faculty(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["faculty"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_subject"], parse_mode='Markdown')
    return ENTER_SUBJECT

async def receive_subject(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["subject"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_fullname"], parse_mode='Markdown')
    return ENTER_FULLNAME

async def receive_fullname(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["fullname"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_course"], parse_mode='Markdown')
    return ENTER_COURSE

async def receive_course(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["course"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_group"], parse_mode='Markdown')
    return ENTER_GROUP

async def receive_group(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["group"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_teacher"], parse_mode='Markdown')
    return ENTER_TEACHER

async def receive_teacher(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["teacher"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_city"], parse_mode='Markdown')
    return ENTER_CITY

async def receive_city(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["city"] = update.message.text.strip()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text(TEXTS[lang]["enter_phone"], parse_mode='Markdown')
    return ENTER_PHONE

async def receive_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["phone"] = update.message.text.strip()
    
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    work_type = context.user_data.get("work_type")
    
    # ✅ KURSOVAYA üçin ZADANIE soramaly
    if work_type == "kursovaya":
        text = "📋 *ЗАДАНИЕ*\n\n📸 Отправьте фото задания\n⏭️ Или нажмите Пропустить"
        keyboard = [[InlineKeyboardButton("⏭️ Пропустить / Skip", callback_data="skip_zadanie")]]
        await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
        return UPLOAD_ZADANIE
    
    # ✅ BEÝLEKILER - GÖNI ORDER SUMMARY
    else:
        context.user_data["zadanie_photo"] = None
        return await show_order_summary(update, context)

async def receive_zadanie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.message.photo:
        context.user_data["zadanie_photo"] = update.message.photo[-1].file_id
        user_id = update.effective_user.id
        lang = get_user(user_id).get("language", "ru")
        msg = "✅ Задание получено!" if lang == "ru" else "✅ Assignment received!"
        await update.message.reply_text(msg)
    return await show_order_summary(update, context)

async def skip_zadanie(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    context.user_data["zadanie_photo"] = None
    return await show_order_summary_from_callback(update, context)

async def show_order_summary(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user = update.effective_user
    user_id = user.id
    user_data = get_user(user_id)
    lang = user_data.get("language", "ru")
    return await _show_order_summary_common(update.message.reply_text, context, user_id, lang)

async def show_order_summary_from_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    user_id = query.from_user.id
    user_data = get_user(user_id)
    lang = user_data.get("language", "ru")
    return await _show_order_summary_common(query.message.reply_text, context, user_id, lang)

async def _show_order_summary_common(reply_func, context: ContextTypes.DEFAULT_TYPE, user_id: int, lang: str):
    """✅ Common logic for showing order summary - FIXED"""
    
    user_data = get_user(user_id)
    
    # ✅ GET ALL REQUIRED DATA FROM CONTEXT
    country = context.user_data.get("country")
    work_type = context.user_data.get("work_type")
    pages = context.user_data.get("pages")
    base_price = context.user_data.get("base_price")
    promo = context.user_data.get("promo_code")
    
    # ✅ VALIDATE DATA
    if not all([country, work_type, pages, base_price]):
        logger.error(f"❌ Missing order data! country={country}, work_type={work_type}, pages={pages}, base_price={base_price}")
        error_msg = "❌ Ошибка: неполные данные заказа!" if lang == "ru" else "❌ Error: incomplete order data!"
        await reply_func(error_msg)
        return PAYMENT_PHOTO
    
    # ✅ CALCULATE FINAL PRICE
    final_price, discounts = calculate_final_price(user_id, base_price, promo)
    context.user_data["final_price"] = final_price
    
    # ✅ GET CURRENCY & PAYMENT INFO
    currency = get_currency_symbol(country)
    payment = PAYMENTS[country]
    work_type_name = WORK_TYPES[work_type]["ru" if lang == "ru" else "en"]
    
    # ✅ DISCOUNT TEXT
    discount_text = ""
    if discounts:
        discount_text = "\n🎉 *Скидки:*\n" if lang == "ru" else "\n🎉 *Discounts:*\n"
        for name, percent in discounts:
            discount_text += f"• {name}: -{percent}%\n"
    
    # ✅ FORMAT INFO
    if work_type == "presentation":
        format_info = "📁 PPTX"
    else:
        format_info = "📁 DOCX"
    
    # ✅ PAGE WORD
    page_word = "Страниц" if lang == "ru" else "Pages"
    if work_type == "presentation":
        page_word = "Слайдов" if lang == "ru" else "Slides"
    
    # ✅ BUILD SUMMARY TEXT
    text = f"""📋 *{"ИТОГ ЗАКАЗА" if lang == "ru" else "ORDER SUMMARY"}*

━━━━━━━━━━━━━━━━━━━━
📝 *{"Работа" if lang == "ru" else "Work"}:*
• {"Тип" if lang == "ru" else "Type"}: {work_type_name}
• {"Тема" if lang == "ru" else "Topic"}: {context.user_data.get('topic', '-')}
• {page_word}: {pages}
• {"Формат" if lang == "ru" else "Format"}: {format_info}

👤 *{"Студент" if lang == "ru" else "Student"}:*
• {"ФИО" if lang == "ru" else "Full Name"}: {context.user_data.get('fullname', '-')}
• {"Университет" if lang == "ru" else "University"}: {context.user_data.get('university', '-')}
• {"Курс" if lang == "ru" else "Course"}: {context.user_data.get('course', '-')}
• {"Группа" if lang == "ru" else "Group"}: {context.user_data.get('group', '-')}
• {"Город" if lang == "ru" else "City"}: {context.user_data.get('city', '-')}
{discount_text}
━━━━━━━━━━━━━━━━━━━━
💰 {"Базовая" if lang == "ru" else "Base"}: ~{base_price} {currency}~
💵 *{"ИТОГО" if lang == "ru" else "TOTAL"}: {final_price} {currency}*

━━━━━━━━━━━━━━━━━━━━
💳 *{"ОПЛАТА" if lang == "ru" else "PAYMENT"}:*
🏦 {payment['bank']}
💳 `{payment['card']}`
👤 {payment['name']}
💵 {final_price} {currency}

━━━━━━━━━━━━━━━━━━━━
📸 *{"Отправьте скриншот оплаты!" if lang == "ru" else "Send payment screenshot!"}*"""
    
    # ✅ KEYBOARD
    keyboard = [[InlineKeyboardButton(TEXTS[lang]["cancel"], callback_data="cancel_order")]]
    
    # ✅ SEND MESSAGE
    try:
        await reply_func(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
        logger.info(f"✅ Order summary sent to user {user_id}")
    except Exception as e:
        logger.error(f"❌ Failed to send order summary: {e}")
        raise
    
    return PAYMENT_PHOTO

async def receive_payment_photo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """✅ Receive payment screenshot and send to admin"""
    
    # ✅ CHECK IF PHOTO EXISTS
    if not update.message.photo:
        logger.warning("⚠️ No photo in message!")
        user_id = update.effective_user.id
        lang = get_user(user_id).get("language", "ru")
        msg = "❌ Отправьте *фото* чека!" if lang == "ru" else "❌ Send *photo* of payment!"
        await update.message.reply_text(msg, parse_mode='Markdown')
        return PAYMENT_PHOTO
    
    # ✅ GET USER & PHOTO
    user = update.effective_user
    user_id = user.id
    user_data = get_user(user_id)
    lang = user_data.get("language", "ru")
    
    # ✅ GET PHOTO - DEFINE photo VARIABLE
    photo = update.message.photo[-1]  # ✅ SAKLA!
    photo_id = photo.file_id
    
    logger.info(f"📸 Payment photo received from user {user_id}: {photo_id}")
    
    # ✅ GENERATE ORDER ID
    order_id = generate_order_id()
    
    # ✅ CREATE ORDER DATA
    order_data = {
        "order_id": order_id,
        "user_id": user_id,
        "username": user.username or "N/A",
        "full_name": user.full_name,
        "language": lang,
        "country": context.user_data["country"],
        "work_type": context.user_data["work_type"],
        "pages": context.user_data["pages"],
        "topic": context.user_data.get("topic", "-"),
        "university": context.user_data.get("university", "-"),
        "faculty": context.user_data.get("faculty", "-"),
        "subject": context.user_data.get("subject", "-"),
        "fullname": context.user_data.get("fullname", "-"),
        "course": context.user_data.get("course", "-"),
        "group": context.user_data.get("group", "-"),
        "teacher": context.user_data.get("teacher", "-"),
        "city": context.user_data.get("city", "-"),
        "phone": context.user_data.get("phone", "-"),
        "base_price": context.user_data["base_price"],
        "final_price": context.user_data["final_price"],
        "promo_code": context.user_data.get("promo_code"),
        "payment_photo": photo_id,  # ✅ USE photo_id
        "zadanie_photo": context.user_data.get("zadanie_photo"),
        "status": "pending",
        "created_at": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    }
    
    # ✅ SAVE TO PENDING
    pending_payments[order_id] = order_data
    
    logger.info(f"✅ Order {order_id} created and saved to pending_payments")
    
    # ✅ NOTIFY CUSTOMER
    currency = get_currency_symbol(order_data["country"])
    customer_msg = f"""✅ <b>ЗАКАЗ ПРИНЯТ!</b>

📋 ID: <code>{order_id}</code>
💵 {order_data['final_price']} {currency}

⏳ Ожидание подтверждения администратора..."""
    
    await update.message.reply_text(customer_msg, parse_mode='HTML')
    
    # ✅ PREPARE ADMIN MESSAGE
    work_type_name = WORK_TYPES[order_data["work_type"]]["ru"]
    country_name = "🇧🇾 Беларусь" if order_data["country"] == "BY" else "🇷🇺 Россия"
    
    admin_text = f"""🆕 <b>НОВЫЙ ЗАКАЗ!</b>

━━━━━━━━━━━━━━━━━━━━
📋 ID: <code>{order_id}</code>
🌍 Страна: {country_name}

👤 <b>Клиент:</b>
• Имя: {user.full_name}
• Username: @{user.username or 'N/A'}
• User ID: <code>{user_id}</code>
• Телефон: {order_data['phone']}

📝 <b>Работа:</b>
• Тип: {work_type_name}
• Тема: {order_data['topic'][:50]}
• Страниц: {order_data['pages']}
• Предмет: {order_data['subject']}

🎓 <b>Данные:</b>
• ВУЗ: {order_data['university']}
• Курс: {order_data['course']}
• Группа: {order_data['group']}

💰 <b>Оплата:</b>
• Цена: {order_data['final_price']} {currency}
━━━━━━━━━━━━━━━━━━━━

⬇️ Скриншот оплаты ниже"""
    
    # ✅ APPROVAL BUTTONS
    keyboard = [
        [
            InlineKeyboardButton("✅ ПОДТВЕРДИТЬ", callback_data=f"confirm_{order_id}"),
            InlineKeyboardButton("❌ ОТКЛОНИТЬ", callback_data=f"reject_{order_id}")
        ]
    ]
    
    # ✅ SEND TO ADMIN
    try:
        # First send text
        await context.bot.send_message(
            chat_id=ADMIN_ID,
            text=admin_text,
            parse_mode='HTML'
        )
        
        # Then send photo with buttons
        await context.bot.send_photo(
            chat_id=ADMIN_ID,
            photo=photo_id,  # ✅ USE photo_id
            caption=f"📸 Скриншот оплаты\n📋 Order: <code>{order_id}</code>",
            reply_markup=InlineKeyboardMarkup(keyboard),
            parse_mode='HTML'
        )
        
        logger.info(f"✅ Order {order_id} sent to admin {ADMIN_ID}")
        
    except Exception as e:
        logger.error(f"❌ Failed to send to admin: {e}")
        
        # Notify customer about error
        error_msg = "⚠️ Техническая ошибка при отправке администратору. Попробуйте снова через /start"
        await update.message.reply_text(error_msg)
        
        # Remove from pending
        if order_id in pending_payments:
            del pending_payments[order_id]
        
        return ConversationHandler.END
    
    # ✅ CLEAR USER DATA
    context.user_data.clear()
    
    return ConversationHandler.END
    
    # ✅ CLEAR USER DATA
    context.user_data.clear()
    
    return ConversationHandler.END

# ============== DOCUMENT GENERATION ==============

def create_title_page(doc: Document, order_data: dict, lang: str):
    section = doc.sections[0]
    section.top_margin = Cm(2)
    section.bottom_margin = Cm(2)
    section.left_margin = Cm(3)
    section.right_margin = Cm(1.5)
    
    ministry = doc.add_paragraph()
    ministry.alignment = WD_ALIGN_PARAGRAPH.CENTER
    ministry_text = "МИНИСТЕРСТВО ОБРАЗОВАНИЯ РЕСПУБЛИКИ БЕЛАРУСЬ" if order_data["country"] == "BY" else "МИНИСТЕРСТВО НАУКИ И ВЫСШЕГО ОБРАЗОВАНИЯ РОССИЙСКОЙ ФЕДЕРАЦИИ"
    run = ministry.add_run(ministry_text)
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    
    uni = doc.add_paragraph()
    uni.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = uni.add_run(order_data["university"].upper())
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    
    faculty = doc.add_paragraph()
    faculty.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = faculty.add_run(order_data["faculty"])
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    
    for _ in range(4):
        doc.add_paragraph()
    
    work_type_name = WORK_TYPES[order_data["work_type"]]["ru"]
    wt = doc.add_paragraph()
    wt.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = wt.add_run(work_type_name.upper())
    run.font.size = Pt(16)
    run.font.name = 'Times New Roman'
    run.bold = True
    
    subj = doc.add_paragraph()
    subj.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = subj.add_run(f"по дисциплине «{order_data['subject']}»")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    
    doc.add_paragraph()
    
    topic_p = doc.add_paragraph()
    topic_p.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = topic_p.add_run(f"на тему: «{order_data['topic']}»")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    
    for _ in range(4):
        doc.add_paragraph()
    
    student_info = doc.add_paragraph()
    student_info.alignment = WD_ALIGN_PARAGRAPH.RIGHT
    text = f"""Выполнил(а):
студент(ка) {order_data['course']} курса
группы {order_data['group']}
{order_data['fullname']}

Проверил(а):
{order_data['teacher']}"""
    run = student_info.add_run(text)
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    
    for _ in range(4):
        doc.add_paragraph()
    
    city_year = doc.add_paragraph()
    city_year.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = city_year.add_run(f"{order_data['city']}, {datetime.now().year}")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    
    doc.add_page_break()

def create_zadanie_page(doc: Document, order_data: dict):
    if order_data.get("zadanie_photo"):
        header = doc.add_paragraph()
        header.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = header.add_run("ЗАДАНИЕ")
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
        run.bold = True
        
        doc.add_paragraph()
        
        note = doc.add_paragraph()
        note.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = note.add_run("(см. приложение)")
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
        run.italic = True
        
        doc.add_page_break()

def generate_chapter_title(chapter_num: int, topic: str) -> str:
    titles = [
        ["Теоретические основы", "Основные понятия", "Общие положения"],
        ["Анализ проблемы", "Практические аспекты", "Современное состояние"],
        ["Перспективы развития", "Рекомендации", "Пути решения"]
    ]
    return titles[chapter_num][0] if chapter_num < len(titles) else f"Глава {chapter_num + 1}"

def generate_subsection_title(content: str, chapter: int, subsection: int) -> str:
    words = re.findall(r'\b[А-ЯЁ][а-яё]{4,}\b', content)
    if words and len(words) >= 2:
        return f"{words[0]} {words[1].lower()}"
    return f"Подраздел {chapter+1}.{subsection+1}"

def generate_references(order_data: dict, count: int) -> list:
    references = []
    current_year = datetime.now().year
    
    authors = ["Иванов И.И.", "Петров П.П.", "Сидоров С.С.", "Козлов К.К.", "Новиков Н.Н.", "Морозов М.М."]
    publishers_by = ["Вышэйшая школа", "БГУ", "БГУИР"]
    publishers_ru = ["Наука", "Юрайт", "ИНФРА-М"]
    cities_by = ["Минск", "Гомель", "Брест"]
    cities_ru = ["Москва", "Санкт-Петербург"]
    
    country = order_data["country"]
    cities = cities_by if country == "BY" else cities_ru
    publishers = publishers_by if country == "BY" else publishers_ru
    
    for i in range(count):
        author = random.choice(authors)
        publisher = random.choice(publishers)
        city = random.choice(cities)
        year = random.randint(current_year - 8, current_year - 1)
        pages = random.randint(120, 450)
        
        topic_words = order_data["topic"].split()[:3]
        title = " ".join(topic_words) if topic_words else order_data["subject"]
        
        ref = f"{author} {title} / {author}. – {city}: {publisher}, {year}. – {pages} с."
        references.append(ref)
    
    return references

def parse_content_structure(content: str, pages: int, order_data: dict) -> dict:
    structure = {"introduction": "", "chapters": [], "conclusion": "", "references": []}
    
    paragraphs = [p.strip() for p in content.split('\n\n') if p.strip()]
    
    intro_paras = max(2, pages // 10)
    conclusion_paras = max(2, pages // 10)
    chapter_paras = len(paragraphs) - intro_paras - conclusion_paras
    
    num_chapters = 2 if pages < 30 else 3
    paras_per_chapter = chapter_paras // num_chapters
    
    structure["introduction"] = "\n\n".join(paragraphs[:intro_paras])
    
    current_pos = intro_paras
    for i in range(num_chapters):
        num_subsections = random.randint(2, 4)
        subsection_size = paras_per_chapter // num_subsections
        
        subsections = []
        for j in range(num_subsections):
            start = current_pos + (j * subsection_size)
            end = start + subsection_size
            subsection_text = "\n\n".join(paragraphs[start:end])
            
            if subsection_text:
                subsections.append({
                    "number": f"{i+1}.{j+1}",
                    "title": generate_subsection_title(subsection_text, i, j),
                    "content": subsection_text
                })
        
        structure["chapters"].append({
            "number": i + 1,
            "title": generate_chapter_title(i, order_data.get("topic", "Тема")),
            "subsections": subsections
        })
        
        current_pos += paras_per_chapter
    
    structure["conclusion"] = "\n\n".join(paragraphs[current_pos:current_pos + conclusion_paras])
    structure["references"] = generate_references(order_data, random.randint(6, 13))
    
    return structure

def create_document(order_data: dict, content: str, lang: str) -> BytesIO:
    doc = Document()
    
    for section in doc.sections:
        section.top_margin = Cm(2)
        section.bottom_margin = Cm(2)
        section.left_margin = Cm(3)
        section.right_margin = Cm(1.5)
    
    create_title_page(doc, order_data, lang)
    
    if order_data.get("zadanie_photo"):
        create_zadanie_page(doc, order_data)
    
    structure = parse_content_structure(content, order_data["pages"], order_data)
    
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    
    toc_entries = [("ВВЕДЕНИЕ", 3)]
    page_num = 4
    
    for chapter in structure["chapters"]:
        toc_entries.append((f"ГЛАВА {chapter['number']} {chapter['title'].upper()}", page_num))
        page_num += 1
        for subsection in chapter["subsections"]:
            toc_entries.append((f"{subsection['number']} {subsection['title']}", page_num))
            page_num += 1
    
    toc_entries.append(("ЗАКЛЮЧЕНИЕ", page_num))
    page_num += 1
    toc_entries.append(("СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ", page_num))
    
    for title, page in toc_entries:
        p = doc.add_paragraph()
        is_main = title.isupper() or title.startswith("ГЛАВА")
        
        if is_main:
            p.alignment = WD_ALIGN_PARAGRAPH.CENTER
            run = p.add_run(title)
            run.font.bold = True
        else:
            p.paragraph_format.left_indent = Cm(1.25)
            p.alignment = WD_ALIGN_PARAGRAPH.LEFT
            dots_count = 80 - len(title) - len(str(page))
            full_text = f"{title}{'.' * dots_count}{page}"
            run = p.add_run(full_text)
        
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
        p.paragraph_format.space_after = Pt(6)
    
    doc.add_page_break()
    
    intro_header = doc.add_paragraph()
    intro_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = intro_header.add_run("ВВЕДЕНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    intro_header.paragraph_format.space_before = Pt(18)
    intro_header.paragraph_format.space_after = Pt(18)
    
    intro_paragraphs = structure["introduction"].split('\n\n')
    for para_text in intro_paragraphs:
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            clean_text = re.sub(r'[#\*_]', '', para_text.strip())
            run = p.add_run(clean_text)
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    
    for chapter in structure["chapters"]:
        ch_header = doc.add_paragraph()
        ch_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = ch_header.add_run(f"ГЛАВА {chapter['number']} {chapter['title'].upper()}")
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
        run.bold = True
        ch_header.paragraph_format.space_before = Pt(18)
        ch_header.paragraph_format.space_after = Pt(18)
        
        doc.add_page_break()
        
        for subsection in chapter["subsections"]:
            sub_header = doc.add_paragraph()
            sub_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
            run = sub_header.add_run(f"{subsection['number']} {subsection['title']}")
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
            run.bold = True
            sub_header.paragraph_format.space_before = Pt(18)
            sub_header.paragraph_format.space_after = Pt(18)
            
            sub_paragraphs = subsection["content"].split('\n\n')
            for para_text in sub_paragraphs:
                if para_text.strip():
                    p = doc.add_paragraph()
                    p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
                    p.paragraph_format.first_line_indent = Cm(1.25)
                    p.paragraph_format.line_spacing = Pt(18)
                    clean_text = re.sub(r'[#\*_]', '', para_text.strip())
                    run = p.add_run(clean_text)
                    run.font.size = Pt(14)
                    run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    concl_header = doc.add_paragraph()
    concl_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = concl_header.add_run("ЗАКЛЮЧЕНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    concl_header.paragraph_format.space_before = Pt(18)
    concl_header.paragraph_format.space_after = Pt(18)
    
    concl_paragraphs = structure["conclusion"].split('\n\n')
    for para_text in concl_paragraphs:
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            clean_text = re.sub(r'[#\*_]', '', para_text.strip())
            run = p.add_run(clean_text)
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    ref_header = doc.add_paragraph()
    ref_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = ref_header.add_run("СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    ref_header.paragraph_format.space_before = Pt(18)
    ref_header.paragraph_format.space_after = Pt(18)
    
    for i, ref in enumerate(structure["references"], 1):
        p = doc.add_paragraph()
        p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
        p.paragraph_format.left_indent = Cm(1.25)
        p.paragraph_format.first_line_indent = Cm(-1.25)
        p.paragraph_format.line_spacing = Pt(18)
        run = p.add_run(f"{i}. {ref}")
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
    
    buffer = BytesIO()
    doc.save(buffer)
    buffer.seek(0)
    return buffer

# ============== PRESENTATION ==============

PRESENTATION_THEMES = [
    {"name": "Modern Blue", "bg": "0a1929", "title": "90caf9", "text": "e3f2fd", "accent": "42a5f5"},
    {"name": "Corporate Red", "bg": "1a1a2e", "title": "ff6b6b", "text": "f8f9fa", "accent": "ee5a6f"},
    {"name": "Nature Green", "bg": "1b4332", "title": "95d5b2", "text": "d8f3dc", "accent": "52b788"},
    {"name": "Royal Purple", "bg": "2d1b69", "title": "b794f6", "text": "e9d8fd", "accent": "9f7aea"},
    {"name": "Ocean Teal", "bg": "004d61", "title": "4dd0e1", "text": "e0f7fa", "accent": "00acc1"}
]

def hex_to_rgb(hex_color: str):
    hex_color = hex_color.lstrip('#')
    return RGBColor(int(hex_color[0:2], 16), int(hex_color[2:4], 16), int(hex_color[4:6], 16))

def search_images(query: str, num_images: int = 10) -> list:
    """✅ Pixabay-dan birnäçe surat netijesini alýar"""
    try:
        clean_query = re.sub(r'[^a-zA-Z\s]', '', query).strip()
        
        params = {
            'key': PIXABAY_API_KEY,
            'q': clean_query,
            'image_type': 'photo',
            'orientation': 'horizontal',
            'safesearch': 'true',
            'per_page': 20, # ✅ Has köp netije soraýarys (20 sany)
            'lang': 'en'
        }
        
        response = requests.get('https://pixabay.com/api/', params=params, timeout=10)
        
        if response.status_code == 200:
            hits = response.json().get('hits', [])
            if hits:
                # Suratlaryň URL-lerini sanaw hökmünde yzyna berýäris
                return [h['largeImageURL'] for h in hits if h['imageWidth'] > 1000]
        
        return []
    except Exception as e:
        logger.error(f"❌ Image search error: {e}")
        return []

def download_image(url: str):
    """✅ Suraty internetden göçürip alýar we BytesIO görnüşinde gaýtarýar"""
    try:
        # Brauzer ýaly görünmek üçin header goşýarys
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.0.0 Safari/537.36"
        }
        response = requests.get(url, headers=headers, timeout=10)
        if response.status_code == 200 and len(response.content) > 10000: # 10KB-dan uly bolmaly
            return BytesIO(response.content)
    except Exception as e:
        logger.error(f"❌ Surat göçürmekde ýalňyşlyk ({url[:30]}...): {e}")
    return None

def get_unique_image(query: str):
    """✅ Internetden täze we ulanylmadyk suraty tapyp getirýär"""
    global USED_IMAGE_URLS
    try:
        with DDGS() as ddgs:
            # Gözleg sözüne professional sypatlar goşýarys
            search_query = f"{query} professional photography high resolution"
            logger.info(f"🔎 Gözleg başlandy: {search_query}")
            
            # Gözleg netijeleri (Uly we giň formatly suratlar)
            results = ddgs.images(
                keywords=search_query,
                region="wt-wt",
                safesearch="on",
                size="Large",
                layout="Wide"
            )
            
            # Tapylan suratlaryň içinden täzesini saýlaýarys
            count = 0
            for r in results:
                url = r['image']
                if url not in USED_IMAGE_URLS:
                    image_data = download_image(url)
                    if image_data:
                        USED_IMAGE_URLS.add(url)
                        return image_data
                
                count += 1
                if count > 15: # Ilkinji 15 suraty barlap görýäris
                    break
    except Exception as e:
        logger.error(f"❌ Gözleg ulgamynda ýalňyşlyk: {e}")
    
    return None

def parse_presentation_content(content: str, num_slides: int) -> list:
    """✅ Slaýdyň adyny we punktlaryny dogry bölüp alýar"""
    slides = [{"type": "title"}]
    
    # Slaýdlary bölmek
    raw_slides = [s.strip() for s in content.split('\n\n') if len(s.strip()) > 50]
    
    for slide_text in raw_slides:
        lines = [l.strip() for l in slide_text.split('\n') if l.strip()]
        if not lines: continue

        # 🔍 IMAGE_KEYWORD-y gözlemek we arassalamak
        img_keyword = "business professional"
        filtered_lines = []
        for l in lines:
            if "IMAGE_KEYWORD:" in l:
                img_keyword = l.split("IMAGE_KEYWORD:")[1].strip().replace('"', '')
            else:
                filtered_lines.append(l)

        if not filtered_lines: continue

        # ✅ BIZIN DÜZELDIŞIMIZ:
        # Birinji setiri Title hökmünde alýarys, galanlary Bullets
        title = filtered_lines[0].replace('#', '').strip()
        bullets = [p.replace('• ', '').replace('-', '').strip() for p in filtered_lines[1:] if len(p) > 5]
        
        # Eger hiç hili punkt ýok bolsa, birinji setiri bullet edip, title-y boş goýýarys
        if not bullets and len(filtered_lines) > 0:
            bullets = [title]
            title = ""

        slides.append({
            "type": "content",
            "title": title,         # Indi title boş däl
            "bullets": bullets[:5],
            "search_query": img_keyword
        })
            
    slides.append({"type": "final"})
    return slides

def create_title_slide(slide, order_data: dict, theme: dict):
    """✅ Title slide - STAYS THE SAME"""
    
    # ✅ TOPIC (main title)
    title_box = slide.shapes.add_textbox(PptxInches(0.5), PptxInches(2), PptxInches(12.333), PptxInches(1.5))
    tf = title_box.text_frame
    tf.word_wrap = True
    p = tf.paragraphs[0]
    p.text = order_data['topic'].upper()
    p.font.size = PptxPt(44)
    p.font.bold = True
    p.font.color.rgb = hex_to_rgb(theme["title"])
    p.alignment = PP_ALIGN.CENTER
    
    # ✅ WORK TYPE
    work_type_name = WORK_TYPES[order_data["work_type"]]["ru"]
    sub_box = slide.shapes.add_textbox(PptxInches(0.5), PptxInches(3.5), PptxInches(12.333), PptxInches(0.5))
    tf = sub_box.text_frame
    p = tf.paragraphs[0]
    p.text = work_type_name
    p.font.size = PptxPt(28)
    p.font.color.rgb = hex_to_rgb(theme["accent"])
    p.alignment = PP_ALIGN.CENTER
    
    # ✅ AUTHOR INFO
    author_box = slide.shapes.add_textbox(PptxInches(0.5), PptxInches(5.5), PptxInches(12.333), PptxInches(1))
    tf = author_box.text_frame
    p = tf.paragraphs[0]
    p.text = f"Выполнил(а): {order_data['fullname']}"
    p.font.size = PptxPt(20)
    p.font.color.rgb = hex_to_rgb(theme["text"])
    p.alignment = PP_ALIGN.CENTER
    
    p = tf.add_paragraph()
    p.text = f"{order_data['university']}, {order_data['city']}, {datetime.now().year}"
    p.font.size = PptxPt(18)
    p.font.color.rgb = hex_to_rgb(theme["text"])
    p.alignment = PP_ALIGN.CENTER

def create_content_slide(slide, slide_data: dict, theme: dict):
    title_box = slide.shapes.add_textbox(PptxInches(0.5), PptxInches(0.3), PptxInches(12.333), PptxInches(1))
    tf = title_box.text_frame
    p = tf.paragraphs[0]
    p.text = slide_data.get("title", "").upper()
    p.font.size = PptxPt(36)
    p.font.bold = True
    p.font.color.rgb = hex_to_rgb(theme["title"])
    p.alignment = PP_ALIGN.CENTER
    
    content_box = slide.shapes.add_textbox(
        PptxInches(1.0),      # Left margin
        PptxInches(1.5),      # Top margin
        PptxInches(11.333),   # Width (almost full)
        PptxInches(5.5)       # Height
    )
    tf = content_box.text_frame
    tf.word_wrap = True
    tf.vertical_anchor = 1  # Center vertically
    
    bullets = slide_data.get("bullets", [])
    
    # ✅ Show all bullets (max 5)
    for i, bullet in enumerate(bullets[:5]):
        if i == 0:
            p = tf.paragraphs[0]
        else:
            p = tf.add_paragraph()
        
        # ✅ Full bullet text
        bullet_text = bullet.strip()
        
        p.text = f"• {bullet_text}"
        p.font.size = PptxPt(24)  # ✅ Bigger font (no title means more space)
        p.font.color.rgb = hex_to_rgb(theme["text"])
        p.space_after = PptxPt(20)  # More spacing
        p.line_spacing = 1.3
        p.alignment = PP_ALIGN.LEFT

def create_content_slide_with_image(slide, slide_data, theme, image_stream):
    """✅ Slaýdyň dizaýny: "Maglumat" sözi aýryldy"""
    from pptx.util import Inches, Pt
    from pptx.dml.color import RGBColor

    # --- 1. Slaýdyň adyny (Title) goýmak ---
    # "Maglumat" sözi aýryldy, diňe slide_data-dan gelýän title ulanylýar
    title_str = slide_data.get("title", "").upper()
    
    if title_str: # Diňe title bar bolsa tekst gutusyny döret
        title_box = slide.shapes.add_textbox(Inches(0.5), Inches(0.3), Inches(12), Inches(1))
        title_frame = title_box.text_frame
        title_p = title_frame.paragraphs[0]
        title_p.text = title_str
        title_p.font.bold = True
        title_p.font.size = Pt(32)
        title_p.font.color.rgb = hex_to_rgb(theme["title"])

    # --- 2. Çep tarapda tekstleri (Bullets) ýerleşdirmek ---
    text_box = slide.shapes.add_textbox(Inches(0.5), Inches(1.5), Inches(6.5), Inches(5))
    text_frame = text_box.text_frame
    text_frame.word_wrap = True

    bullets = slide_data.get("bullets", [])
    for point in bullets:
        p = text_frame.add_paragraph()
        p.text = f"• {point}"
        p.font.size = Pt(20)
        p.font.color.rgb = hex_to_rgb(theme["text"])
        p.space_after = Pt(12)

    # --- 3. Sag tarapda suraty ýerleşdirmek ---
    try:
        picture = slide.shapes.add_picture(image_stream, Inches(7.2), Inches(1.5), width=Inches(5.5), height=Inches(5.0))
        # Surata professional çarçuwa
        picture.line.color.rgb = RGBColor(255, 255, 255)
        picture.line.width = Pt(1)
    except Exception as e:
        logger.error(f"❌ Surat goýup bolmady: {e}")
        text_box.width = Inches(12) # Surat ýok bolsa teksti giňelt

def create_final_slide(slide, theme: dict):
    title_box = slide.shapes.add_textbox(PptxInches(0.5), PptxInches(2.5), PptxInches(12.333), PptxInches(2))
    tf = title_box.text_frame
    p = tf.paragraphs[0]
    p.text = "СПАСИБО ЗА ВНИМАНИЕ!"
    p.font.size = PptxPt(54)
    p.font.bold = True
    p.font.color.rgb = hex_to_rgb(theme["title"])
    p.alignment = PP_ALIGN.CENTER
    
    q_box = slide.shapes.add_textbox(PptxInches(0.5), PptxInches(4.5), PptxInches(12.333), PptxInches(1))
    tf = q_box.text_frame
    p = tf.paragraphs[0]
    p.font.size = PptxPt(32)
    p.font.color.rgb = hex_to_rgb(theme["accent"])
    p.alignment = PP_ALIGN.CENTER

def create_presentation(order_data: dict, content: str) -> BytesIO:
    """✅ Web-den suratly we professional prezentasiýa döretmek"""
    from pptx import Presentation
    from pptx.util import Inches as PptxInches
    
    # Her täze prezentasiýa başlanda ulanylan suratlaryň sanawyny arassalaýarys
    global USED_IMAGE_URLS
    USED_IMAGE_URLS.clear()

    prs = Presentation()
    prs.slide_width = PptxInches(13.333) # 16:9 format
    prs.slide_height = PptxInches(7.5)
    
    theme = random.choice(PRESENTATION_THEMES)
    slides_content = parse_presentation_content(content, order_data['pages'])
    
    total_slides = len(slides_content)
    logger.info(f"🎬 Prezentasiýa döredilýär: {total_slides} slaýd.")

    for idx, slide_data in enumerate(slides_content):
        slide_layout = prs.slide_layouts[6] 
        slide = prs.slides.add_slide(slide_layout)
        
        # Fon reňki
        background = slide.background
        fill = background.fill
        fill.solid()
        fill.fore_color.rgb = hex_to_rgb(theme["bg"])
        
        if idx == 0:
            create_title_slide(slide, order_data, theme)
        elif idx == total_slides - 1:
            create_final_slide(slide, theme)
        else:
            # Slaýdyň temasyna görä surat gözlemek
            search_query = slide_data.get("search_query")
            image_stream = None
            
            if search_query:
                # ✅ Täze gözleg funksiýasyny çagyrýarys
                image_stream = get_unique_image(search_query)

            if image_stream:
                # Surat tapyldy: Suratly slaýd dizaýny
                create_content_slide_with_image(slide, slide_data, theme, image_stream)
                logger.info(f"✅ Slaýd {idx+1}: Web surat goýuldy.")
            else:
                # Surat tapylmady: Diňe tekstli slaýd
                create_content_slide(slide, slide_data, theme)
                logger.warning(f"⚠️ Slaýd {idx+1}: Surat tapylmady, diňe tekst.")

    buffer = BytesIO()
    prs.save(buffer)
    buffer.seek(0)
    return buffer


# ============== AI FUNCTIONS ==============



def extend_content_to_required_pages(content: str, order_data: dict) -> str:
    """✅ Extend content to EXACTLY match ordered pages - SIMPLE & ACCURATE"""
    
    work_type = order_data.get('work_type')
    pages = order_data['pages']
    
    # ✅ CALCULATE WORDS NEEDED
    if work_type == 'referat':
        # Title + TOC + References = 3 pages without content
        # So for 15 pages ordered → need 12 pages of text
        content_pages = max(pages - 3, 1)
        words_per_page = 550
        required_words = content_pages * words_per_page
        
    elif work_type == 'kursovaya':
        # Title + Zadanie + TOC + References = 4 pages without content
        content_pages = max(pages - 4, 1)
        words_per_page = 550
        required_words = content_pages * words_per_page
        
    elif work_type == 'esse':
        # Title only = 1 page without content
        content_pages = max(pages - 1, 1)
        words_per_page = 450
        required_words = content_pages * words_per_page
        
    elif work_type == 'doklad':
        # No title page! All pages are content
        words_per_page = 450
        required_words = pages * words_per_page
        
    elif work_type == 'presentation':
        # Not applicable for presentations
        return content
        
    else:
        # Fallback
        words_per_page = 450
        required_words = pages * words_per_page
    
    current_words = len(content.split())
    
    logger.info(f"📊 {work_type.upper()}: Current={current_words} words | Required={required_words} words | Pages={pages}")
    
    # ✅ CHECK IF OK
    tolerance = 0.1  # 10% tolerance
    min_acceptable = int(required_words * (1 - tolerance))
    max_acceptable = int(required_words * (1 + tolerance))
    
    if min_acceptable <= current_words <= max_acceptable:
        logger.info(f"✅ Content is OK: {current_words} words (range: {min_acceptable}-{max_acceptable})")
        return content
    
    # ✅ TOO SHORT - EXTEND
    if current_words < min_acceptable:
        missing_words = required_words - current_words
        logger.warning(f"⚠️ TOO SHORT by {missing_words} words! Extending...")
        
        topic = order_data['topic']
        subject = order_data['subject']
        
        # ✅ Extensions (each ≈150 words)
        extensions = [
            f"""Детальное рассмотрение проблемы {topic} требует всестороннего анализа теоретических и практических аспектов в контексте {subject}. Современные исследования показывают многогранность данной проблематики и необходимость комплексного междисциплинарного подхода. Систематизация накопленного научного знания и практического опыта позволяет выявить ключевые закономерности и тенденции развития. Важно отметить, что интеграция различных методологических подходов обеспечивает получение более полного и объективного представления об изучаемом явлении.""",
            
            f"""Практическое применение результатов исследований в области {topic} демонстрирует эффективность разработанных теоретических положений и методических рекомендаций в контексте {subject}. Анализ конкретных примеров из практики показывает значительные улучшения при внедрении современных подходов и технологий. Обобщение практического опыта создает основу для формирования стратегий дальнейшего развития и совершенствования существующих методов работы.""",
            
            f"""Методологические аспекты исследования {topic} играют ключевую роль в обеспечении научной строгости и достоверности получаемых результатов в области {subject}. Выбор адекватных методов исследования определяется спецификой изучаемого объекта, поставленными целями и задачами. Комплексное применение различных методов позволяет получить всестороннее представление о проблеме и обеспечить высокое качество исследования.""",
            
            f"""Сравнительный анализ различных подходов к решению проблемы {topic} позволяет выявить преимущества и недостатки каждого метода в контексте {subject}. Систематизация результатов сравнения создает основу для разработки оптимальной стратегии, учитывающей специфику конкретных условий применения. Критическая оценка существующих решений способствует формированию более эффективных подходов.""",
            
            f"""Теоретическое осмысление проблемы {topic} требует анализа фундаментальных концепций и подходов, разработанных в рамках {subject}. Изучение классических и современных работ позволяет проследить эволюцию научных представлений и выявить основные тенденции развития. Критический анализ теоретических построений способствует совершенствованию научного знания и формированию более адекватных представлений.""",
            
            f"""Перспективы дальнейшего развития исследований в области {topic} связаны с применением инновационных методологических подходов в контексте {subject}. Внедрение современных технологий анализа открывает новые возможности для углубленного изучения проблемы. Развитие исследовательской базы способствует повышению качества и достоверности получаемых результатов."""
        ]
        
        # ✅ Add needed extensions
        paragraphs_needed = (missing_words // 150) + 1
        added_extensions = []
        
        for i in range(min(paragraphs_needed, len(extensions) * 5)):
            ext_index = i % len(extensions)
            added_extensions.append(extensions[ext_index])
        
        extended_content = content + "\n\n" + "\n\n".join(added_extensions)
        
        final_words = len(extended_content.split())
        logger.info(f"✅ Extended from {current_words} to {final_words} words")
        
        return extended_content
    
    # ✅ TOO LONG - TRIM
    else:
        logger.warning(f"⚠️ TOO LONG by {current_words - max_acceptable} words! Trimming...")
        
        paragraphs = content.split('\n\n')
        # Calculate percentage to keep
        keep_ratio = required_words / current_words
        target_para_count = int(len(paragraphs) * keep_ratio)
        
        trimmed_content = '\n\n'.join(paragraphs[:target_para_count])
        
        final_words = len(trimmed_content.split())
        logger.info(f"✅ Trimmed from {current_words} to {final_words} words")
        
        return trimmed_content


def add_table_to_docx(doc, table_text):
    """Markdown tablisasyny professional 14 Pt Word tablisasyna öwürýär"""
    try:
        raw_lines = table_text.strip().split('\n')
        lines = []
        for line in raw_lines:
            if '|' in line:
                if re.search(r'^[|\s:-]+$', line): continue
                cols = [c.strip() for c in line.split('|') if c.strip()]
                if cols: lines.append(cols)

        if len(lines) < 2: return

        num_rows = len(lines)
        num_cols = max(len(row) for row in lines)
        table = doc.add_table(rows=num_rows, cols=num_cols)
        table.style = 'Table Grid'
        table.alignment = WD_ALIGN_PARAGRAPH.CENTER

        for row_idx, row_data in enumerate(lines):
            row_cells = table.rows[row_idx].cells
            for col_idx, cell_value in enumerate(row_data):
                if col_idx < num_cols:
                    cell = row_cells[col_idx]
                    p = cell.paragraphs[0]
                    p.alignment = WD_ALIGN_PARAGRAPH.CENTER
                    run = p.add_run(cell_value)
                    run.font.name = 'Times New Roman'
                    run.font.size = Pt(14) # ✅ Tablisa 14 Pt

                    if row_idx == 0: # Sözbaşy bezegi
                        run.bold = True
                        shading_elm = parse_xml(r'<w:shd {} w:fill="E7E6E6"/>'.format(nsdecls('w')))
                        cell._element.get_or_add_tcPr().append(shading_elm)
        doc.add_paragraph()
    except Exception as e:
        logger.error(f"Tablisa hatasy: {e}")

def insert_smart_content(doc, content):
    """Ähli tekstleri we media elementleri 14 Pt Times New Roman görnüşinde ýazýar"""
    parts = re.split(r'(\[IMAGE:.*?\]|\[SCHEMA:.*?\]|(?:\n|^)\|.*?\|.*?\|(?:\n|$))', content, flags=re.DOTALL)
    for part in parts:
        part = part.strip()
        if not part: continue
        if part.startswith('[IMAGE:'):
            q = part.replace('[IMAGE:', '').replace(']', '').strip()
            add_image_to_docx(doc, q)
        elif part.startswith('[SCHEMA:'):
            s = part.replace('[SCHEMA:', '').replace(']', '').strip()
            add_schema_placeholder(doc, s)
        elif '|' in part and '-' in part:
            add_table_to_docx(doc, part)
        else:
            paragraphs = part.split('\n')
            for para_text in paragraphs:
                if para_text.strip():
                    p = doc.add_paragraph()
                    p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
                    p.paragraph_format.first_line_indent = Cm(1.25)
                    p.paragraph_format.line_spacing = Pt(18)
                    run = p.add_run(re.sub(r'[#\*_]', '', para_text.strip()))
                    run.font.size = Pt(14) # ✅ Adaty tekst 14 Pt
                    run.font.name = 'Times New Roman'

def add_image_to_docx(doc, query):
    """Internetden surat tapyp Word-a goşýar"""
    image_stream = get_unique_image(query) # Siziň öňki funksiýaňyz
    if image_stream:
        try:
            doc.add_picture(image_stream, width=Inches(5.5))
            last_p = doc.paragraphs[-1]
            last_p.alignment = WD_ALIGN_PARAGRAPH.CENTER
            # Suratyň aşagyna düşündiriş
            caption = doc.add_paragraph(f"Рисунок — {query}")
            caption.alignment = WD_ALIGN_PARAGRAPH.CENTER
            caption.font.italic = True
        except:
            pass

def add_schema_placeholder(doc, schema_desc):
    """Shemany owadan ramka we tekst hökmünde goşýar"""
    table = doc.add_table(rows=1, cols=1)
    table.style = 'Light Shading Accent 1'
    cell = table.rows[0].cells[0]
    p = cell.paragraphs[0]
    p.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = p.add_run(f"ЛОГИЧЕСКАЯ СХЕМА:\n{schema_desc}")
    run.bold = True
    run.font.size = Pt(14)
    doc.add_paragraph()

def insert_smart_content(doc, content):
    """Teksti parse edip, media we 14 Pt tekstleri goşýar"""
    parts = re.split(r'(\[IMAGE:.*?\]|\[SCHEMA:.*?\]|(?:\n|^)\|.*?\|.*?\|(?:\n|$))', content, flags=re.DOTALL)

    for part in parts:
        part = part.strip()
        if not part: continue

        if part.startswith('[IMAGE:'):
            q = part.replace('[IMAGE:', '').replace(']', '').strip()
            add_image_to_docx(doc, q)
        elif part.startswith('[SCHEMA:'):
            s = part.replace('[SCHEMA:', '').replace(']', '').strip()
            add_schema_placeholder(doc, s)
        elif '|' in part and '-' in part:
            add_table_to_docx(doc, part)
        else:
            paragraphs = part.split('\n')
            for para_text in paragraphs:
                if para_text.strip():
                    p = doc.add_paragraph()
                    p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
                    p.paragraph_format.first_line_indent = Cm(1.25)
                    p.paragraph_format.line_spacing = Pt(18)
                    clean_text = re.sub(r'[#\*_]', '', para_text.strip())
                    run = p.add_run(clean_text)
                    run.font.size = Pt(14)  # ✅ Tekst 14 Pt
                    run.font.name = 'Times New Roman'

def add_page_numbers_referat(doc: Document):
    from docx.oxml import OxmlElement
    from docx.oxml.ns import qn
    
    sections = doc.sections
    first_section = sections[0]
    first_section.different_first_page_header_footer = True
    
    for section in sections:
        footer = section.footer
        
        for para in footer.paragraphs:
            para.clear()
        
        footer_para = footer.paragraphs[0] if footer.paragraphs else footer.add_paragraph()
        footer_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
        
        run = footer_para.add_run()
        
        fldChar1 = OxmlElement('w:fldChar')
        fldChar1.set(qn('w:fldCharType'), 'begin')
        
        instrText = OxmlElement('w:instrText')
        instrText.set(qn('xml:space'), 'preserve')
        instrText.text = 'PAGE'
        
        fldChar2 = OxmlElement('w:fldChar')
        fldChar2.set(qn('w:fldCharType'), 'end')
        
        run._r.append(fldChar1)
        run._r.append(instrText)
        run._r.append(fldChar2)
        
        run.font.size = Pt(12)
        run.font.name = 'Times New Roman'

def parse_content_structure_referat(content: str, pages: int, order_data: dict) -> dict:
    structure = {"introduction": "", "chapters": [], "conclusion": "", "references": []}
    
    paragraphs = [p.strip() for p in content.split('\n\n') if p.strip()]
    total_paras = len(paragraphs)
    
    intro_count = max(3, int(total_paras * 0.12))
    chapter1_count = int(total_paras * 0.35)
    chapter2_count = int(total_paras * 0.35)
    conclusion_count = max(3, int(total_paras * 0.12))
    
    structure["introduction"] = "\n\n".join(paragraphs[:intro_count])
    
    current_pos = intro_count
    
    chapter1_paras = paragraphs[current_pos:current_pos + chapter1_count]
    subsection_size = len(chapter1_paras) // 4
    
    subsections1 = []
    for j in range(4):
        start = j * subsection_size
        end = start + subsection_size if j < 3 else len(chapter1_paras)
        subsection_text = "\n\n".join(chapter1_paras[start:end])
        
        if subsection_text:
            titles1 = ["Основные понятия и определения", "Исторический аспект", "Классификация и виды", "Современные подходы"]
            subsections1.append({
                "number": f"1.{j+1}",
                "title": titles1[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 1,
        "title": "Теоретические основы",
        "subsections": subsections1
    })
    
    current_pos += chapter1_count
    
    chapter2_paras = paragraphs[current_pos:current_pos + chapter2_count]
    subsection_size2 = len(chapter2_paras) // 4
    
    subsections2 = []
    for j in range(4):
        start = j * subsection_size2
        end = start + subsection_size2 if j < 3 else len(chapter2_paras)
        subsection_text = "\n\n".join(chapter2_paras[start:end])
        
        if subsection_text:
            titles2 = ["Текущее состояние проблемы", "Анализ существующих решений", "Сравнительный анализ", "Примеры из практики"]
            subsections2.append({
                "number": f"2.{j+1}",
                "title": titles2[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 2,
        "title": "Практический анализ",
        "subsections": subsections2
    })
    
    current_pos += chapter2_count
    
    structure["conclusion"] = "\n\n".join(paragraphs[current_pos:current_pos + conclusion_count])
    structure["references"] = generate_references(order_data, random.randint(8, 12))
    
    return structure

def calculate_actual_page_numbers(structure: dict, order_data: dict) -> dict:
    page_map = {}
    current_page = 1
    
    current_page += 1
    
    if order_data.get("zadanie_photo"):
        current_page += 1
    
    page_map["toc"] = current_page
    current_page += 1
    
    page_map["introduction"] = current_page
    intro_words = len(structure["introduction"].split())
    intro_pages = max(1, intro_words // 400)
    current_page += intro_pages
    
    page_map["chapters"] = {}
    for chapter in structure["chapters"]:
        chapter_num = chapter["number"]
        page_map["chapters"][chapter_num] = current_page
        
        chapter_words = 0
        for subsection in chapter["subsections"]:
            chapter_words += len(subsection["content"].split())
        
        chapter_pages = max(1, chapter_words // 400)
        current_page += chapter_pages
    
    page_map["conclusion"] = current_page
    concl_words = len(structure["conclusion"].split())
    concl_pages = max(1, concl_words // 400)
    current_page += concl_pages
    
    page_map["references"] = current_page
    
    return page_map

def create_toc_referat(doc: Document, structure: dict, order_data: dict):
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    toc_header.paragraph_format.space_before = Pt(0)
    
    doc.add_paragraph()
    
    page_map = calculate_actual_page_numbers(structure, order_data)
    
    toc_entries = []
    
    toc_entries.append(("ВВЕДЕНИЕ", page_map["introduction"], False))
    
    for chapter in structure["chapters"]:
        chapter_num = chapter["number"]
        page_num = page_map["chapters"][chapter_num]
        
        toc_entries.append((f"ГЛАВА {chapter_num} {chapter['title'].upper()}", page_num, False))
        
        for subsection in chapter["subsections"]:
            toc_entries.append((f"{subsection['number']} {subsection['title']}", page_num, True))
    
    toc_entries.append(("ЗАКЛЮЧЕНИЕ", page_map["conclusion"], False))
    toc_entries.append(("СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ", page_map["references"], False))
    
    for title, page, is_subsection in toc_entries:
        p = doc.add_paragraph()
        p.paragraph_format.space_after = Pt(0)
        p.paragraph_format.line_spacing = Pt(18)
        
        if is_subsection:
            p.paragraph_format.left_indent = Cm(1.25)
        
        from docx.oxml import OxmlElement
        from docx.oxml.ns import qn
        
        tab_stops_element = p._element.get_or_add_pPr().get_or_add_tabs()
        tab_stop = OxmlElement('w:tab')
        tab_stop.set(qn('w:val'), 'right')
        tab_stop.set(qn('w:leader'), 'dot')
        tab_stop.set(qn('w:pos'), str(int(Cm(16.5).twips)))
        tab_stops_element.append(tab_stop)
        
        run_title = p.add_run(title)
        run_title.font.size = Pt(14)
        run_title.font.name = 'Times New Roman'
        
        if not is_subsection:
            run_title.font.bold = True
        
        p.add_run('\t')
        
        run_page = p.add_run(str(page))
        run_page.font.size = Pt(14)
        run_page.font.name = 'Times New Roman'
        
        if not is_subsection:
            run_page.font.bold = True

def create_document_referat(order_data: dict, content: str, lang: str) -> BytesIO:
    doc = Document()
    create_title_page(doc, order_data, lang)
    if order_data.get("zadanie_photo"): create_zadanie_page(doc, order_data)

    content = extend_content_to_required_pages(content, order_data)
    structure = parse_content_structure_referat(content, order_data["pages"], order_data)
    create_toc_referat(doc, structure, order_data)
    doc.add_page_break()

    # ВВЕДЕНИЕ
    intro_h = doc.add_paragraph()
    intro_h.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = intro_h.add_run("ВВЕДЕНИЕ")
    run.font.size = Pt(14) # ✅ 14 Pt
    run.font.name = 'Times New Roman'
    run.bold = True
    insert_smart_content(doc, structure["introduction"])
    doc.add_page_break()

    for chapter in structure["chapters"]:
        ch_header = doc.add_paragraph()
        ch_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = ch_header.add_run(f"ГЛАВА {chapter['number']} {chapter['title'].upper()}")
        run.font.size = Pt(14) # ✅ 14 Pt
        run.font.name = 'Times New Roman'
        run.bold = True
        
        for subsection in chapter["subsections"]:
            sub_h = doc.add_paragraph()
            sub_h.paragraph_format.left_indent = Cm(1.25)
            run = sub_h.add_run(f"{subsection['number']} {subsection['title']}")
            run.font.size = Pt(14) # ✅ 14 Pt
            run.font.name = 'Times New Roman'
            run.bold = True
            insert_smart_content(doc, subsection["content"])
        doc.add_page_break()

    concl_h = doc.add_paragraph()
    concl_h.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = concl_h.add_run("ЗАКЛЮЧЕНИЕ")
    run.font.size = Pt(14) # ✅ 14 Pt
    run.font.name = 'Times New Roman'
    run.bold = True
    insert_smart_content(doc, structure["conclusion"])
    
    # Список источников (Eýýäm kodyňyzda bar, şol galybermeli)
    # ...
    add_page_numbers_referat(doc)
    buffer = BytesIO(); doc.save(buffer); buffer.seek(0)
    return buffer

# ============== ESSE FUNCTIONS ==============



def parse_content_structure_esse(content: str, pages: int, order_data: dict) -> dict:
    """✅ ESSE structure"""
    structure = {"introduction": "", "main_part": "", "conclusion": "", "references": []}
    
    paragraphs = [p.strip() for p in content.split('\n\n') if p.strip()]
    total_paras = len(paragraphs)
    
    intro_count = max(2, int(total_paras * 0.15))
    main_count = int(total_paras * 0.70)
    conclusion_count = max(2, int(total_paras * 0.15))
    
    structure["introduction"] = "\n\n".join(paragraphs[:intro_count])
    structure["main_part"] = "\n\n".join(paragraphs[intro_count:intro_count + main_count])
    structure["conclusion"] = "\n\n".join(paragraphs[intro_count + main_count:intro_count + main_count + conclusion_count])
    structure["references"] = generate_references(order_data, random.randint(5, 8))
    
    return structure


def create_toc_esse(doc: Document, structure: dict, order_data: dict):
    """✅ TOC for ESSE"""
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    
    doc.add_paragraph()
    
    current_page = 2
    toc_page = current_page
    current_page += 1
    
    intro_page = current_page
    intro_words = len(structure["introduction"].split())
    current_page += max(1, intro_words // 400)
    
    main_page = current_page
    main_words = len(structure["main_part"].split())
    current_page += max(1, main_words // 400)
    
    conclusion_page = current_page
    concl_words = len(structure["conclusion"].split())
    current_page += max(1, concl_words // 400)
    
    ref_page = current_page
    
    toc_entries = [
        ("ВВЕДЕНИЕ", intro_page),
        ("ОСНОВНАЯ ЧАСТЬ", main_page),
        ("ЗАКЛЮЧЕНИЕ", conclusion_page),
        ("СПИСОК ЛИТЕРАТУРЫ", ref_page)
    ]
    
    from docx.oxml import OxmlElement
    from docx.oxml.ns import qn
    
    for title, page in toc_entries:
        p = doc.add_paragraph()
        p.paragraph_format.space_after = Pt(0)
        p.paragraph_format.line_spacing = Pt(18)
        
        tab_stops_element = p._element.get_or_add_pPr().get_or_add_tabs()
        tab_stop = OxmlElement('w:tab')
        tab_stop.set(qn('w:val'), 'right')
        tab_stop.set(qn('w:leader'), 'dot')
        tab_stop.set(qn('w:pos'), str(int(Cm(16.5).twips)))
        tab_stops_element.append(tab_stop)
        
        run_title = p.add_run(title)
        run_title.font.size = Pt(14)
        run_title.font.name = 'Times New Roman'
        run_title.font.bold = True
        
        p.add_run('\t')
        
        run_page = p.add_run(str(page))
        run_page.font.size = Pt(14)
        run_page.font.name = 'Times New Roman'
        run_page.font.bold = True


def create_document_esse(order_data: dict, content: str, lang: str) -> BytesIO:
    """✅ ESSE document"""
    doc = Document()
    
    for section in doc.sections:
        section.top_margin = Cm(2)
        section.bottom_margin = Cm(2)
        section.left_margin = Cm(3)
        section.right_margin = Cm(1.5)
    
    create_title_page(doc, order_data, lang)
    content = extend_content_to_required_pages(content, order_data)
    structure = parse_content_structure_esse(content, order_data["pages"], order_data)
    
    create_toc_esse(doc, structure, order_data)
    doc.add_page_break()
    
    # ВВЕДЕНИЕ
    intro_header = doc.add_paragraph()
    intro_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = intro_header.add_run("ВВЕДЕНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    intro_header.paragraph_format.space_after = Pt(18)
    
    for para_text in structure["introduction"].split('\n\n'):
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            run = p.add_run(re.sub(r'[#\*_]', '', para_text.strip()))
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    
    # ОСНОВНАЯ ЧАСТЬ
    main_header = doc.add_paragraph()
    main_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = main_header.add_run("ОСНОВНАЯ ЧАСТЬ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    main_header.paragraph_format.space_after = Pt(18)
    
    for para_text in structure["main_part"].split('\n\n'):
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            run = p.add_run(re.sub(r'[#\*_]', '', para_text.strip()))
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    
    # ЗАКЛЮЧЕНИЕ
    concl_header = doc.add_paragraph()
    concl_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = concl_header.add_run("ЗАКЛЮЧЕНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    concl_header.paragraph_format.space_after = Pt(18)
    
    for para_text in structure["conclusion"].split('\n\n'):
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            run = p.add_run(re.sub(r'[#\*_]', '', para_text.strip()))
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    
    # СПИСОК ЛИТЕРАТУРЫ
    ref_header = doc.add_paragraph()
    ref_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = ref_header.add_run("СПИСОК ЛИТЕРАТУРЫ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    ref_header.paragraph_format.space_after = Pt(18)
    
    for i, ref in enumerate(structure["references"], 1):
        p = doc.add_paragraph()
        p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
        p.paragraph_format.left_indent = Cm(1.25)
        p.paragraph_format.first_line_indent = Cm(-1.25)
        p.paragraph_format.line_spacing = Pt(18)
        run = p.add_run(f"{i}. {ref}")
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
    
    add_page_numbers_referat(doc)
    
    buffer = BytesIO()
    doc.save(buffer)
    buffer.seek(0)
    return buffer


# ============== DOKLAD FUNCTIONS ==============



def parse_content_structure_doklad(content: str, pages: int, order_data: dict) -> dict:
    """✅ DOKLAD - 2 chapters"""
    structure = {"introduction": "", "chapters": [], "conclusion": "", "references": []}
    
    paragraphs = [p.strip() for p in content.split('\n\n') if p.strip()]
    total_paras = len(paragraphs)
    
    intro_count = max(2, int(total_paras * 0.10))
    chapter1_count = int(total_paras * 0.40)
    chapter2_count = int(total_paras * 0.40)
    conclusion_count = max(2, int(total_paras * 0.10))
    
    structure["introduction"] = "\n\n".join(paragraphs[:intro_count])
    
    current_pos = intro_count
    
    chapter1_paras = paragraphs[current_pos:current_pos + chapter1_count]
    structure["chapters"].append({
        "number": 1,
        "title": "Теоретические аспекты",
        "content": "\n\n".join(chapter1_paras)
    })
    
    current_pos += chapter1_count
    
    chapter2_paras = paragraphs[current_pos:current_pos + chapter2_count]
    structure["chapters"].append({
        "number": 2,
        "title": "Практические вопросы",
        "content": "\n\n".join(chapter2_paras)
    })
    
    current_pos += chapter2_count
    
    structure["conclusion"] = "\n\n".join(paragraphs[current_pos:current_pos + conclusion_count])
    structure["references"] = generate_references(order_data, random.randint(6, 10))
    
    return structure


def create_toc_doklad(doc: Document, structure: dict, order_data: dict):
    """✅ TOC for DOKLAD"""
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    
    doc.add_paragraph()
    
    current_page = 2
    current_page += 1
    
    intro_page = current_page
    intro_words = len(structure["introduction"].split())
    current_page += max(1, intro_words // 400)
    
    chapter_pages = {}
    for chapter in structure["chapters"]:
        chapter_pages[chapter["number"]] = current_page
        chapter_words = len(chapter["content"].split())
        current_page += max(1, chapter_words // 400)
    
    conclusion_page = current_page
    current_page += max(1, len(structure["conclusion"].split()) // 400)
    
    ref_page = current_page
    
    toc_entries = [("ВВЕДЕНИЕ", intro_page, False)]
    
    for chapter in structure["chapters"]:
        toc_entries.append((f"{chapter['number']}. {chapter['title'].upper()}", chapter_pages[chapter['number']], False))
    
    toc_entries.append(("ЗАКЛЮЧЕНИЕ", conclusion_page, False))
    toc_entries.append(("СПИСОК ЛИТЕРАТУРЫ", ref_page, False))
    
    from docx.oxml import OxmlElement
    from docx.oxml.ns import qn
    
    for title, page, is_sub in toc_entries:
        p = doc.add_paragraph()
        p.paragraph_format.space_after = Pt(0)
        p.paragraph_format.line_spacing = Pt(18)
        
        tab_stops_element = p._element.get_or_add_pPr().get_or_add_tabs()
        tab_stop = OxmlElement('w:tab')
        tab_stop.set(qn('w:val'), 'right')
        tab_stop.set(qn('w:leader'), 'dot')
        tab_stop.set(qn('w:pos'), str(int(Cm(16.5).twips)))
        tab_stops_element.append(tab_stop)
        
        run_title = p.add_run(title)
        run_title.font.size = Pt(14)
        run_title.font.name = 'Times New Roman'
        run_title.font.bold = True
        
        p.add_run('\t')
        
        run_page = p.add_run(str(page))
        run_page.font.size = Pt(14)
        run_page.font.name = 'Times New Roman'
        run_page.font.bold = True


# ============== KURSOVAYA FUNCTIONS ==============

def generate_content_kursovaya(order_data: dict) -> Optional[str]:
    """✅ Generate KURSOVAYA with GROQ ONLY"""
    
    try:
        logger.info("🚀 Generating with GROQ...")
        url = "https://api.groq.com/openai/v1/chat/completions"
        
        pages = order_data['pages']
        total_words = pages * 500
        
        prompt = f"""Напиши подробную курсовую работу на тему: "{order_data['topic']}"

Предмет: {order_data['subject']}
Университет: {order_data['university']}

СТРУКТУРА (минимум {total_words} слов):

ВВЕДЕНИЕ (10%):
- Актуальность темы
- Цель и задачи работы
- Краткий обзор структуры

ГЛАВА 1. ТЕОРЕТИЧЕСКАЯ ЧАСТЬ (30%):
1.1 Теоретические основы проблемы
1.2 Анализ научной литературы по теме
1.3 Методология исследования

ГЛАВА 2. ПРАКТИЧЕСКАЯ ЧАСТЬ (30%):
2.1 Анализ текущего состояния проблемы
2.2 Выявленные проблемы и их причины
2.3 Предлагаемые пути решения

ГЛАВА 3. РЕЗУЛЬТАТЫ И ВЫВОДЫ (20%):
3.1 Результаты исследования
3.2 Практические рекомендации

ЗАКЛЮЧЕНИЕ (10%):
- Основные выводы
- Достижение поставленной цели
- Практическая значимость

ТРЕБОВАНИЯ:
✅ Минимум {total_words} слов
✅ БЕЗ заголовков в самом тексте
✅ ТОЛЬКО русский язык
✅ БЕЗ списков, только связный текст
✅ Каждый абзац минимум 5-7 предложений
✅ Академический стиль

Начинай писать текст сразу. Разделяй части работы двойным переводом строки."""

        headers = {
            "Authorization": f"Bearer {GROQ_API_KEY}",
            "Content-Type": "application/json"
        }
        
        payload = {
            "model": "llama-3.3-70b-versatile",
            "messages": [{"role": "user", "content": prompt}],
            "temperature": 0.85,
            "max_tokens": 16000,
            "top_p": 0.95
        }
        
        response = requests.post(url, json=payload, headers=headers, timeout=300)
        
        if response.status_code == 200:
            data = response.json()
            content = data["choices"][0]["message"]["content"]
            word_count = len(content.split())
            logger.info(f"✅ Groq KURSOVAYA: {word_count} words")
            return content
        else:
            logger.error(f"❌ Groq error: {response.status_code} - {response.text[:200]}")
            return None
            
    except Exception as e:
        logger.error(f"❌ Groq exception: {e}")
        return None


def parse_content_structure_kursovaya(content: str, pages: int, order_data: dict) -> dict:
    """✅ KURSOVAYA - 3 chapters with subsections"""
    structure = {"introduction": "", "chapters": [], "conclusion": "", "references": []}
    
    paragraphs = [p.strip() for p in content.split('\n\n') if p.strip()]
    total_paras = len(paragraphs)
    
    intro_count = max(3, int(total_paras * 0.10))
    chapter1_count = int(total_paras * 0.30)
    chapter2_count = int(total_paras * 0.30)
    chapter3_count = int(total_paras * 0.20)
    conclusion_count = max(3, int(total_paras * 0.10))
    
    structure["introduction"] = "\n\n".join(paragraphs[:intro_count])
    
    current_pos = intro_count
    
    # CHAPTER 1
    chapter1_paras = paragraphs[current_pos:current_pos + chapter1_count]
    subsection_size1 = len(chapter1_paras) // 3
    
    subsections1 = []
    titles1 = ["Теоретические основы", "Анализ литературы", "Методология"]
    for j in range(3):
        start = j * subsection_size1
        end = start + subsection_size1 if j < 2 else len(chapter1_paras)
        subsection_text = "\n\n".join(chapter1_paras[start:end])
        
        if subsection_text:
            subsections1.append({
                "number": f"1.{j+1}",
                "title": titles1[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 1,
        "title": "Теоретическая часть",
        "subsections": subsections1
    })
    
    current_pos += chapter1_count
    
    # CHAPTER 2
    chapter2_paras = paragraphs[current_pos:current_pos + chapter2_count]
    subsection_size2 = len(chapter2_paras) // 3
    
    subsections2 = []
    titles2 = ["Анализ состояния", "Выявленные проблемы", "Решения"]
    for j in range(3):
        start = j * subsection_size2
        end = start + subsection_size2 if j < 2 else len(chapter2_paras)
        subsection_text = "\n\n".join(chapter2_paras[start:end])
        
        if subsection_text:
            subsections2.append({
                "number": f"2.{j+1}",
                "title": titles2[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 2,
        "title": "Практическая часть",
        "subsections": subsections2
    })
    
    current_pos += chapter2_count
    
    # CHAPTER 3
    chapter3_paras = paragraphs[current_pos:current_pos + chapter3_count]
    subsection_size3 = len(chapter3_paras) // 2
    
    subsections3 = []
    titles3 = ["Результаты", "Рекомендации"]
    for j in range(2):
        start = j * subsection_size3
        end = start + subsection_size3 if j < 1 else len(chapter3_paras)
        subsection_text = "\n\n".join(chapter3_paras[start:end])
        
        if subsection_text:
            subsections3.append({
                "number": f"3.{j+1}",
                "title": titles3[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 3,
        "title": "Результаты",
        "subsections": subsections3
    })
    
    current_pos += chapter3_count
    
    structure["conclusion"] = "\n\n".join(paragraphs[current_pos:current_pos + conclusion_count])
    structure["references"] = generate_references(order_data, random.randint(15, 25))
    
    return structure


def create_toc_kursovaya(doc: Document, structure: dict, order_data: dict):
    """✅ TOC for KURSOVAYA"""
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    
    doc.add_paragraph()
    
    current_page = 2
    
    if order_data.get("zadanie_photo"):
        current_page += 1
    
    current_page += 1
    
    intro_page = current_page
    intro_words = len(structure["introduction"].split())
    current_page += max(1, intro_words // 400)
    
    toc_entries = [("ВВЕДЕНИЕ", intro_page, False)]
    
    for chapter in structure["chapters"]:
        chapter_page = current_page
        toc_entries.append((f"ГЛАВА {chapter['number']}. {chapter['title'].upper()}", chapter_page, False))
        
        for subsection in chapter["subsections"]:
            toc_entries.append((f"{subsection['number']} {subsection['title']}", chapter_page, True))
        
        chapter_words = sum(len(sub["content"].split()) for sub in chapter["subsections"])
        current_page += max(1, chapter_words // 400)
    
    conclusion_page = current_page
    current_page += max(1, len(structure["conclusion"].split()) // 400)
    
    ref_page = current_page
    
    toc_entries.append(("ЗАКЛЮЧЕНИЕ", conclusion_page, False))
    toc_entries.append(("СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ", ref_page, False))
    
    from docx.oxml import OxmlElement
    from docx.oxml.ns import qn
    
    for title, page, is_subsection in toc_entries:
        p = doc.add_paragraph()
        p.paragraph_format.space_after = Pt(0)
        p.paragraph_format.line_spacing = Pt(18)
        
        if is_subsection:
            p.paragraph_format.left_indent = Cm(1.25)
        
        tab_stops_element = p._element.get_or_add_pPr().get_or_add_tabs()
        tab_stop = OxmlElement('w:tab')
        tab_stop.set(qn('w:val'), 'right')
        tab_stop.set(qn('w:leader'), 'dot')
        tab_stop.set(qn('w:pos'), str(int(Cm(16.5).twips)))
        tab_stops_element.append(tab_stop)
        
        run_title = p.add_run(title)
        run_title.font.size = Pt(14)
        run_title.font.name = 'Times New Roman'
        
        if not is_subsection:
            run_title.font.bold = True
        
        p.add_run('\t')
        
        run_page = p.add_run(str(page))
        run_page.font.size = Pt(14)
        run_page.font.name = 'Times New Roman'
        
        if not is_subsection:
            run_page.font.bold = True


def create_document_kursovaya(order_data: dict, content: str, lang: str) -> BytesIO:
    doc = Document()
    create_title_page(doc, order_data, lang)
    if order_data.get("zadanie_photo"): create_zadanie_page(doc, order_data)
    
    content = extend_content_to_required_pages(content, order_data)
    structure = parse_content_structure_kursovaya(content, order_data["pages"], order_data)
    create_toc_kursovaya(doc, structure, order_data)
    doc.add_page_break()

    # --- BÖLÜMLER (14 Pt Bold Headers) ---
    sections = [("introduction", "ВВЕДЕНИЕ")] + \
               [(ch, f"ГЛАВА {ch['number']}. {ch['title'].upper()}") for ch in structure["chapters"]] + \
               [("conclusion", "ЗАКЛЮЧЕНИЕ")]

    for key, title in sections:
        h = doc.add_paragraph()
        h.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = h.add_run(title)
        run.font.size = Pt(14) # ✅ Başlyk 14 Pt
        run.font.name = 'Times New Roman'
        run.bold = True
        
        if key == "introduction":
            insert_smart_content(doc, structure["introduction"])
        elif key == "conclusion":
            insert_smart_content(doc, structure["conclusion"])
        else: # Chapters
            for sub in key["subsections"]:
                sh = doc.add_paragraph()
                sh.paragraph_format.left_indent = Cm(1.25)
                run_sub = sh.add_run(f"{sub['number']} {sub['title']}")
                run_sub.font.size = Pt(14) # ✅ Podrazdel 14 Pt
                run_sub.font.name = 'Times New Roman'
                run_sub.bold = True
                insert_smart_content(doc, sub["content"])
        doc.add_page_break()

    # Список литературы (14 Pt)
    ref_h = doc.add_paragraph()
    ref_h.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = ref_h.add_run("СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ")
    run.font.size = Pt(14); run.font.bold = True; run.font.name = 'Times New Roman'
    for i, ref in enumerate(structure["references"], 1):
        p = doc.add_paragraph()
        run = p.add_run(f"{i}. {ref}")
        run.font.size = Pt(14); run.font.name = 'Times New Roman'
    
    add_page_numbers_referat(doc)
    buffer = BytesIO(); doc.save(buffer); buffer.seek(0)
    return buffer



def parse_content_structure_esse(content: str, pages: int, order_data: dict) -> dict:
    """✅ ESSE - NO intro/conclusion/references structure"""
    structure = {"main_content": content}  # Весь текст - это основная часть
    return structure


def create_document_esse(order_data: dict, content: str, lang: str) -> BytesIO:
    """✅ ESSE document - БЕЗ TOC, БЕЗ введения/заключения/списка"""
    doc = Document()
    
    for section in doc.sections:
        section.top_margin = Cm(2)
        section.bottom_margin = Cm(2)
        section.left_margin = Cm(3)
        section.right_margin = Cm(1.5)
    
    # Title page
    create_title_page(doc, order_data, lang)
    
    # Extend content
    content = extend_content_to_required_pages(content, order_data)
    
    # NO TOC, NO structure parsing
    # Just write the content directly
    
    paragraphs = content.split('\n\n')
    for para_text in paragraphs:
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            p.paragraph_format.space_after = Pt(0)
            
            clean_text = re.sub(r'[#\*_]', '', para_text.strip())
            run = p.add_run(clean_text)
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    # Page numbers
    add_page_numbers_referat(doc)
    
    buffer = BytesIO()
    doc.save(buffer)
    buffer.seek(0)
    return buffer


def create_toc_esse(doc: Document, structure: dict, order_data: dict):
    """✅ TOC for ESSE"""
    
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    toc_header.paragraph_format.space_before = Pt(0)
    
    doc.add_paragraph()
    
    # Calculate pages
    current_page = 2  # After title
    
    toc_page = current_page
    current_page += 1
    
    intro_page = current_page
    intro_words = len(structure["introduction"].split())
    current_page += max(1, intro_words // 400)
    
    main_page = current_page
    main_words = len(structure["main_part"].split())
    current_page += max(1, main_words // 400)
    
    conclusion_page = current_page
    concl_words = len(structure["conclusion"].split())
    current_page += max(1, concl_words // 400)
    
    ref_page = current_page
    
    toc_entries = [
        ("ВВЕДЕНИЕ", intro_page),
        ("ОСНОВНАЯ ЧАСТЬ", main_page),
        ("ЗАКЛЮЧЕНИЕ", conclusion_page),
        ("СПИСОК ЛИТЕРАТУРЫ", ref_page)
    ]
    
    for title, page in toc_entries:
        p = doc.add_paragraph()
        p.paragraph_format.space_after = Pt(0)
        p.paragraph_format.line_spacing = Pt(18)
        
        from docx.oxml import OxmlElement
        from docx.oxml.ns import qn
        
        tab_stops_element = p._element.get_or_add_pPr().get_or_add_tabs()
        tab_stop = OxmlElement('w:tab')
        tab_stop.set(qn('w:val'), 'right')
        tab_stop.set(qn('w:leader'), 'dot')
        tab_stop.set(qn('w:pos'), str(int(Cm(16.5).twips)))
        tab_stops_element.append(tab_stop)
        
        run_title = p.add_run(title)
        run_title.font.size = Pt(14)
        run_title.font.name = 'Times New Roman'
        run_title.font.bold = True
        
        p.add_run('\t')
        
        run_page = p.add_run(str(page))
        run_page.font.size = Pt(14)
        run_page.font.name = 'Times New Roman'
        run_page.font.bold = True

def parse_content_structure_doklad(content: str, pages: int, order_data: dict) -> dict:
    """✅ DOKLAD - simple 2-part structure, NO intro/conclusion"""
    structure = {"parts": []}
    
    paragraphs = [p.strip() for p in content.split('\n\n') if p.strip()]
    
    # Split into 2 equal parts
    mid_point = len(paragraphs) // 2
    
    structure["parts"].append({
        "title": "ТЕОРЕТИЧЕСКАЯ ЧАСТЬ",
        "content": "\n\n".join(paragraphs[:mid_point])
    })
    
    structure["parts"].append({
        "title": "ПРАКТИЧЕСКАЯ ЧАСТЬ",
        "content": "\n\n".join(paragraphs[mid_point:])
    })
    
    return structure


def create_document_doklad(order_data: dict, content: str, lang: str) -> BytesIO:
    """✅ DOKLAD - БЕЗ титульного листа, header справа + тема по центру"""
    doc = Document()
    
    for section in doc.sections:
        section.top_margin = Cm(2)
        section.bottom_margin = Cm(2)
        section.left_margin = Cm(3)
        section.right_margin = Cm(1.5)
    
    # ✅ NO TITLE PAGE! Start with header
    
    # ✅ HEADER - Ýokarda sagda FIO + группа
    header_para = doc.add_paragraph()
    header_para.alignment = WD_ALIGN_PARAGRAPH.RIGHT
    run = header_para.add_run(f"{order_data['fullname']}\nгруппа {order_data['group']}")
    run.font.size = Pt(12)
    run.font.name = 'Times New Roman'
    header_para.paragraph_format.space_after = Pt(24)
    
    # ✅ TEMA - Ortada
    topic_para = doc.add_paragraph()
    topic_para.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = topic_para.add_run(order_data['topic'])
    run.font.size = Pt(16)
    run.font.name = 'Times New Roman'
    run.bold = True
    topic_para.paragraph_format.space_after = Pt(24)
    
    # ✅ CONTENT - Extend
    content = extend_content_to_required_pages(content, order_data)
    
    structure = parse_content_structure_doklad(content, order_data["pages"], order_data)
    
    # ✅ Write parts
    for part in structure["parts"]:
        # Part header
        part_header = doc.add_paragraph()
        part_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = part_header.add_run(part["title"])
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
        run.bold = True
        part_header.paragraph_format.space_before = Pt(18)
        part_header.paragraph_format.space_after = Pt(18)
        
        # Part content
        paragraphs = part["content"].split('\n\n')
        for para_text in paragraphs:
            if para_text.strip():
                p = doc.add_paragraph()
                p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
                p.paragraph_format.first_line_indent = Cm(1.25)
                p.paragraph_format.line_spacing = Pt(18)
                p.paragraph_format.space_after = Pt(0)
                
                clean_text = re.sub(r'[#\*_]', '', para_text.strip())
                run = p.add_run(clean_text)
                run.font.size = Pt(14)
                run.font.name = 'Times New Roman'
    
    # ✅ Page numbers
    add_page_numbers_referat(doc)
    
    buffer = BytesIO()
    doc.save(buffer)
    buffer.seek(0)
    return buffer


def create_toc_doklad(doc: Document, structure: dict, order_data: dict):
    """✅ TOC for DOKLAD"""
    
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    toc_header.paragraph_format.space_before = Pt(0)
    
    doc.add_paragraph()
    
    current_page = 2
    toc_page = current_page
    current_page += 1
    
    intro_page = current_page
    intro_words = len(structure["introduction"].split())
    current_page += max(1, intro_words // 400)
    
    chapter_pages = {}
    for chapter in structure["chapters"]:
        chapter_pages[chapter["number"]] = current_page
        chapter_words = len(chapter["content"].split())
        current_page += max(1, chapter_words // 400)
    
    conclusion_page = current_page
    concl_words = len(structure["conclusion"].split())
    current_page += max(1, concl_words // 400)
    
    ref_page = current_page
    
    toc_entries = [("ВВЕДЕНИЕ", intro_page, False)]
    
    for chapter in structure["chapters"]:
        toc_entries.append((f"{chapter['number']}. {chapter['title'].upper()}", chapter_pages[chapter['number']], False))
    
    toc_entries.append(("ЗАКЛЮЧЕНИЕ", conclusion_page, False))
    toc_entries.append(("СПИСОК ЛИТЕРАТУРЫ", ref_page, False))
    
    for title, page, is_sub in toc_entries:
        p = doc.add_paragraph()
        p.paragraph_format.space_after = Pt(0)
        p.paragraph_format.line_spacing = Pt(18)
        
        from docx.oxml import OxmlElement
        from docx.oxml.ns import qn
        
        tab_stops_element = p._element.get_or_add_pPr().get_or_add_tabs()
        tab_stop = OxmlElement('w:tab')
        tab_stop.set(qn('w:val'), 'right')
        tab_stop.set(qn('w:leader'), 'dot')
        tab_stop.set(qn('w:pos'), str(int(Cm(16.5).twips)))
        tab_stops_element.append(tab_stop)
        
        run_title = p.add_run(title)
        run_title.font.size = Pt(14)
        run_title.font.name = 'Times New Roman'
        run_title.font.bold = True
        
        p.add_run('\t')
        
        run_page = p.add_run(str(page))
        run_page.font.size = Pt(14)
        run_page.font.name = 'Times New Roman'
        run_page.font.bold = True

def extend_content_if_short(content: str, order_data: dict) -> str:
    words = len(content.split())
    required_words = order_data['pages'] * 350
    
    logger.info(f"Content: {words} words, required: {required_words}")
    
    if words < required_words:
        logger.warning("Content too short! Extending...")
        extensions = f"""

ДОПОЛНИТЕЛЬНЫЙ АНАЛИЗ

Рассматривая данную тему более подробно, необходимо отметить следующие аспекты. 
{order_data['topic']} является комплексной проблемой, требующей всестороннего изучения.

МЕТОДОЛОГИЧЕСКИЕ ОСНОВЫ

При изучении темы {order_data['topic']} применяются различные методы исследования.
Теоретические методы включают анализ, синтез, обобщение и систематизацию знаний.

ПРАКТИЧЕСКОЕ ПРИМЕНЕНИЕ

Результаты исследований в области {order_data['subject']} находят широкое применение на практике.
Внедрение современных технологий и методов позволяет повысить эффективность работы."""
        content += extensions
    
    return content



def parse_content_structure_kursovaya(content: str, pages: int, order_data: dict) -> dict:
    """✅ KURSOVAYA - detailed 3-chapter structure"""
    
    structure = {"introduction": "", "chapters": [], "conclusion": "", "references": []}
    
    paragraphs = [p.strip() for p in content.split('\n\n') if p.strip()]
    total_paras = len(paragraphs)
    
    intro_count = max(3, int(total_paras * 0.10))
    chapter1_count = int(total_paras * 0.30)
    chapter2_count = int(total_paras * 0.30)
    chapter3_count = int(total_paras * 0.20)
    conclusion_count = max(3, int(total_paras * 0.10))
    
    structure["introduction"] = "\n\n".join(paragraphs[:intro_count])
    
    current_pos = intro_count
    
    # CHAPTER 1 - Theory
    chapter1_paras = paragraphs[current_pos:current_pos + chapter1_count]
    subsection_size1 = len(chapter1_paras) // 3
    
    subsections1 = []
    for j in range(3):
        start = j * subsection_size1
        end = start + subsection_size1 if j < 2 else len(chapter1_paras)
        subsection_text = "\n\n".join(chapter1_paras[start:end])
        
        if subsection_text:
            titles1 = ["Теоретические основы", "Анализ литературы", "Методология исследования"]
            subsections1.append({
                "number": f"1.{j+1}",
                "title": titles1[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 1,
        "title": "Теоретическая часть",
        "subsections": subsections1
    })
    
    current_pos += chapter1_count
    
    # CHAPTER 2 - Practice
    chapter2_paras = paragraphs[current_pos:current_pos + chapter2_count]
    subsection_size2 = len(chapter2_paras) // 3
    
    subsections2 = []
    for j in range(3):
        start = j * subsection_size2
        end = start + subsection_size2 if j < 2 else len(chapter2_paras)
        subsection_text = "\n\n".join(chapter2_paras[start:end])
        
        if subsection_text:
            titles2 = ["Анализ текущего состояния", "Выявленные проблемы", "Предлагаемые решения"]
            subsections2.append({
                "number": f"2.{j+1}",
                "title": titles2[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 2,
        "title": "Практическая часть",
        "subsections": subsections2
    })
    
    current_pos += chapter2_count
    
    # CHAPTER 3 - Results
    chapter3_paras = paragraphs[current_pos:current_pos + chapter3_count]
    subsection_size3 = len(chapter3_paras) // 2
    
    subsections3 = []
    for j in range(2):
        start = j * subsection_size3
        end = start + subsection_size3 if j < 1 else len(chapter3_paras)
        subsection_text = "\n\n".join(chapter3_paras[start:end])
        
        if subsection_text:
            titles3 = ["Результаты исследования", "Рекомендации и выводы"]
            subsections3.append({
                "number": f"3.{j+1}",
                "title": titles3[j],
                "content": subsection_text
            })
    
    structure["chapters"].append({
        "number": 3,
        "title": "Результаты и рекомендации",
        "subsections": subsections3
    })
    
    current_pos += chapter3_count
    
    structure["conclusion"] = "\n\n".join(paragraphs[current_pos:current_pos + conclusion_count])
    structure["references"] = generate_references(order_data, random.randint(15, 25))
    
    return structure


def create_document_kursovaya(order_data: dict, content: str, lang: str) -> BytesIO:
    """✅ KURSOVAYA - full structure like referat"""
    doc = Document()
    
    for section in doc.sections:
        section.top_margin = Cm(2)
        section.bottom_margin = Cm(2)
        section.left_margin = Cm(3)
        section.right_margin = Cm(1.5)
    
    create_title_page(doc, order_data, lang)
    
    # ZADANIE page (REQUIRED for kursovaya)
    if order_data.get("zadanie_photo"):
        create_zadanie_page(doc, order_data)
    
    content = extend_content_to_required_pages(content, order_data)
    
    structure = parse_content_structure_kursovaya(content, order_data["pages"], order_data)
    
    create_toc_kursovaya(doc, structure, order_data)
    
    doc.add_page_break()
    
    # ВВЕДЕНИЕ
    intro_header = doc.add_paragraph()
    intro_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = intro_header.add_run("ВВЕДЕНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    intro_header.paragraph_format.space_before = Pt(0)
    intro_header.paragraph_format.space_after = Pt(18)
    
    intro_paragraphs = structure["introduction"].split('\n\n')
    for para_text in intro_paragraphs:
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            p.paragraph_format.space_after = Pt(0)
            
            clean_text = re.sub(r'[#\*_]', '', para_text.strip())
            run = p.add_run(clean_text)
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    
    # CHAPTERS
    for chapter in structure["chapters"]:
        ch_header = doc.add_paragraph()
        ch_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
        run = ch_header.add_run(f"ГЛАВА {chapter['number']}. {chapter['title'].upper()}")
        
        # ✅ ŞU ÝERDE ŞRIFTI BERKIDÝÄRIS:
        run.font.size = Pt(14)  # 11 Pt-den 14 Pt-e üýtgedildi
        run.font.name = 'Times New Roman'
        run.bold = True
        ch_header.paragraph_format.space_before = Pt(12)
        ch_header.paragraph_format.space_after = Pt(18)
        
        for subsection in chapter["subsections"]:
            sub_header = doc.add_paragraph()
            sub_header.alignment = WD_ALIGN_PARAGRAPH.LEFT
            sub_header.paragraph_format.left_indent = Cm(1.25)
            
            run = sub_header.add_run(f"{subsection['number']} {subsection['title']}")
            
            # ✅ KIÇI BÖLÜM ŞRIFTI HEM 14 PT:
            run.font.size = Pt(14) # 11 Pt-den 14 Pt-e üýtgedildi
            run.font.name = 'Times New Roman'
            run.bold = True
            sub_header.paragraph_format.space_before = Pt(12)
            sub_header.paragraph_format.space_after = Pt(12)
            
            # Mazmuny (Media elementleri bilen) goşmak
            insert_smart_content(doc, subsection["content"])
            
            sub_paragraphs = subsection["content"].split('\n\n')
            for para_text in sub_paragraphs:
                if para_text.strip():
                    p = doc.add_paragraph()
                    p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
                    p.paragraph_format.first_line_indent = Cm(1.25)
                    p.paragraph_format.line_spacing = Pt(18)
                    p.paragraph_format.space_after = Pt(0)
                    
                    clean_text = re.sub(r'[#\*_]', '', para_text.strip())
                    run = p.add_run(clean_text)
                    run.font.size = Pt(14)
                    run.font.name = 'Times New Roman'
        
        doc.add_page_break()
    
    # ЗАКЛЮЧЕНИЕ
    concl_header = doc.add_paragraph()
    concl_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = concl_header.add_run("ЗАКЛЮЧЕНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    concl_header.paragraph_format.space_before = Pt(0)
    concl_header.paragraph_format.space_after = Pt(18)
    
    concl_paragraphs = structure["conclusion"].split('\n\n')
    for para_text in concl_paragraphs:
        if para_text.strip():
            p = doc.add_paragraph()
            p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
            p.paragraph_format.first_line_indent = Cm(1.25)
            p.paragraph_format.line_spacing = Pt(18)
            p.paragraph_format.space_after = Pt(0)
            
            clean_text = re.sub(r'[#\*_]', '', para_text.strip())
            run = p.add_run(clean_text)
            run.font.size = Pt(14)
            run.font.name = 'Times New Roman'
    
    doc.add_page_break()
    
    # СПИСОК ЛИТЕРАТУРЫ
    ref_header = doc.add_paragraph()
    ref_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = ref_header.add_run("СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    ref_header.paragraph_format.space_before = Pt(0)
    ref_header.paragraph_format.space_after = Pt(18)
    
    for i, ref in enumerate(structure["references"], 1):
        p = doc.add_paragraph()
        p.alignment = WD_ALIGN_PARAGRAPH.JUSTIFY
        p.paragraph_format.left_indent = Cm(1.25)
        p.paragraph_format.first_line_indent = Cm(-1.25)
        p.paragraph_format.line_spacing = Pt(18)
        p.paragraph_format.space_after = Pt(0)
        
        run = p.add_run(f"{i}. {ref}")
        run.font.size = Pt(14)
        run.font.name = 'Times New Roman'
    
    add_page_numbers_referat(doc)
    
    buffer = BytesIO()
    doc.save(buffer)
    buffer.seek(0)
    return buffer


def create_toc_kursovaya(doc: Document, structure: dict, order_data: dict):
    """✅ TOC for KURSOVAYA"""
    
    toc_header = doc.add_paragraph()
    toc_header.alignment = WD_ALIGN_PARAGRAPH.CENTER
    run = toc_header.add_run("СОДЕРЖАНИЕ")
    run.font.size = Pt(14)
    run.font.name = 'Times New Roman'
    run.bold = True
    toc_header.paragraph_format.space_after = Pt(18)
    toc_header.paragraph_format.space_before = Pt(0)
    
    doc.add_paragraph()
    
    current_page = 2
    
    if order_data.get("zadanie_photo"):
        current_page += 1  # Zadanie page
    
    toc_page = current_page
    current_page += 1
    
    intro_page = current_page
    intro_words = len(structure["introduction"].split())
    current_page += max(1, intro_words // 400)
    
    toc_entries = [("ВВЕДЕНИЕ", intro_page, False)]
    
    for chapter in structure["chapters"]:
        chapter_page = current_page
        toc_entries.append((f"ГЛАВА {chapter['number']}. {chapter['title'].upper()}", chapter_page, False))
        
        for subsection in chapter["subsections"]:
            toc_entries.append((f"{subsection['number']} {subsection['title']}", chapter_page, True))
        
        chapter_words = sum(len(sub["content"].split()) for sub in chapter["subsections"])
        current_page += max(1, chapter_words // 400)
    
    conclusion_page = current_page
    concl_words = len(structure["conclusion"].split())
    current_page += max(1, concl_words // 400)
    
    ref_page = current_page
    
    toc_entries.append(("ЗАКЛЮЧЕНИЕ", conclusion_page, False))
    toc_entries.append(("СПИСОК ИСПОЛЬЗОВАННЫХ ИСТОЧНИКОВ", ref_page, False))
    
    for title, page, is_subsection in toc_entries:
        p = doc.add_paragraph()
        p.paragraph_format.space_after = Pt(0)
        p.paragraph_format.line_spacing = Pt(18)
        
        if is_subsection:
            p.paragraph_format.left_indent = Cm(1.25)
        
        from docx.oxml import OxmlElement
        from docx.oxml.ns import qn
        
        tab_stops_element = p._element.get_or_add_pPr().get_or_add_tabs()
        tab_stop = OxmlElement('w:tab')
        tab_stop.set(qn('w:val'), 'right')
        tab_stop.set(qn('w:leader'), 'dot')
        tab_stop.set(qn('w:pos'), str(int(Cm(16.5).twips)))
        tab_stops_element.append(tab_stop)
        
        run_title = p.add_run(title)
        run_title.font.size = Pt(14)
        run_title.font.name = 'Times New Roman'
        
        if not is_subsection:
            run_title.font.bold = True
        
        p.add_run('\t')
        
        run_page = p.add_run(str(page))
        run_page.font.size = Pt(14)
        run_page.font.name = 'Times New Roman'
        
        if not is_subsection:
            run_page.font.bold = True

# ============== ADMIN FUNCTIONS ==============

async def admin_confirm_payment(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """✅ Admin confirms payment - COMPLETE WORKING VERSION"""
    query = update.callback_query
    
    if query.from_user.id != ADMIN_ID:
        await query.answer("⛔ Доступ запрещен!", show_alert=True)
        return
    
    order_id = query.data.split("_", 1)[1]
    
    if order_id not in pending_payments:
        await query.answer("❌ Заказ не найден!", show_alert=True)
        return
    
    await query.answer("✅ Начинаю обработку...")
    
    order = pending_payments[order_id]
    user_id = order["user_id"]
    lang = order["language"]
    work_type = order["work_type"]
    currency = get_currency_symbol(order["country"])
    
    order["status"] = "processing"
    
    processing_msg = f"""✅ *ОПЛАТА ПОДТВЕРЖДЕНА!*

📋 Заказ: `{order_id}`
💵 Сумма: {order['final_price']} {currency}

🚀 Начинаю генерацию работы...
⏱️ Это займёт 2-5 минут

Не закрывайте чат, файл придёт сюда!"""
    
    try:
        await context.bot.send_message(user_id, processing_msg, parse_mode='Markdown')
    except Exception as e:
        logger.error(f"Failed to notify user {user_id}: {e}")
    
    await query.edit_message_caption(f"⏳ Генерация работы: {order_id}...")
    
    try:
        logger.info(f"[{order_id}] Starting generation for {work_type}...")
        
        # ✅ GENERATE AI CONTENT
        content = generate_ai_content(order, work_type)
        
        if not content:
            raise Exception("AI generation returned empty content")
        
        content_length = len(content)
        word_count = len(content.split())
        logger.info(f"[{order_id}] Generated: {content_length} chars, {word_count} words")
        
        # ✅ EXTEND IF SHORT
        if work_type not in ["presentation", "table"]:
            content = extend_content_to_required_pages(content, order)
            logger.info(f"[{order_id}] After extension: {len(content)} chars, {len(content.split())} words")
        
        work_type_name = WORK_TYPES[work_type]["ru"]
        safe_topic = re.sub(r'[^\w\s-]', '', order["topic"])[:30]
        
        file_buffer = None
        filename = None
        file_format = None
        
        # ✅ GENERATE FILE BY TYPE
        logger.info(f"[{order_id}] Creating {work_type} file...")
        
        if work_type == "presentation":
            file_buffer = create_presentation(order, content)
            filename = f"{work_type_name}_{safe_topic}_{order_id}.pptx"
            file_format = "PPTX"
                      
        elif work_type == "referat":
            file_buffer = create_document_referat(order, content, lang)
            filename = f"{work_type_name}_{safe_topic}_{order_id}.docx"
            file_format = "DOCX"
            
        elif work_type == "esse":
            file_buffer = create_document_esse(order, content, lang)
            filename = f"{work_type_name}_{safe_topic}_{order_id}.docx"
            file_format = "DOCX"
            
        elif work_type == "doklad":
            file_buffer = create_document_doklad(order, content, lang)
            filename = f"{work_type_name}_{safe_topic}_{order_id}.docx"
            file_format = "DOCX"
            
        elif work_type == "kursovaya":
            file_buffer = create_document_kursovaya(order, content, lang)
            filename = f"{work_type_name}_{safe_topic}_{order_id}.docx"
            file_format = "DOCX"
            
        else:
            # Fallback to basic document
            file_buffer = create_document(order, content, lang)
            filename = f"{work_type_name}_{safe_topic}_{order_id}.docx"
            file_format = "DOCX"
        
        if not file_buffer:
            raise Exception("File buffer is empty")
        
        file_buffer.seek(0, 2)
        file_size = file_buffer.tell()
        file_buffer.seek(0)
        
        logger.info(f"[{order_id}] File created: {filename}, {file_size} bytes")
        
        if file_size < 5000:  # Less than 5KB is suspicious
            raise Exception(f"File too small: {file_size} bytes")
        
        # ✅ SEND FILE TO CUSTOMER
        caption = f"""📚 *ВАША РАБОТА ГОТОВА!*

━━━━━━━━━━━━━━━━━━━━
📋 Заказ: `{order_id}`
📝 Тема: {order['topic'][:50]}
📄 Страниц: {order['pages']}
📁 Формат: {file_format}
💾 Размер: {file_size // 1024} KB

━━━━━━━━━━━━━━━━━━━━
✅ Работа полностью готова!
🎓 Проверьте содержание
🎁 Каждый 8-й заказ БЕСПЛАТНО!

Новый заказ: /start"""
        
        max_retries = 3
        send_success = False
        
        for attempt in range(max_retries):
            try:
                file_buffer.seek(0)  # Reset position
                
                await context.bot.send_document(
                    chat_id=user_id,
                    document=InputFile(file_buffer, filename=filename),
                    caption=caption,
                    parse_mode='Markdown',
                    read_timeout=180,
                    write_timeout=180,
                    connect_timeout=90
                )
                
                logger.info(f"[{order_id}] ✅ File sent to customer!")
                send_success = True
                break
                
            except Exception as send_error:
                logger.error(f"[{order_id}] Send attempt {attempt + 1} failed: {send_error}")
                if attempt < max_retries - 1:
                    await asyncio.sleep(3)
                else:
                    raise send_error
        
        if not send_success:
            raise Exception("Failed to send file after 3 attempts")
        
        # ✅ UPDATE USER STATS
        user_data = get_user(user_id)
        user_data["orders_count"] += 1
        user_data["total_spent"] += order["final_price"]
        
        if order.get("promo_code"):
            promo_upper = order["promo_code"].upper()
            if promo_upper not in user_data["used_promos"]:
                user_data["used_promos"].append(promo_upper)
        
        # ✅ SAVE ORDER
        order["status"] = "completed"
        order["completed_at"] = datetime.now().isoformat()
        order["file_format"] = file_format
        order["file_size"] = file_size
        orders_db[order_id] = order
        del pending_payments[order_id]
        
        # ✅ UPDATE ADMIN MESSAGE
        await query.edit_message_caption(
            f"""✅ *ЗАКАЗ ВЫПОЛНЕН*

━━━━━━━━━━━━━━━━━━━━
📋 ID: `{order_id}`
📁 Формат: {file_format}
💾 Размер: {file_size // 1024} KB
⏰ Время: {datetime.now().strftime('%H:%M:%S')}

✅ Файл отправлен клиенту!""",
            parse_mode='Markdown'
        )
        
        logger.info(f"[{order_id}] ✅✅✅ ORDER COMPLETED SUCCESSFULLY ✅✅✅")
        
    except Exception as e:
        logger.error(f"[{order_id}] ❌ CRITICAL ERROR: {str(e)}", exc_info=True)
        
        # ✅ NOTIFY CUSTOMER ABOUT ERROR
        error_msg = f"""❌ *ТЕХНИЧЕСКАЯ ОШИБКА*

📋 Заказ: `{order_id}`

🔧 Ваша работа будет выполнена вручную администратором
⏱️ Доставка в течение 1-3 часов
💰 Оплата учтена и сохранена

Приносим извинения за задержку!"""
        
        try:
            await context.bot.send_message(user_id, error_msg, parse_mode='Markdown')
        except:
            pass
        
        # ✅ NOTIFY ADMIN
        admin_error = f"""❌ *ОШИБКА ОБРАБОТКИ*

📋 Order: `{order_id}`
👤 User: {order['full_name']} (@{order.get('username', 'N/A')})
📝 Type: {work_type}
💵 Price: {order['final_price']} {currency}

❌ Error: {str(e)[:500]}

⚠️ Требуется ручная обработка!"""
        
        try:
            await context.bot.send_message(ADMIN_ID, admin_error, parse_mode='Markdown')
        except:
            pass
        
        # ✅ UPDATE ADMIN MESSAGE
        try:
            await query.edit_message_caption(
                f"❌ *ОШИБКА*\n\nЗаказ: `{order_id}`\n\n⚠️ Требуется ручная обработка!",
                parse_mode='Markdown'
            )
        except:
            pass


# ============== AI GENERATION - GROQ ONLY ==============

def generate_ai_content(order_data: dict, work_type: str) -> Optional[str]:
    """✅ Täzelenen we durnukly AI generatory. 
    Limitleriň dolmazlygy we referatlaryň ýitmezligi üçin optimizirlenen."""
    
    try:
        url = "https://api.groq.com/openai/v1/chat/completions"
        
        pages = order_data['pages']
        topic = order_data['topic']
        subject = order_data['subject']
        university = order_data.get('university', '')
        
        # ⚠️ MÖHÜM: AI-dan bir gezekde 1500-den köp söz soramaň, ýogsam ýazyp bilmez.
        # Galan sahypalary "extend_content_to_required_pages" funksiýasy doldurar.
        target_words = 1500 

        # --- IŞ GÖRNÜŞINE GÖRÄ PROMTLAR ---
        
        if work_type in ["referat", "kursovaya"]:
            prompt = f"""Напиши академическую работу на тему: "{order_data['topic']}"
Предмет: {order_data['subject']}

СТРУКТУРА: Введение, Глава 1, Глава 2, Заключение.

ОБЯЗАТЕЛЬНО ДОБАВЬ В ТЕКСТ:
1. Минимум 2 таблицы в формате Markdown (например: | Заголовок | Значение |).
2. Укажи места для 2-3 иллюстраций тегом: [IMAGE: подробное описание картинки на английском].
3. Укажи место для 1 логической схемы тегом: [SCHEMA: описание схемы на русском].

ТРЕБОВАНИЯ:
- Стиль: Научный.
- Текст: Только русский.
- Без списков, только длинные и информативные абзацы."""

        elif work_type == "presentation":
            prompt = f"""Напиши контент для презентации на тему: "{topic}"
Количество слайдов: {pages}
Для каждого слайда напиши 4-5 подробных пунктов и IMAGE_KEYWORD: [ключевое слово на английском]."""

        elif work_type == "esse":
            prompt = f"""Напиши подробное эссе на тему: "{topic}". Минимум 800 слов. Академический стиль."""

        elif work_type == "doklad":
            prompt = f"""Напиши доклад на тему: "{topic}". Минимум 1000 слов. Только суть без вступлений."""

        elif work_type == "kursovaya":
            prompt = f"""Напиши курсовую работу на тему: "{topic}". Предмет: {subject}. 
            Максимально подробно разпиши теорию и практику. Минимум 2000 слов."""

        else:
            prompt = f"""Напиши подробную работу на тему: "{topic}". Минимум 1000 слов."""
        
        # --- API ÇAGYRYŞY (HAS DURNUKLY MODEL BILEN) ---
        
        headers = {
            "Authorization": f"Bearer {GROQ_API_KEY}",
            "Content-Type": "application/json"
        }
        
        payload = {
            # "llama-3.1-8b-instant" has çalt we limitleri has uly
            "model": "llama-3.1-8b-instant", 
            "messages": [{"role": "user", "content": prompt}],
            "temperature": 0.7,
            "max_tokens": 8000, # AI-yň jogap berip biljek max ýeri
            "top_p": 0.9
        }
        
        logger.info(f"🤖 AI generirleýär: {work_type.upper()} (Model: llama-3.1-8b)")
        
        response = requests.post(url, json=payload, headers=headers, timeout=300)
        
        if response.status_code == 200:
            data = response.json()
            content = data["choices"][0]["message"]["content"]
            
            if not content or len(content.split()) < 50:
                logger.error("❌ AI gaty gysga jogap berdi!")
                return None
                
            logger.info(f"✅ AI Jogap berdi: {len(content.split())} söz.")
            return content
        else:
            logger.error(f"❌ GROQ Error {response.status_code}: {response.text}")
            return None
            
    except Exception as e:
        logger.error(f"❌ generate_ai_content içinde ýalňyşlyk: {str(e)}")
        return None
    
    # ✅ EXCEPT BLOKY (MANDATORY!)
    except Exception as e:
        logger.error(f"❌ Generation exception for {work_type}: {str(e)}")
        return None

async def admin_reject_payment(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    
    if query.from_user.id != ADMIN_ID:
        await query.answer("⛔ Доступ запрещен!", show_alert=True)
        return
    
    order_id = query.data.split("_", 1)[1]
    
    if order_id not in pending_payments:
        await query.answer("❌ Заказ не найден!", show_alert=True)
        return
    
    order = pending_payments.pop(order_id)
    lang = order["language"]
    
    msg = f"❌ *ОПЛАТА НЕ ПОДТВЕРЖДЕНА*\n\n📋 ID: `{order_id}`\n\nПопробуйте снова: /start"
    await context.bot.send_message(order["user_id"], msg, parse_mode='Markdown')
    
    await query.edit_message_caption(f"❌ *ОТКЛОНЕНО*\n\nЗаказ: {order_id}", parse_mode='Markdown')
    await query.answer("❌ Заказ отклонен")

async def show_promotions(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    lang = get_user(user_id).get("language", "ru")
    
    text = """🎁 *АКЦИИ*

━━━━━━━━━━━━━━━━━━━━
🎁 *8-й заказ БЕСПЛАТНО!*
☀️ *Утро (06:00-07:00): -10%*
👥 *Приведи друга: -30% ОБОИМ!*
🎉 *Выходные: -10%*
🏷️ *Промокоды: до -30%*

💡 Скидки суммируются!
*Максимум 50%*
━━━━━━━━━━━━━━━━━━━━"""
    
    keyboard = [
        [InlineKeyboardButton(TEXTS[lang]["new_order"], callback_data="new_order")],
        [InlineKeyboardButton(TEXTS[lang]["back"], callback_data="main_menu")]
    ]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def show_account(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    user = get_user(user_id)
    lang = user.get("language", "ru")
    
    orders_to_free = 8 - (user["orders_count"] % 8)
    if orders_to_free == 8 and user["orders_count"] > 0:
        orders_to_free = 0
    
    progress = "🟢" * (user["orders_count"] % 8) + "⚪" * orders_to_free
    
    text = f"""📊 *МОЙ АККАУНТ*

━━━━━━━━━━━━━━━━━━━━
👤 ID: `{user_id}`

📈 *Статистика:*
• Заказов: {user['orders_count']}
• Потрачено: {user['total_spent']}
• Рефералов: {len(user['referrals'])}

🎁 *До бесплатного:*
{progress}
• {user['orders_count'] % 8}/7
• Осталось: {orders_to_free if orders_to_free > 0 else '🎉 СЛЕДУЮЩИЙ БЕСПЛАТНО!'}
━━━━━━━━━━━━━━━━━━━━"""
    
    keyboard = [
        [InlineKeyboardButton(TEXTS[lang]["new_order"], callback_data="new_order")],
        [InlineKeyboardButton(TEXTS[lang]["back"], callback_data="main_menu")]
    ]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def show_referral(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    user = get_user(user_id)
    lang = user.get("language", "ru")
    
    bot_info = await context.bot.get_me()
    ref_link = f"https://t.me/{bot_info.username}?start={user_id}"
    
    text = f"""👥 *РЕФЕРАЛЬНАЯ ПРОГРАММА*

━━━━━━━━━━━━━━━━━━━━
🎁 *Как работает:*
1️⃣ Поделитесь ссылкой
2️⃣ Друг регистрируется
3️⃣ *Оба получаете -30%!* 🎉

━━━━━━━━━━━━━━━━━━━━
🔗 *Ваша ссылка:*
`{ref_link}`

📊 *Статистика:*
• Рефералов: {len(user['referrals'])}
━━━━━━━━━━━━━━━━━━━━"""
    
    keyboard = [[InlineKeyboardButton(TEXTS[lang]["back"], callback_data="main_menu")]]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def enter_promo(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    lang = get_user(user_id).get("language", "ru")
    
    text = """🏷️ *ПРОМОКОД*

Введите код:

*Доступные:*
• WELCOME — 20%
• STUDENT — 15%
• FIRST — 25%
• VIP2025 — 30%"""
    
    keyboard = [[InlineKeyboardButton(TEXTS[lang]["cancel"], callback_data="main_menu")]]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')
    context.user_data["waiting_promo"] = True

async def handle_promo_input(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.user_data.get("waiting_promo"): return
    code = update.message.text.strip().upper()
    user_id = update.effective_user.id
    user = get_user(user_id)
    context.user_data["waiting_promo"] = False
    
    today = datetime.now().strftime("%m-%d") # Häzirki sene
    
    if code in HOLIDAY_PROMOS:
        promo = HOLIDAY_PROMOS[code]
        if promo["date"] == today: # ✅ Diňe baýram gününde
            if code in user["used_promos"]:
                await update.message.reply_text(f"❌ Код {code} уже использован!")
            else:
                context.user_data["promo_code"] = code
                await update.message.reply_text(f"🎉 Скидка {promo['discount']}% принята в честь праздника: {promo['name']}!")
        else:
            d = promo["date"].split("-")
            await update.message.reply_text(f"⚠️ Код {code} работает только {d[1]}.{d[0]} ({promo['name']})!")
    else:
        await update.message.reply_text("❌ Неверный или неактивный код.")


async def show_help(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    lang = get_user(user_id).get("language", "ru")
    
    text = """❓ *ПОМОЩЬ*

1️⃣ Нажмите "Новый заказ"
2️⃣ Выберите страну
3️⃣ Выберите тип работы
4️⃣ Заполните данные
5️⃣ Оплатите
6️⃣ Отправьте скриншот
7️⃣ Получите файл! 🎉

*Команды:*
/start — Главное меню
/help — Помощь
/cancel — Отмена"""
    
    keyboard = [[InlineKeyboardButton(TEXTS[lang]["back"], callback_data="main_menu")]]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def admin_panel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    
    if query.from_user.id != ADMIN_ID:
        await query.answer("⛔ Доступ запрещен!", show_alert=True)
        return
    
    await query.answer()
    
    total_users = len(users_db)
    total_pending = len(pending_payments)
    total_completed = len(orders_db)
    total_revenue_by = sum(o.get("final_price", 0) for o in orders_db.values() if o.get("country") == "BY")
    total_revenue_ru = sum(o.get("final_price", 0) for o in orders_db.values() if o.get("country") == "RU")
    
    text = f"""🔐 *ADMIN PANEL*

━━━━━━━━━━━━━━━━━━━━
📊 *Статистика:*
• Пользователей: {total_users}
• Ожидают: {total_pending}
• Выполнено: {total_completed}

💰 *Доход:*
• 🇧🇾 BY: {total_revenue_by} BYN
• 🇷🇺 RU: {total_revenue_ru} ₽
━━━━━━━━━━━━━━━━━━━━"""
    
    keyboard = [
        [InlineKeyboardButton(f"⏳ Ожидающие ({total_pending})", callback_data="admin_pending")],
        [InlineKeyboardButton("🔙 Назад", callback_data="main_menu")]
    ]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def admin_pending(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    
    if query.from_user.id != ADMIN_ID:
        await query.answer("⛔ Доступ запрещен!", show_alert=True)
        return
    
    await query.answer()
    
    if not pending_payments:
        text = "✅ Нет ожидающих заказов!"
    else:
        text = "⏳ *ОЖИДАЮЩИЕ:*\n\n"
        for order_id, order in pending_payments.items():
            currency = get_currency_symbol(order["country"])
            flag = "🇧🇾" if order["country"] == "BY" else "🇷🇺"
            text += f"📋 `{order_id}` {flag}\n• {order['full_name']}\n• {order['topic'][:30]}...\n• {order['final_price']} {currency}\n━━━━━━━━━━━━━━━━━━\n"
    
    keyboard = [[InlineKeyboardButton("🔙 Назад", callback_data="admin")]]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def change_language(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    text = "🌍 *Выберите язык / Select language:*"
    keyboard = [[InlineKeyboardButton("🇷🇺 Русский", callback_data="lang_ru"), InlineKeyboardButton("🇬🇧 English", callback_data="lang_en")]]
    
    await query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode='Markdown')

async def cancel_order(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    
    user_id = query.from_user.id
    lang = get_user(user_id).get("language", "ru")
    
    context.user_data.clear()
    
    msg = "❌ *Заказ отменен*\n\n/start"
    await query.edit_message_text(msg, parse_mode='Markdown')
    return ConversationHandler.END

async def main_menu_callback(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    await show_main_menu(update, context)

async def cancel_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data.clear()
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    msg = "❌ Отменено. /start"
    await update.message.reply_text(msg)
    return ConversationHandler.END

async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    text = "/start для начала"
    await update.message.reply_text(text)

async def handle_text_messages(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if context.user_data.get("waiting_promo"):
        await handle_promo_input(update, context)
        return
    
    user_id = update.effective_user.id
    lang = get_user(user_id).get("language", "ru")
    await update.message.reply_text("Для заказа: /start\nПомощь: /help")

# ============== MAIN ==============

def main():
    app = Application.builder().token(BOT_TOKEN).build()
    
    order_conv = ConversationHandler(
    entry_points=[CallbackQueryHandler(new_order, pattern="^new_order$")],
    states={
        SELECT_COUNTRY: [CallbackQueryHandler(select_country, pattern="^country_")],
        SELECT_WORK_TYPE: [CallbackQueryHandler(select_work_type, pattern="^work_")],
        SELECT_PAGES: [CallbackQueryHandler(select_pages, pattern="^pages_")],
        
        # ❌ UPLOAD_TABLE_TASK AÝRYLDY!
        
        ENTER_TOPIC: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_topic)],
        ENTER_UNIVERSITY: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_university)],
        ENTER_FACULTY: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_faculty)],
        ENTER_SUBJECT: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_subject)],
        ENTER_FULLNAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_fullname)],
        ENTER_COURSE: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_course)],
        ENTER_GROUP: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_group)],
        ENTER_TEACHER: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_teacher)],
        ENTER_CITY: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_city)],
        ENTER_PHONE: [MessageHandler(filters.TEXT & ~filters.COMMAND, receive_phone)],
        
        UPLOAD_ZADANIE: [
            MessageHandler(filters.PHOTO, receive_zadanie),
            CallbackQueryHandler(skip_zadanie, pattern="^skip_zadanie$")
        ],
        
        PAYMENT_PHOTO: [
            MessageHandler(filters.PHOTO, receive_payment_photo),
            CallbackQueryHandler(cancel_order, pattern="^cancel_order$")
        ]
    },
    fallbacks=[
        CommandHandler("cancel", cancel_command),
        CallbackQueryHandler(cancel_order, pattern="^cancel_order$"),
        CallbackQueryHandler(main_menu_callback, pattern="^main_menu$")
    ],
    per_message=False
)
    
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(CommandHandler("cancel", cancel_command))
    app.add_handler(CallbackQueryHandler(select_language, pattern="^lang_"))
    app.add_handler(CallbackQueryHandler(change_language, pattern="^change_lang$"))
    app.add_handler(order_conv)
    app.add_handler(CallbackQueryHandler(main_menu_callback, pattern="^main_menu$"))
    app.add_handler(CallbackQueryHandler(show_promotions, pattern="^promotions$"))
    app.add_handler(CallbackQueryHandler(show_account, pattern="^account$"))
    app.add_handler(CallbackQueryHandler(show_referral, pattern="^referral$"))
    app.add_handler(CallbackQueryHandler(enter_promo, pattern="^enter_promo$"))
    app.add_handler(CallbackQueryHandler(show_help, pattern="^help$"))
    app.add_handler(CallbackQueryHandler(admin_panel, pattern="^admin$"))
    app.add_handler(CallbackQueryHandler(admin_pending, pattern="^admin_pending$"))
    app.add_handler(CallbackQueryHandler(admin_confirm_payment, pattern="^confirm_"))
    app.add_handler(CallbackQueryHandler(admin_reject_payment, pattern="^reject_"))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_text_messages))
    
    print("🚀 Bot işleýär...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
