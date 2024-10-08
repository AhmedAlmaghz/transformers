# mLUKE

## نظرة عامة

اقترح نموذج mLUKE في [mLUKE: قوة تمثيلات الكيانات في نماذج اللغة متعددة اللغات المُدربة مسبقًا](https://arxiv.org/abs/2110.08151) بواسطة Ryokan Ri وIkuya Yamada وYoshimasa Tsuruoka. إنه امتداد متعدد اللغات لنموذج [LUKE](https://arxiv.org/abs/2010.01057) تم تدريبه بناءً على XLM-RoBERTa.

يستند النموذج إلى XLM-RoBERTa ويضيف تمثيلات الكيانات (entity embeddings)، مما يساعد في تحسين الأداء في مهام مختلفة أسفل النموذج تتضمن الاستدلال حول الكيانات مثل التعرف على الكيانات المسماة، والإجابة على الأسئلة الاستخراجية، وتصنيف العلاقات، واستكمال المعرفة على طريقة Cloze.

 فيما يلي الملخص من الورقة البحثية:

*أظهرت الدراسات الحديثة أن نماذج اللغة متعددة اللغات المُدربة مسبقًا يمكن تحسينها بشكل فعال باستخدام معلومات محاذاة عبر اللغات من كيانات Wikipedia. ومع ذلك، لا تستغل الطرق الحالية سوى معلومات الكيانات في التدريب المُسبق ولا تستخدم الكيانات بشكل صريح في المهام أسفل النموذج. في هذه الدراسة، نستكشف فعالية الاستفادة من تمثيلات الكيانات للمهام عبر اللغات أسفل النموذج. نقوم بتدريب نموذج لغة متعدد اللغات مع 24 لغة بتمثيلات الكيانات ونظهر أن النموذج يتفوق باستمرار على النماذج المستندة إلى الكلمات في مهام نقل اللغة المختلفة. كما نقوم بتحليل النموذج وتتمثل الفكرة الرئيسية في أن دمج تمثيلات الكيانات في المدخلات يسمح لنا باستخراج ميزات أكثر عمومية للغة. نقوم أيضًا بتقييم النموذج باستخدام مهمة استكمال موجهة متعددة اللغات باستخدام مجموعة بيانات mLAMA. ونظهر أن الموجه القائم على الكيانات يستحضر المعرفة الوقائعية الصحيحة أكثر من استخدام تمثيلات الكلمات فقط.*

تمت المساهمة بهذا النموذج بواسطة [ryo0634](https://huggingface.co/ryo0634). يمكن العثور على الكود الأصلي [هنا](https://github.com/studio-ousia/luke).

## نصائح الاستخدام

يمكن توصيل أوزان mLUKE مباشرة في نموذج LUKE، كما هو موضح أدناه:

```python
from transformers import LukeModel

model = LukeModel.from_pretrained("studio-ousia/mluke-base")
```

لاحظ أن mLUKE لديه محدد الرموز الخاص به، [`MLukeTokenizer`]. يمكن تهيئته على النحو التالي:

```python
from transformers import MLukeTokenizer

tokenizer = MLukeTokenizer.from_pretrained("studio-ousia/mluke-base")
```

<Tip>
نظرًا لأن بنية mLUKE تعادل بنية LUKE، فيمكن الرجوع إلى [صفحة وثائق LUKE](luke) لجميع النصائح وأمثلة الكود والمفكرات.
</Tip>

## MLukeTokenizer

[[autodoc]] MLukeTokenizer

- __call__

- save_vocabulary