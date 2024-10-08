# استخدام خطوط الأنابيب لخادم ويب 

<Tip>

إن إنشاء محرك استدلال موضوع معقد، وقد يعتمد "أفضل" حل على مساحة المشكلة لديك. هل أنت على وحدة المعالجة المركزية أو وحدة معالجة الرسومات؟ هل تريد أدنى مستوى من التأخير، أو أعلى مستوى من الإنتاجية، أو دعم العديد من النماذج، أو فقط تحسين نموذج واحد محدد؟ هناك العديد من الطرق لتناول هذا الموضوع، لذا فإن ما سنقدمه هو إعداد افتراضي جيد للبدء، ولكنه قد لا يكون الحل الأمثل بالنسبة لك. 

</Tip> 

الشيء الرئيسي الذي يجب فهمه هو أننا يمكن أن نستخدم مؤشرًا، تمامًا كما تفعل [على مجموعة بيانات](pipeline_tutorial#using-pipelines-on-a-dataset)، نظرًا لأن خادم الويب هو أساسًا نظام ينتظر الطلبات ويعالجها عند استلامها. 

عادةً ما تكون خوادم الويب متعددة الإرسال (متعددة الخيوط، غير متزامنة، إلخ) للتعامل مع الطلبات المختلفة بشكل متزامن. من ناحية أخرى، فإن خطوط الأنابيب (وبشكل رئيسي النماذج الأساسية) ليست رائعة حقًا للتوازي؛ حيث تستهلك الكثير من ذاكرة الوصول العشوائي، لذا من الأفضل منحها جميع الموارد المتاحة عندما تعمل أو أنها مهمة مكثفة الحساب. 

سنحل ذلك من خلال جعل خادم الويب يتعامل مع الحمل الخفيف لاستقبال الطلبات وإرسالها، ووجود خيط واحد للتعامل مع العمل الفعلي. سيستخدم هذا المثال `starlette`. إطار العمل الفعلي ليس مهمًا حقًا، ولكن قد يتعين عليك ضبط الكود أو تغييره إذا كنت تستخدم إطار عمل آخر لتحقيق نفس التأثير. 

أنشئ `server.py`: 

```py
from starlette.applications import Starlette
from starlette.responses import JSONResponse
from starlette.routing import Route
from transformers import pipeline
import asyncio


async def homepage(request):
    payload = await request.body()
    string = payload.decode("utf-8")
    response_q = asyncio.Queue()
    await request.app.model_queue.put((string, response_q))
    output = await response_q.get()
    return JSONResponse(output)


async def server_loop(q):
    pipe = pipeline(model="google-bert/bert-base-uncased")
    while True:
        (string, response_q) = await q.get()
        out = pipe(string)
        await response_q.put(out)


app = Starlette(
    routes=[
        Route("/", homepage, methods=["POST"]),
    ],
)


@app.on_event("startup")
async def startup_event():
    q = asyncio.Queue()
    app.model_queue = q
    asyncio.create_task(server_loop(q))
```

الآن يمكنك تشغيله باستخدام: 

```bash
uvicorn server:app
```

ويمكنك الاستعلام عنه: 

```bash
curl -X POST -d "test [MASK]" http://localhost:8000/
#[{"score":0.7742936015129089,"token":1012,"token_str":".","sequence":"test."},...]
```

وهكذا، لديك الآن فكرة جيدة عن كيفية إنشاء خادم ويب! 

ما هو مهم حقًا هو أننا نحمل النموذج **مرة واحدة** فقط، لذلك لا توجد نسخ من النموذج على خادم الويب. بهذه الطريقة، لا يتم استخدام ذاكرة الوصول العشوائي غير الضرورية. تسمح آلية وضع الطابور بالقيام بأشياء متقدمة مثل ربما تجميع بعض العناصر قبل الاستدلال لاستخدام الدفعات الديناميكية: 

<Tip warning={true}>

تم كتابة عينة الكود أدناه بشكل مقصود مثل كود وهمي للقراءة. لا تقم بتشغيله دون التحقق مما إذا كان منطقيًا لموارد النظام الخاص بك! 

</Tip> 

```py
(string, rq) = await q.get()
strings = []
queues = []
while True:
    try:
        (string, rq) = await asyncio.wait_for(q.get(), timeout=0.001) # 1ms
    except asyncio.exceptions.TimeoutError:
        break
    strings.append(string)
    queues.append(rq)
strings
outs = pipe(strings, batch_size=len(strings))
for rq, out in zip(queues, outs):
    await rq.put(out)
```

مرة أخرى، تم تحسين الكود المقترح للقراءة، وليس ليكون أفضل كود. أولاً، لا يوجد حد لحجم الدفعة، والذي عادةً ما لا يكون فكرة عظيمة. بعد ذلك، يتم إعادة تعيين المهلة في كل عملية استرداد للصف، مما يعني أنه قد يتعين عليك الانتظار لفترة أطول بكثير من 1 مللي ثانية قبل تشغيل الاستدلال (تأخير الطلب الأول بهذا القدر). 

سيكون من الأفضل وجود موعد نهائي واحد لمدة 1 مللي ثانية. 

سيظل هذا ينتظر دائمًا لمدة 1 مللي ثانية حتى إذا كان الطابور فارغًا، والذي قد لا يكون الأفضل نظرًا لأنك تريد على الأرجح البدء في إجراء الاستدلال إذا لم يكن هناك شيء في الطابور. ولكن ربما يكون منطقيًا إذا كانت الدفعات مهمة حقًا لحالتك الاستخدام. مرة أخرى، لا يوجد حل واحد الأفضل. 

## بعض الأشياء التي قد ترغب في مراعاتها 

### التحقق من الأخطاء 

هناك الكثير مما يمكن أن يسير على ما يرام في الإنتاج: نفاد الذاكرة، أو نفاد المساحة، أو قد يفشل تحميل النموذج، أو قد يكون الاستعلام خاطئًا، أو قد يكون الاستعلام صحيحًا ولكنه يفشل في التشغيل بسبب إعداد نموذج غير صحيح، وهكذا. 

من الجيد بشكل عام إذا قام الخادم بإخراج الأخطاء إلى المستخدم، لذا فإن إضافة الكثير من عبارات `try..except` لعرض هذه الأخطاء فكرة جيدة. ولكن ضع في اعتبارك أنه قد يكون أيضًا خطرًا أمنيًا للكشف عن جميع هذه الأخطاء اعتمادًا على سياق الأمان الخاص بك. 

### كسر الدائرة 

عادةً ما تبدو خوادم الويب أفضل عندما تقوم بكسر الدائرة. وهذا يعني أنها تعيد أخطاء صحيحة عندما تكون مثقلة بالأعباء بدلاً من الانتظار إلى أجل غير مسمى للاستعلام. قم بإرجاع خطأ 503 بدلاً من الانتظار لفترة طويلة جدًا أو 504 بعد فترة طويلة. 

من السهل نسبيًا تنفيذ ذلك في الكود المقترح نظرًا لوجود طابور واحد. إن النظر في حجم الطابور هو طريقة أساسية لبدء إعادة الأخطاء قبل فشل خادم الويب بسبب التحميل الزائد. 

### حظر الخيط الرئيسي 

حاليًا، PyTorch غير مدرك للأساليب غير المتزامنة، وسيؤدي الحساب إلى حظر الخيط الرئيسي أثناء تشغيله. وهذا يعني أنه سيكون من الأفضل إذا تم إجبار PyTorch على التشغيل على خيط/عملية خاصة به. لم يتم ذلك هنا لأن الكود أكثر تعقيدًا (في الغالب لأن الخيوط والأساليب غير المتزامنة والطوابير لا تتوافق معًا). ولكن في النهاية، فإنه يؤدي نفس الوظيفة. 

سيكون هذا مهمًا إذا كان الاستدلال للعناصر الفردية طويلًا (> 1 ثانية) لأنه في هذه الحالة، يجب أن ينتظر كل استعلام أثناء الاستدلال لمدة 1 ثانية قبل حتى تلقي خطأ. 

### الدفعات الديناميكية 

بشكل عام، قد لا يكون التجميع تحسينًا بالمقارنة مع تمرير عنصر واحد في كل مرة (راجع [تفاصيل الدفعات](./main_classes/pipelines#pipeline-batching) لمزيد من المعلومات). ولكنه يمكن أن يكون فعالًا جدًا عند استخدامه في الإعداد الصحيح. في واجهة برمجة التطبيقات، لا يوجد تجميع ديناميكي بشكل افتراضي (الكثير من الفرص لتباطؤ). ولكن بالنسبة لاستدلال BLOOM - وهو نموذج كبير جدًا - فإن التجميع الديناميكي **أساسي** لتوفير تجربة جيدة للجميع.