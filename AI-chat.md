# ملخص أسئلة وإجابات مشروع REST Countries API (Milestone 1)

---

## 1. كيفية عمل DataFrame للموجود داخل رابط API؟

### الإجابة:
لتحويل البيانات من استجابة API إلى `Pandas DataFrame`، يجب أولاً استخراج القائمة التي تحتوي على البيانات الفعلية من القاموس (Dictionary). في استجابة هذا الـ API، توجد البيانات داخل المفاتيح `'data'` ثم `'objects'`.

```python
import requests
import pandas as pd

# 1. جلب البيانات من الـ API
response = requests.get(
    '[https://api.restcountries.com/countries/v5?q=canada](https://api.restcountries.com/countries/v5?q=canada)',
    headers={'Authorization': 'Bearer rc_live_200497a641014a519165cfe2be30e21d'}
)

data = response.json()

# 2. استخراج قائمة الكائنات (objects)
if 'data' in data and 'objects' in data['data']:
    objects_list = data['data']['objects']
    
    # 3. إنشاء DataFrame مع فك القواميس المتداخلة (Flattening)
    df = pd.json_normalize(objects_list)
    print(df)
else:
    print("لم يتم العثور على المفاتيح 'data' و 'objects' في الاستجابة.")
```

> **ملاحظة:** استخدام `pd.json_normalize()` أفضل من `pd.DataFrame()` هنا لأنه يفك تداخل القواميس المتداخلة (مثل `names.common` و `names.native.eng.official`) إلى أعمدة منفصلة.

---

## 2. ما المطلوب مني أفعله في أول خطوتين بالـ Milestone الأولى بالتفصيل؟

### الإجابة:

**الخطوة الأولى: استخراج البيانات وفك التداخل (Data Extraction & Flattening)**
* **المطلوب:** سحب بيانات جميع الدول من REST Countries API.
* **التفصيل:** الاستجابة تأتي بصيغة JSON وتحتوي على حقول متداخلة (Nested Fields) مثل اللغات (`languages`)، العملات (`currencies`)، والحدود (`borders`). المطلوب فك تداخل هذه الحقول وتحويلها إلى بنية جدولية قابلة للاستخدام (Tabular Structure) مثل `DataFrame`.

**الخطوة الثانية: فحص البيانات (Data Profiling)**
* **المطلوب:** فحص البيانات الخام (Raw Data) بعد تحويلها لجدول، ويشمل ذلك 3 عناصر:
  1. **الإبلاغ عن مشاكل الجودة (Quality Issues):** تحديد وتوثيق المشاكل مثل القيم المفقودة (`Missing Values`).
  2. **أنواع البيانات (Data Types):** التحقق من نوع البيانات في كل عمود (نصوص، أرقام صحيحة، أرقام عشرية).
  3. **الإحصائيات الوصفية (Summary Statistics):** استخراج إحصائيات عامة مثل توزيع السكان والمساحة (`population` & `area`) وتعداد المناطق (`region counts`).

---

## 3. الكود اشتغل ولكن هو يطلب جلب جميع الدول، لماذا يظهر فقط 25 دولة؟

### الإجابة:
السبب هو نظام **تقسيم الصفحات (Pagination)** في الـ API؛ حيث يُرجع الخادم 25 دولة كحد أقصى في الصفحة الواحدة. كما يجب التأكد من إزالة معامل البحث `?q=canada` من الرابط الأساسي.

**الحل باستخدام حلقة تكرار لجلب جميع الصفحات:**

```python
import requests
import pandas as pd

url = '[https://api.restcountries.com/countries/v5](https://api.restcountries.com/countries/v5)'
headers = {'Authorization': 'Bearer rc_live_200497a641014a519165cfe2be30e21d'}

all_countries = []
page = 1

while True:
    response = requests.get(url, headers=headers, params={'page': page})
    data = response.json()
    
    objects = data.get('data', {}).get('objects', [])
    if not objects:
        break
        
    all_countries.extend(objects)
    page += 1

df = pd.json_normalize(all_countries)
print(f"إجمالي عدد الدول التي تم جلبها: {len(df)}")
```

---

## 4. لماذا هذا الكود (الذي يعتمد على `more_data` و `offset`) لا يجلب كامل الدول؟

### الإجابة:
يتوقف الكود عند أول 25 دولة لأن شرط استمرار الحلقة (`more_data`) يتحول إلى `False` مباشرة بعد أول طلب للأسباب التالية:
1. **اختلاف موقع أو اسم مفتاح `'meta'`:** البحث في الجذر عبر `res_json.get('meta', {})` يُرجع قاموساً فارغاً `{}` إذا كانت بيانات الترقيم موجودة داخل `'data'` أو بأسماء مختلفة، وبالتالي ترجع `meta.get('more', False)` القيمة `False` وتتوقف الحلقة فوراً.
2. **صياغة معاملات الرابط:** استخدام `params` في `requests.get` أكثر أماناً من دمج النصوص يدوياً.

**الحل المعدّل:**

```python
raw_countries = []
offset = 0
limit = 25

while True:
    response = requests.get(url, headers=headers, params={'offset': offset, 'limit': limit})
    res_json = response.json()
    
    current_page_countries = res_json.get('data', {}).get('objects', [])
    
    # التوقف فقط إذا عادت القائمة فارغة
    if not current_page_countries:
        break
        
    raw_countries.extend(current_page_countries)
    
    meta = res_json.get('meta') or res_json.get('data', {}).get('meta', {})
    if meta and meta.get('more') is False:
        break
        
    offset += meta.get('limit', limit)

print(f"العدد الكلي للدول: {len(raw_countries)}")
```

---

## 5. هل هذه خطوة صحيحة لعمل Cleaning للكود (استخدام دوال `flatten_*` واختيار `important_columns`)؟

### الإجابة:
هذا الكود ممتاز ويُغطي **الخطوة الأولى (Extraction & Flattening)** بنجاح، لكنه لا يمثل خطوة **التنظيف الكاملة (Data Cleaning)** المطلوبة في المتطلب الثالث.

**ملاحظات لتحسين كود التسطيح الحالي:**
* **إرجاع `None` بدلاً من `""`:** إرجاع نص فارغ `""` يجعل `Pandas` تعتبر الخلية ممتلئة بنص فلا تظهر في إحصائيات القيم المفقودة (`isnull()`).
* **حماية دالة `.join()`:** التأكد من وجود قيمة للاسم قبل إضافته لتجنب خطأ `TypeError` إذا كان الاسم `None`.

**ما ينقص لإكمال الفحص (Profiling) والتنظيف (Cleaning):**
1. **الفحص قبل التنظيف:** تشغيل `df_final.info()`، و `df_final.isnull().sum()`، و `df_final.describe()`، و `df_final['region'].value_counts()`.
2. **التنظيف مع التبرير (Drop / Impute / Flag):** اختيار استراتيجية لمعالجة القيم المفقودة وتبريرها في التعليقات (مثل وضع علامة Flag للدول الجزرية التي لا تملك حدوداً برية، أو فحص الأقاليم غير المأهولة).

---

## 6. لماذا لا يظهر `missing values` عند فحص الجدول؟

### الإجابة:
يرجع ذلك لسببين تقنيين:
1. **أولوية المعاملات (Operator Precedence):** في بايثون، يمتلك معامل الربط `|` أولوية أعلى من `==`، لذا فإن كتابة `df[col].isnull() | df[col] == ""` بدون أقواس تعطي نتيجة خاطئة (`False`). يجب وضع كل شرط داخل أقواس: `(df[col].isnull()) | (df[col] == "")`.
2. **النصوص الفارغة `""`:** دالة `isnull()` لا تعتبر النص الفارغ `""` قيمة مفقودة؛ يجب تحويل `""` إلى `np.nan` أولاً.

**الكود الصحيح:**

```python
import numpy as np

df_final = df[important_columns].copy()

# 1. تحويل النصوص الفارغة إلى قيم مفقودة حقيقية (NaN)
df_final.replace("", np.nan, inplace=True)

# 2. طباعة عدد القيم المفقودة الحقيقية
print(df_final.isnull().sum())

# 3. تطبيق طريقة Flagging بشكل صحيح
original_columns = list(df_final.columns)
for column in original_columns:
    df_final[column + '_missing'] = df_final[column].isnull()

# حفظ الملف بدون عمود الفهرس الزائد
df_final.to_csv("countries_clean.csv", index=False)
```

---

## 7. ما وظيفة `index_col` مع `pd.read_csv`؟

### الإجابة:
تُستخدم الخاصية `index_col` لتحديد العمود الذي تريد اعتماده **كفهرس للصفوف (Index)** بدلاً من الترقيم التلقائي (`0, 1, 2, ...`):

* **التخلص من عمود `Unnamed: 0`:** عند قراءة ملف CSV حُفظ سابقاً بدون `index=False`:
  ```python
  df = pd.read_csv("countries_clean.csv", index_col=0)
  ```
* **استخدام عمود بالاسم كفهرس:** لتسهيل الوصول لصفوف محددة عبر `.loc`:
  ```python
  df = pd.read_csv("countries_clean.csv", index_col="names.common")
  ```
* **إنشاء فهرس مركب (MultiIndex):** عبر تمرير قائمة بأسماء عدة أعمدة:
  ```python
  df = pd.read_csv("countries_clean.csv", index_col=["region", "names.common"])
  ```

---

## 8. ما سبب الخطوط السوداء العمودية أعلى الأعمدة في رسم `sns.barplot`؟

### الإجابة:
هذه الخطوط هي **أشرطة الخطأ (Error Bars)**، وتُظهرها دالة `sns.barplot` افتراضياً لتمثيل **فترة الثقة بنسبة 95% (95% Confidence Interval)** حول المتوسط الحسابي، وتوضح مدى تشتت القيم داخل كل منطقة (`region`).

**كيفية إزالتها:**
أضف المعامل `errorbar=None` (أو `ci=None` في الإصدارات القديمة):

```python
sns.barplot(data=df_final, x="region", y="num_languages", hue="region", errorbar=None)
```

**تحسين دالة حساب عدد اللغات لتجنب احتساب `NaN` كلغة:**

```python
def count_languages(country):
    if pd.isnull(country) or country == "":
        return 0
    return len(str(country).split(","))
```
.
