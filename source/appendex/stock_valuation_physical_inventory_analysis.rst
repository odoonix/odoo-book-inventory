راهنمای آموزشی: چرا در Physical Inventory سند حسابداری ساخته نمی‌شود؟
======================================================================================

این راهنما برای افراد مبتدی نوشته شده است. هدف این است که به زبان ساده بفهمیم چرا
گاهی بعد از شمارش انبار در اودوو، حرکت انبار (``stock.move``) ثبت می‌شود اما
سند حسابداری (``Journal Entry``) ساخته نمی‌شود.

.. note::

   قابلیت ارزش‌گذاری لحظه‌ای انبار در این سطح، در نسخه Enterprise اودوو در دسترس است.
   بنابراین تحلیل این صفحه بر همان نسخه متمرکز است.


آنچه در این آموزش یاد می‌گیرید
-------------------------------

- نقش ماژول‌های ``stock_account`` و ``stock_accountant``
- مسیر اجرای کد از Apply All تا تلاش برای ساخت سند حسابداری
- شرط‌های اصلی ایجاد ``Journal Entry``
- دلیل‌های رایج ساخته نشدن سند
- روش عیب‌یابی سریع در Odoo Shell


دامنه تحلیل
-----------

این تحلیل روی دو ماژول زیر انجام شده است:

- ``stock_account``
- ``stock_accountant``

منطق اصلی ساخت سند حسابداری در ``stock_account`` قرار دارد. ماژول
``stock_accountant`` بیشتر تنظیمات و نماهای مرتبط را در اختیار کاربر قرار می‌دهد.


صورت مسئله (سناریوی واقعی)
---------------------------

در یک سناریوی متداول:

1. در نمای **Physical Inventory** یک یا چند رکورد ``stock.quant`` ثبت می‌کنید.
2. مقدار شمارش‌شده را وارد می‌کنید.
3. روی **Apply All** کلیک می‌کنید.
4. نتیجه: ``stock.move`` ساخته می‌شود، اما ``Journal Entry`` ساخته نمی‌شود.


پاسخ کوتاه
----------

ساخته شدن ``stock.move`` به‌تنهایی کافی نیست. برای ساخت ``Journal Entry`` باید
شرط‌های متد ``_should_create_account_move`` برقرار باشند. اگر این شرط‌ها پاس نشوند،
حرکت انبار ثبت می‌شود ولی سند حسابداری ساخته نمی‌شود.

مهم‌ترین علت عملی: روی یکی از لوکیشن‌های مبدأ یا مقصد،
``valuation_account_id`` تنظیم نشده است.


مسیر اجرای کد به زبان ساده
---------------------------

مرحله ۱: Apply Inventory روی Quant
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

در ماژول هسته ``stock``:

- ``stock.quant.action_apply_inventory()`` اجرا می‌شود.
- سپس ``stock.quant._apply_inventory()`` اجرا می‌شود.

در این مرحله، اگر اختلاف موجودی وجود داشته باشد، یک ``stock.move`` ایجاد می‌شود
(معمولاً بین لوکیشن داخلی و Inventory Loss).


مرحله ۲: افزودنی ماژول stock_account روی Quant
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

در ``stock_account/models/stock_quant.py``:

- متد ``_apply_inventory`` تاریخ حسابداری را با context
  ``force_period_date`` پاس می‌دهد.

نکته مهم: این بخش تصمیم‌گیرِ اصلی برای ساخت سند حسابداری نیست. اینجا بیشتر
«تاریخ ثبت حسابداری» کنترل می‌شود.


مرحله ۳: نهایی شدن move و تلاش برای ساخت سند
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

در ``stock_account/models/stock_move.py``:

- متد ``_action_done()`` بعد از ارزش‌گذاری، ``_create_account_move()`` را صدا می‌زند.
- ``_create_account_move()`` فقط برای moveهایی اجرا می‌شود که
  ``_should_create_account_move()`` برایشان True باشد.


مرحله ۴: شرط‌های کلیدی برای ایجاد Journal Entry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

در متد ``_should_create_account_move`` این شرط‌ها باید هم‌زمان برقرار باشند:

- ``product_id.is_storable``
- ``is_valued``
- ``(location_dest_id.valuation_account_id or location_id.valuation_account_id)``
- ``product_id.valuation == 'real_time'``

اگر حتی یکی از این شرط‌ها برقرار نباشد:

- ``stock.move`` ساخته می‌شود.
- ``account.move`` (Journal Entry) ساخته نمی‌شود.


نکته کلیدی در Inventory Adjustment
----------------------------------

در Inventory Adjustment معمولاً حرکت بین لوکیشن داخلی و لوکیشن Inventory Loss است.
اگر روی لوکیشن‌های درگیر ``valuation_account_id`` نداشته باشید،
شرط ساخت سند رد می‌شود.

به همین دلیل این رفتار را می‌بینید:

- ``stock.move`` ایجاد می‌شود.
- ``Journal Entry`` ایجاد نمی‌شود.


نقش هر ماژول به صورت خلاصه
---------------------------

``stock_account``:

- منطق ارزش‌گذاری و ساخت/ثبت ``account.move``
- تصمیم نهایی برای ایجاد سند حسابداری

``stock_accountant``:

- نمایش و expose کردن تنظیمات (روش ارزش‌گذاری، روش هزینه، ژورنال انبار و ...)
- افزودن گزارش‌ها و نماها

بنابراین در عیب‌یابی «چرا سند ساخته نشد؟»، تمرکز اصلی باید روی ``stock_account`` باشد.


چک‌لیست عیب‌یابی سریع (مبتدی‌پسند)
-----------------------------------

1) محصول
~~~~~~~~

- نوع محصول باید ``Storable`` باشد.
- ارزش‌گذاری محصول یا دسته باید ``real_time`` باشد.

2) لوکیشن‌ها
~~~~~~~~~~~~

- لوکیشن داخلی و Inventory Loss را بررسی کنید.
- دست‌کم یکی از ``location_id`` یا ``location_dest_id`` باید
  ``valuation_account_id`` داشته باشد.

3) ارزش‌گذاری شدن حرکت
~~~~~~~~~~~~~~~~~~~~~~~

- حرکت باید واقعاً از دید سیستم ارزش‌گذاری‌شونده باشد (``is_valued``).
- اگر مالکیت متفاوت باشد (consignment)، ممکن است حرکت از ارزش‌گذاری خارج شود.

4) شرکت و ژورنال
~~~~~~~~~~~~~~~~

- ``company.account_stock_journal_id`` باید تنظیم شده و قابل استفاده باشد.


بررسی عملی در Odoo Shell
------------------------

.. code-block:: python

   # 1) آخرین moveهای inventory
   moves = env['stock.move'].search([('is_inventory', '=', True)], order='id desc', limit=20)
   for m in moves:
       print(
           m.id,
           m.reference,
           m.value,
           m.is_valued,
           m.product_id.valuation,
           m.location_id.display_name,
           m.location_id.valuation_account_id.code,
           m.location_dest_id.display_name,
           m.location_dest_id.valuation_account_id.code,
           bool(m.account_move_id),
       )

.. code-block:: python

   # 2) نتیجه مستقیم شرط ایجاد سند
   for m in moves:
       print(m.id, m._should_create_account_move())

.. code-block:: python

   # 3) لوکیشن Inventory Loss پیش‌فرض هر محصول
   products = moves.mapped('product_id')
   for p in products:
       loss_loc = p.with_company(p.company_id).property_stock_inventory
       print(
           p.default_code,
           p.display_name,
           loss_loc.display_name,
           loss_loc.valuation_account_id.code,
       )


جمع‌بندی آموزشی
---------------

در اودوو، مسیر Physical Inventory ابتدا حرکت انبار را می‌سازد. اما ساخت سند حسابداری
فقط در صورت پاس شدن شرط‌های ``_should_create_account_move`` انجام می‌شود.

در تجربه عملی، رایج‌ترین علت عدم ایجاد ``Journal Entry`` این است که
``valuation_account_id`` روی لوکیشن‌های درگیر Inventory Adjustment تنظیم نشده است.

پس اگر ``stock.move`` دارید ولی ``Journal Entry`` ندارید، از لوکیشن‌ها و
شرط‌های متد بالا شروع کنید.
