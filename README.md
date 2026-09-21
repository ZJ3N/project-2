# Book Store API — PostgreSQL

نسخة المشروع بعد التحويل من SQLite إلى PostgreSQL، مع جداول علائقية للمؤلفين (`author`) والتصنيفات (`genre`)، مرتبطة بالكتب بعلاقات one-to-many وmany-to-many وone-to-one.

## المتطلبات

- Python 3.12+
- Docker Desktop (لتشغيل PostgreSQL)

## 1. تشغيل قاعدة بيانات PostgreSQL

```bash
docker run --name bookstore-db2 -e POSTGRES_PASSWORD=devpassword -e POSTGRES_DB=bookstore -p 5432:5432 -d postgres:16
```

تأكد أنها جاهزة:

```bash
docker exec bookstore-db2 pg_isready -U postgres -d bookstore
```

يجب أن تشوف: `accepting connections`

> لاحقاً، لإيقافها وتشغيلها من جديد (بدون فقدان البيانات):
> `docker stop bookstore-db2` / `docker start bookstore-db2`
> حذف الحاوية نهائياً (وحذف البيانات معها): `docker rm -f bookstore-db2`

## 2. تجهيز بيئة Python

```bash
python -m venv venv
```

تفعيل البيئة الافتراضية:

- macOS / Linux: `source venv/bin/activate`
- Windows: `venv\Scripts\activate`

تثبيت المكتبات:

```bash
pip install -r requirements.txt
```

## 3. إعداد ملف `.env`

ملف `.env` موجود مسبقاً بهذا المحتوى، تأكد أنه يطابق بيانات الحاوية اللي سويتها في الخطوة 1:

```
SECRET_KEY=...
ACCESS_TOKEN_EXPIRE_MINUTES=30
DEBUG=false
DATABASE_URL=postgresql://postgres:devpassword@localhost:5432/bookstore
```

`DATABASE_URL` بهذا الشكل:

```
postgresql://  postgres  :  devpassword  @  localhost  :  5432  /  bookstore
    النوع        المستخدم      الباسورد        السيرفر       المنفذ    اسم القاعدة
```

## 4. تشغيل السيرفر

```bash
uvicorn main:app --reload
```

عند أول تشغيل، `create_all` ينشئ كل الجداول تلقائياً (`user`, `author`, `author_profile`, `genre`, `book`, `book_genre`, `order`) — ما تحتاج تسوي شي إضافي.

السيرفر يشتغل على: `http://127.0.0.1:8000`
التوثيق التفاعلي (Swagger): `http://127.0.0.1:8000/docs`

## 5. إنشاء حساب أدمن

قاعدة PostgreSQL تبدأ فارغة، فلازم تنشئ حساب الأدمن من جديد:

```bash
python create_admin.py
```

## بنية المشروع

```
app/
  config.py          # إعدادات المشروع (env vars)
  database.py         # اتصال قاعدة البيانات (SQLAlchemy engine + session)
  security.py          # تشفير الباسورد وJWT
  dependencies.py       # صلاحيات الأدوار (role-based auth)
  enums.py              # Role, OrderStatus
  models.py             # جداول قاعدة البيانات (User, Author, AuthorProfile, Genre, Book, BookGenreLink, Order)
  dtos/
    requests.py          # شكل البيانات المسموح استلامها من العميل
    responses.py          # شكل البيانات المسموح إرجاعها للعميل
  routers/
    auth.py               # تسجيل / دخول
    users.py               # إدارة المستخدمين والأدوار
    authors.py              # المؤلفين وبروفايلاتهم
    genres.py                # التصنيفات
    books.py                  # الكتب (مرتبطة بمؤلف وتصنيفات)
    orders.py                  # الطلبات
main.py                # نقطة تشغيل التطبيق (FastAPI + الراوترات)
create_admin.py         # سكربت لإنشاء حساب أدمن
```

## العلاقات بين الجداول

| العلاقة | النوع | كيف مبنية |
|---|---|---|
| مؤلف → كتب | one-to-many | `book.author_id` → `author.id` |
| كتاب ↔ تصنيفات | many-to-many | جدول وسيط `book_genre` |
| مؤلف → بروفايل | one-to-one | `author_profile.author_id` → `author.id` (وهو أيضاً المفتاح الأساسي) |
| مستخدم → طلبات | one-to-many | `order.user_id` → `user.id` |
| كتاب → طلبات | one-to-many | `order.book_id` → `book.id` |

## استكشاف الأخطاء

**`Bind for 0.0.0.0:5432 failed: port is already allocated`**
فيه شي ثاني مستخدم المنفذ 5432 (سيرفر PostgreSQL آخر أو مثبت على جهازك مباشرة). تحقق بـ `docker ps` وأوقفه، أو استخدم منفذ ثاني: `-p 5433:5432` وعدّل `DATABASE_URL` ليصير `5433`.

**`Conflict. The container name "/bookstore-db" is already in use`**
الحاوية موجودة أصلاً. شغّلها بـ `docker start bookstore-db`، أو احذفها وابدأ من جديد (تفقد البيانات): `docker rm -f bookstore-db`.

**`ModuleNotFoundError: No module named 'psycopg2'`**
فعّل البيئة الافتراضية وسوي `pip install psycopg2-binary`.

**`password authentication failed for user "postgres"`**
الباسورد بـ `DATABASE_URL` ما يطابق `POSTGRES_PASSWORD`. لاحظ أن `POSTGRES_PASSWORD` يشتغل بس أول مرة تنشئ فيها الحاوية القاعدة — تغييره بعدين ما يأثر على حاوية موجودة أصلاً.

**رد `500` بعد أول طلب يجي بعد إعادة تشغيل القاعدة**
هذا طبيعي إذا `pool_pre_ping=True` مو موجود بـ `create_engine` — لكنه موجود بالنسخة الحالية من `app/database.py`، فما ينصير.
