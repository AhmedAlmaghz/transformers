# التدريب الفعال على وحدات معالجة الرسومات (GPUs) متعددة

إذا كان تدريب نموذج على وحدة معالجة رسومات (GPU) واحدة بطيئًا جدًا أو إذا لم تتسع ذاكرة وحدة المعالجة الرسومية لوزن النموذج، فقد يكون الانتقال إلى إعداد متعدد وحدات معالجة الرسومات خيارًا قابلًا للتطبيق. قبل إجراء هذا الانتقال، قم باستكشاف جميع الاستراتيجيات المشمولة في "أساليب وأدوات التدريب الفعال على وحدة معالجة رسومات واحدة" بشكل شامل، حيث أنها تنطبق عالميًا على تدريب النماذج على أي عدد من وحدات معالجة الرسومات. بمجرد استخدامك لهذه الاستراتيجيات ووجدت أنها غير كافية لحالتك على وحدة معالجة رسومات واحدة، فكر في الانتقال إلى وحدات معالجة الرسومات المتعددة.

يتطلب الانتقال من وحدة معالجة رسومات واحدة إلى وحدات معالجة الرسومات المتعددة تقديم بعض أشكال التوازي، حيث يجب توزيع عبء العمل عبر الموارد. يمكن استخدام تقنيات متعددة لتحقيق التوازي، مثل التوازي في البيانات، والتوازي في المصفوفات، والتوازي في الأنابيب. ومن المهم ملاحظة أنه لا يوجد حل واحد يناسب الجميع، وأن الإعدادات المثالية تعتمد على تكوين الأجهزة المحدد الذي تستخدمه.

يقدم هذا الدليل نظرة متعمقة على أنواع فردية من التوازي، بالإضافة إلى توجيهات حول طرق الجمع بين التقنيات واختيار النهج المناسب. للحصول على دروس خطوة بخطوة حول التدريب الموزع، يرجى الرجوع إلى وثائق 🤗 Accelerate.

على الرغم من أن المفاهيم الرئيسية التي تمت مناقشتها في هذا الدليل تنطبق على الأطر المختلفة، إلا أننا نركز هنا على عمليات التنفيذ المستندة إلى PyTorch.

قبل الغوص في تفاصيل كل تقنية، دعنا نلقي نظرة على عملية اتخاذ القرار عند تدريب نماذج كبيرة على بنية تحتية كبيرة.

## استراتيجية قابلية التوسع

ابدأ بتقدير مقدار ذاكرة الوصول العشوائي الظاهري (vRAM) المطلوبة لتدريب نموذجك. بالنسبة للنماذج المستضافة على 🤗 Hub، استخدم "حاسبة ذاكرة النموذج" الخاصة بنا، والتي توفر لك حسابات دقيقة ضمن هامش بنسبة 5%.

**استراتيجية الموازاة لعقدة واحدة / إعداد متعدد وحدات معالجة الرسومات**

عند تدريب نموذج على عقدة واحدة مع وحدات معالجة الرسومات المتعددة، يمكن أن يؤثر اختيارك لاستراتيجية الموازاة بشكل كبير على الأداء. فيما يلي تفصيل لخياراتك:

**الحالة 1: يناسب نموذجك وحدة معالجة رسومات واحدة**

إذا كان نموذجك يناسب وحدة معالجة الرسومات الواحدة بشكل مريح، فهناك خياران رئيسيان:

1. DDP - Distributed DataParallel
2. Zero Redundancy Optimizer (ZeRO) - اعتمادًا على الوضع والتكوين المستخدم، قد تكون هذه الطريقة أسرع أو لا، ولكن من الجدير تجربتها.

**الحالة 2: لا يناسب نموذجك وحدة معالجة رسومات واحدة**

إذا كان نموذجك أكبر من وحدة معالجة الرسومات الواحدة، فهناك العديد من البدائل التي يجب مراعاتها:

1. PipelineParallel (PP)
2. ZeRO
3. TensorParallel (TP)

مع اتصال سريع بين العقد (مثل NVLINK أو NVSwitch)، يجب أن تؤدي جميع الاستراتيجيات الثلاث (PP وZeRO وTP) إلى أداء مماثل. ومع ذلك، بدون هذه الميزات، سيكون PP أسرع من TP أو ZeRO. قد يُحدث أيضًا درجة TP فرقًا. من الأفضل تجربة إعدادك المحدد لتحديد الاستراتيجية الأنسب.

يتم استخدام TP دائمًا تقريبًا داخل عقدة واحدة. وهذا يعني أن حجم TP يساوي أو أقل من وحدات معالجة الرسومات لكل عقدة.

**الحالة 3: لا تناسب الطبقة الأكبر في نموذجك وحدة معالجة رسومات واحدة**

1. إذا كنت لا تستخدم ZeRO، فيجب عليك استخدام TensorParallel (TP)، لأن PipelineParallel (PP) وحدها لن تكون كافية لاستيعاب الطبقة الكبيرة.
2. إذا كنت تستخدم ZeRO، فاعتمد أيضًا تقنيات من "أساليب وأدوات التدريب الفعال على وحدة معالجة رسومات واحدة".

**استراتيجية الموازاة لعقدة متعددة / إعداد متعدد وحدات معالجة الرسومات**

* عند وجود اتصال سريع بين العقد (مثل NVLINK أو NVSwitch)، فكر في استخدام أحد الخيارات التالية:

    1. ZeRO - لأنه يتطلب إجراء تعديلات طفيفة على النموذج تقريبًا
    2. مزيج من PipelineParallel (PP) مع TensorParallel (TP) وDataParallel (DP) - سيؤدي هذا النهج إلى تقليل الاتصالات، ولكنه يتطلب إجراء تغييرات كبيرة على النموذج

* عند وجود اتصال بطيء بين العقد ولا تزال ذاكرة وحدة معالجة الرسومات منخفضة:

    1. استخدم مزيجًا من DataParallel (DP) مع PipelineParallel (PP) وTensorParallel (TP) وZeRO.

في الأقسام التالية من هذا الدليل، سنغوص بشكل أعمق في كيفية عمل أساليب التوازي المختلفة هذه.

## التوازي في البيانات

حتى مع وجود وحدتي معالجة رسومات فقط، يمكنك الاستفادة بسهولة من قدرات التدريب المعجلة التي توفرها ميزات PyTorch المدمجة، مثل `DataParallel` (DP) و`DistributedDataParallel` (DDP). لاحظ أن وثائق PyTorch توصي باستخدام `DistributedDataParallel` (DDP) بدلاً من `DataParallel` (DP) للتدريب على وحدات معالجة الرسومات المتعددة لأنه يعمل مع جميع النماذج. دعونا نلقي نظرة على كيفية عمل هاتين الطريقتين وما الذي يميزهما.

### DataParallel مقابل DistributedDataParallel

لفهم الاختلافات الرئيسية في عبء الاتصال بين وحدات معالجة الرسومات بين الطريقتين، دعنا نراجع العمليات لكل دفعة:

[DDP]:

- في وقت البدء، تقوم العملية الرئيسية بنسخ النموذج مرة واحدة من وحدة معالجة الرسومات 0 إلى باقي وحدات معالجة الرسومات
- بعد ذلك لكل دفعة:
    1. تستهلك كل وحدة معالجة رسومات مباشرة دفعة البيانات المصغرة الخاصة بها.
    2. أثناء `backward`، بمجرد أن تصبح التدرجات المحلية جاهزة، يتم حساب متوسطها عبر جميع العمليات.

[DP]:

لكل دفعة:
    1. تقرأ وحدة معالجة الرسومات 0 دفعة البيانات وترسل دفعة مصغرة إلى كل وحدة معالجة رسومات.
    2. يتم نسخ النموذج المحدث من وحدة معالجة الرسومات 0 إلى كل وحدة معالجة رسومات.
    3. يتم تنفيذ `forward`، ويتم إرسال الإخراج من كل وحدة معالجة رسومات إلى وحدة معالجة الرسومات 0 لحساب الخسارة.
    4. يتم توزيع الخسارة من وحدة معالجة الرسومات 0 إلى جميع وحدات معالجة الرسومات، ويتم تشغيل `backward`.
    5. يتم إرسال التدرجات من كل وحدة معالجة رسومات إلى وحدة معالجة الرسومات 0 ويتم حساب متوسطها.

تشمل الاختلافات الرئيسية ما يلي:

1. يقوم DDP بتنفيذ عملية اتصال واحدة فقط لكل دفعة - إرسال التدرجات، في حين أن DP يقوم بخمسة تبادلات بيانات مختلفة لكل دفعة. يقوم DDP بنسخ البيانات باستخدام [torch.distributed] https://pytorch.org/docs/master/distributed.html)، في حين أن DP ينسخ البيانات داخل العملية عبر خيوط Python (والتي تقدم قيودًا مرتبطة بـ GIL). ونتيجة لذلك، فإن "DistributedDataParallel" (DDP) أسرع بشكل عام من "DataParallel" (DP) ما لم يكن لديك اتصال بطيء بين بطاقات وحدات معالجة الرسومات.
2. في DP، تقوم وحدة معالجة الرسومات 0 بتنفيذ قدر أكبر من العمل مقارنة بوحدات معالجة الرسومات الأخرى، مما يؤدي إلى انخفاض استخدام وحدة معالجة الرسومات.
3. يدعم DDP التدريب الموزع عبر أجهزة متعددة، في حين أن DP لا يدعم ذلك.

هذه ليست قائمة شاملة بالاختلافات بين DP وDDP، ولكن الدقائق الأخرى خارج نطاق هذا الدليل. يمكنك الحصول على فهم أعمق لهذه الطرق من خلال قراءة هذه المقالة.

دعونا نوضح الاختلافات بين DP وDDP من خلال تجربة. سنقوم باختبار أداء DP وDDP مع إضافة سياق وجود NVLink:

* الأجهزة: 2x TITAN RTX 24 جيجابايت لكل منها + NVlink مع 2 NVLinks (`NV2` في `nvidia-smi topo -m`).
* البرمجيات: `pytorch-1.8-to-be` + `cuda-11.0` / `transformers==4.3.0.dev0`.

لإيقاف ميزة NVLink في أحد الاختبارات، نستخدم `NCCL_P2P_DISABLE=1`.

فيما يلي كود الاختبار وإخراجها:

**DP**

```bash
rm -r /tmp/test-clm; CUDA_VISIBLE_DEVICES=0,1 \
python examples/pytorch/language-modeling/run_clm.py \
--model_name_or_path openai-community/gpt2 --dataset_name wikitext --dataset_config_name wikitext-2-raw-v1 \
--do_train --output_dir /tmp/test-clm --per_device_train_batch_size 4 --max_steps 200

{'train_runtime': 110.5948, 'train_samples_per_second': 1.808, 'epoch': 0.69}
```

**DDP مع NVlink**

```bash
rm -r /tmp/test-clm; CUDA_VISIBLE_DEVICES=0,1 \
torchrun --nproc_per_node 2 examples/pytorch/language-modeling/run_clm.py \
--model_name_or_path openai-community/gpt2 --dataset_name wikitext --dataset_config_name wikitext-2-raw-v1 \
--do_train --output_dir /tmp/test-clm --per_device_train_batch_size 4 --max_steps 200

{'train_runtime': 101.9003, 'train_samples_per_second': 1.963, 'epoch': 0.69}
```

**DDP بدون NVlink**

```bash
rm -r /tmp/test-clm; NCCL_P2P_DISABLE=1 CUDA_VISIBLE_DEVICES=0,1 \
torchrun --nproc_per_node 2 examples/pytorch/language-modeling/run_clm.py \
--model_name_or_path openai-community/gpt2 --dataset_name wikitext --dataset_config_name wikitext-2-raw-v1 \
--do_train --output_dir /tmp/test-clm --per_device_train_batch_size 4 --max_steps 200

{'train_runtime': 131.4367, 'train_samples_per_second': 1.522, 'epoch': 0.69}
```

فيما يلي نتائج الاختبار نفسها مجمعة في جدول للراحة:

| النوع | NVlink | الوقت |
| :----- | -----  | ---: |
| 2:DP | Y | 110s |
| 2:DDP | Y | 101s |
| 2:DDP | N | 131s |

كما ترون، في هذه الحالة، DP أبطأ بنسبة 10% تقريبًا من DDP مع NVlink، ولكنه أسرع بنسبة 15% من DDP بدون NVlink. سيعتمد الاختلاف الحقيقي على مقدار البيانات التي تحتاج كل وحدة معالجة رسومات إلى مزامنتها مع الوحدات الأخرى - فكلما زادت البيانات التي يجب مزامنتها، كلما أعاق الرابط البطيء وقت التشغيل الإجمالي.

## التوازي في البيانات باستخدام ZeRO

يتم توضيح التوازي في البيانات المدعوم من ZeRO (ZeRO-DP) في الرسم البياني التالي من هذه التدوينة.

على الرغم من أنه قد يبدو معقدًا، إلا أنه مفهوم مشابه جدًا لـ `DataParallel` (DP). الفرق هو أنه بدلاً من نسخ معلمات النموذج الكاملة والتدرجات وحالات المحسن، تقوم كل وحدة معالجة رسومات بتخزين شريحة فقط منها. بعد ذلك، في وقت التشغيل عندما تكون معلمات الطبقة الكاملة مطلوبة فقط للطبقة المعطاة، تتم مزامنة جميع وحدات معالجة الرسومات لإعطاء بعضها البعض الأجزاء التي تفتقدها.

لتوضيح هذه الفكرة، ضع في اعتبارك نموذجًا بسيطًا يحتوي على 3 طبقات (La وLb وLc)، حيث تحتوي كل طبقة على 3 معلمات. على سبيل المثال، تحتوي الطبقة La على الأوزان a0 وa1 وa2:

```
La | Lb | Lc
---|----|---
a0 | b0 | c0
a1 | b1 | c1
a2 | b2 | c2
```

إذا كان لدينا 3 وحدات معالجة رسومات، فإن ZeRO-DP يقسم النموذج إلى 3 وحدات معالجة رسومات كما يلي:

```
GPU0:
La | Lb | Lc
---|----|---
a0 | b0 | c0

GPU1:
La | Lb | Lc
---|----|---
a1 | b1 | c1

GPU2:
La | Lb | Lc
---|----|---
a2 | b2 | c2
```

وبطريقة ما، هذا هو نفس التقطيع الأفقي مثل التوازي في المصفوفات، على عكس التقطيع الرأسي، حيث يتم وضع مجموعات الطبقات الكاملة على وحدات معالجة الرسومات المختلفة. الآن دعونا نرى كيف يعمل هذا:

ستحصل كل من وحدات معالجة الرسومات هذه على الدفعة المصغرة المعتادة كما هو الحال في DP:

```
x0 => GPU0
x1 => GPU1
x2 => GPU2
```

يتم تمرير المدخلات دون تعديلات كما لو كانت ستتم معالجتها بواسطة النموذج الأصلي.

أولاً، تصل المدخلات إلى الطبقة "La". ماذا يحدث في هذه المرحلة؟

على وحدة معالجة الرسومات 0: تتطلب الدفعة المصغرة x0 المعلمات a0 وa1 وa2 للانتقال عبر المسار الأمامي للطبقة، ولكن تحتوي وحدة معالجة الرسومات 0 على a0 فقط. سيحصل على a1 من وحدة معالجة الرسومات 1 وa2 من وحدة معالجة الرسومات 2، مما يجمع بين جميع قطع النموذج.

بالتوازي، تحصل وحدة معالجة الرسومات 1 على دفعة مصغرة أخرى - x1. تحتوي وحدة معالجة الرسومات 1 على المعلمة a1، ولكنها تحتاج إلى a0 وa2، لذا فهي تحصل عليها من وحدة معالجة الرسومات 0 ووحدة معالجة الرسومات 2.

ينطبق الشيء نفسه على وحدة معالجة الرسومات 2 التي تحصل على الدفعة المصغرة x2. تحصل على a0 وa1 من وحدة معالجة الرسومات 0 ووحدة معالجة الرسومات 1.

بهذه الطريقة، تحصل كل من وحدات معالجة الرسومات الثلاث على المصفوفات الكاملة المعاد بناؤها وتنفذ عملية انتقال أمامي بمجموعة البيانات المصغرة الخاصة بها.

بمجرد الانتهاء من الحساب، يتم إسقاط البيانات التي لم تعد هناك حاجة إليها - يتم استخدامها فقط أثناء الحساب. يتم تنفيذ إعادة البناء بكفاءة عبر الاسترداد المسبق.

بعد ذلك، يتم تكرار العملية بأكملها للطبقة Lb، ثم Lc للأمام، ثم Lc -> Lb -> La للخلف.

هذه الآلية مشابهة لاستراتيجية التخييم الجماعي الفعالة: يحمل الشخص A الخيمة، ويحمل الشخص B الموقد، ويحمل الشخص C الفأس. كل ليلة يتشاركون ما لديهم مع الآخرين ويحصلون على ما يحتاجون إليه من الآخرين، وفي الصباح يحزمون معداتهم المخصصة ويستمرون في طريقهم. هذا هو ما يفعله ZeRO DP/Sharded DDP.

قارن هذه الاستراتيجية بالاستراتيجية البسيطة حيث يتعين على كل شخص حمل خيمته وموقده وفأسه (مشابه لـ DataParallel (DP وDDP) في PyTorch)، والتي ستكون أقل كفاءة بكثير.

بينما تقرأ الأدبيات حول هذا الموضوع، قد تصادف المرادفات التالية: مجزأة، مجزأة.

إذا كنت تولي اهتمامًا وثيقًا للطريقة التي يقسم بها ZeRO أوزان النموذج، فستبدو مشابهة جدًا للتوازي في المصفوفات والذي سيتم مناقشته لاحقًا. ويرجع ذلك إلى أنه يقسم/يشظي أوزان كل طبقة، على عكس التوازي في النموذج الرأسي الذي تمت مناقشته لاحقًا.

عمليات التنفيذ:

- DeepSpeed ZeRO-DP المراحل 1+2+
## النموذج الساذج للتوازي:

عندما تنتقل البيانات من الطبقة 0 إلى الطبقة 3، لا يختلف الأمر عن المرور الأمامي العادي. ومع ذلك، يتطلب تمرير البيانات من الطبقة 3 إلى الطبقة 4 نقلها من GPU0 إلى GPU1، مما يؤدي إلى حدوث تكلفة إضافية في الاتصال. إذا كانت وحدات GPU المشاركة موجودة على نفس العقدة الحسابية (مثل نفس الجهاز المادي)، فإن عملية النسخ تكون سريعة، ولكن إذا كانت وحدات GPU موزعة على عدة عقد حسابية (مثل أجهزة متعددة)، فقد تكون التكلفة الإضافية للاتصال أكبر بكثير.

بعد ذلك، تعمل الطبقات من 4 إلى 7 كما هي في النموذج الأصلي. وعند الانتهاء من الطبقة السابعة، هناك حاجة في كثير من الأحيان إلى إرسال البيانات مرة أخرى إلى الطبقة 0 حيث توجد التسميات (أو بدلاً من ذلك إرسال التسميات إلى الطبقة الأخيرة). الآن يمكن حساب الخسارة ويمكن لمُحسن الخسارة أن يقوم بعمله.

يأتي النموذج الساذج للتوازي مع العديد من أوجه القصور:

- **جميع وحدات GPU ما عدا واحدة خاملة في أي لحظة معينة**: إذا تم استخدام 4 وحدات GPU، فهذا مشابه تقريبًا لمضاعفة ذاكرة GPU واحدة أربع مرات، وتجاهل بقية الأجهزة.

- **التكلفة الإضافية في نقل البيانات بين الأجهزة**: على سبيل المثال، يمكن لـ 4x 6GB cards استيعاب نفس الحجم مثل 1x 24GB card باستخدام النموذج الساذج للتوازي، ولكن بطاقة 24GB واحدة ستكمل التدريب بشكل أسرع، لأنها لا تحتوي على تكلفة نسخ البيانات. ولكن، على سبيل المثال، إذا كان لديك بطاقات 40 جيجابايت وتحتاج إلى ملاءمة نموذج 45 جيجابايت، فيمكنك ذلك باستخدام 4x 40GB cards (ولكن بالكاد بسبب حالة التدرج ومُحسن الخسارة).

- **نسخ التعليقات التوضيحية المشتركة**: قد تحتاج التعليقات التوضيحية المشتركة إلى نسخها ذهابًا وإيابًا بين وحدات GPU.

الآن بعد أن تعرفت على كيفية عمل النموذج الساذج للتوازي وعيوبه، دعنا نلقي نظرة على التوازي الأنبوبي (PP).

التوازي الأنبوبي متطابق تقريبًا مع النموذج الساذج للتوازي، ولكنه يحل مشكلة تعطيل وحدات GPU عن طريق تقسيم الدفعة الواردة إلى دفعات صغيرة وإنشاء خط أنابيب بشكل مصطنع، مما يسمح لوحدات GPU المختلفة بالمشاركة المتزامنة في عملية الحساب.

يوضح الرسم التوضيحي التالي من [ورقة GPipe](https://ai.googleblog.com/2019/03/introducing-gpipe-open-source-library.html) النموذج الساذج للتوازي في الأعلى، والتوازي الأنبوبي في الأسفل:

يمكنك ملاحظة أن التوازي الأنبوبي في أسفل الرسم التخطيطي يقلل من عدد المناطق الخاملة لوحدة GPU، المشار إليها باسم "الفقاعات". ويظهر كلا جزأي الرسم التخطيطي مستوىًا من التوازي من الدرجة 4، مما يعني أن هناك 4 وحدات GPU مشاركة في خط الأنابيب. ويمكنك أن ترى أن هناك مسارًا أماميًا مكونًا من 4 مراحل أنابيب (F0، F1، F2، وF3) يليها مسار عكسي بالترتيب المعاكس (B3، B2، B1، وB0).

يقدم التوازي الأنبوبي مُحسن خسارة جديدًا للضبط - "chunks"، والذي يحدد عدد قطع البيانات المرسلة في تسلسل عبر نفس مرحلة الأنبوب. على سبيل المثال، في الرسم التخطيطي السفلي، يمكنك رؤية "chunks=4". تقوم وحدة GPU0 بأداء نفس المسار الأمامي على القطعة 0 و1 و2 و3 (F0،0، F0،1، F0،2، F0،3) ثم تنتظر حتى تنتهي وحدات GPU الأخرى من عملها. ولا تبدأ وحدة GPU0 العمل مرة أخرى إلا عندما تبدأ وحدات GPU الأخرى في إكمال عملها، حيث تقوم بالمسار العكسي للقطع 3 و2 و1 و0 (B0،3، B0،2، B0،1، B0،0).

لاحظ أن هذا هو نفس مفهوم خطوات تجميع التدرجات. يستخدم PyTorch "chunks"، بينما يشير DeepSpeed إلى نفس مُحسن الخسارة باسم خطوات تجميع التدرجات.

بسبب "chunks"، يقدم التوازي الأنبوبي مفهوم الدفعات الصغيرة (MBS). ويقسم DP حجم الدفعة الإجمالية للبيانات إلى دفعات صغيرة، لذا إذا كان لديك درجة DP تساوي 4، يتم تقسيم حجم دفعة عالمية تساوي 1024 إلى 4 دفعات صغيرة يبلغ حجم كل منها 256 (1024/4). وإذا كان عدد "chunks" (أو GAS) هو 32، فإننا نحصل على حجم دفعة صغيرة يساوي 8 (256/32). وتعمل كل مرحلة من مراحل خط الأنابيب مع دفعة صغيرة واحدة في كل مرة. لحساب حجم الدفعة العالمية لإعداد DP + PP، استخدم الصيغة التالية: "mbs * chunks * dp_degree" ("8 * 32 * 4 = 1024").

مع "chunks=1"، ستحصل على النموذج الساذج للتوازي، وهو غير فعال. مع قيمة كبيرة من "chunks"، ستحصل على أحجام دفعات صغيرة جدًا وهو أيضًا غير فعال. لهذا السبب، نشجعك على تجربة قيمة "chunks" لإيجاد القيمة التي تؤدي إلى أكثر استخدام فعال لوحدات GPU.

قد تلاحظ وجود "فقاعة" من الوقت "الميت" في الرسم التخطيطي لا يمكن إجراء التوازي لها لأن مرحلة "التقدم" الأخيرة يجب أن تنتظر اكتمال مرحلة "العودة" لإكمال خط الأنابيب. والغرض من إيجاد أفضل قيمة لـ "chunks" هو تمكين الاستخدام المتزامن العالي لوحدة GPU عبر جميع وحدات GPU المشاركة، مما يؤدي إلى تقليل حجم "الفقاعة".

تم تنفيذ حلول واجهة برمجة التطبيقات (API) لخط الأنابيب في:

- PyTorch
- DeepSpeed
- Megatron-LM

تأتي هذه الحلول ببعض أوجه القصور:

- يجب أن تعدل النموذج بشكل كبير، لأن خط الأنابيب يتطلب إعادة كتابة التدفق العادي للوحدات النمطية إلى تسلسل "nn.Sequential" من نفس النوع، والذي قد يتطلب إجراء تغييرات على تصميم النموذج.

- حاليًا، واجهة برمجة تطبيقات خط الأنابيب مقيدة للغاية. إذا كان لديك مجموعة من المتغيرات في Python يتم تمريرها في المرحلة الأولى من خط الأنابيب، فسوف يتعين عليك إيجاد طريقة للتعامل مع ذلك. حاليًا، تتطلب واجهة خط الأنابيب إما Tensor واحدة أو مجموعة من Tensors كإدخال وإخراج وحيدين. يجب أن يكون لهذه المصفوفات حجم دفعة كأول بُعد، نظرًا لأن خط الأنابيب سيقوم بتقسيم الدفعة الصغيرة إلى دفعات صغيرة. تتم مناقشة التحسينات المحتملة هنا https://github.com/pytorch/pytorch/pull/50693

- لا يمكن إجراء تدفق التحكم الشرطي على مستوى مراحل الأنابيب - على سبيل المثال، تتطلب نماذج Encoder-Decoder مثل T5 حلولًا خاصة للتعامل مع مرحلة Encoder شرطية.

- يجب أن تقوم بترتيب كل طبقة بحيث يصبح إخراج إحدى الطبقات إدخالًا للطبقة الأخرى.

تشمل الحلول الأحدث ما يلي:

- Varuna
- Sagemaker

لم نجرب Varuna وSageMaker، ولكن تشير أوراقهما إلى أنهما تغلبا على قائمة المشكلات المذكورة أعلاه وأنهما يتطلبان إجراء تغييرات أقل على نموذج المستخدم.

التطبيقات:

- [PyTorch](https://pytorch.org/docs/stable/pipeline.html) (دعم أولي في pytorch-1.8، وتحسين تدريجي في 1.9 وبشكل أكبر في 1.10). بعض [الأمثلة](https://github.com/pytorch/pytorch/blob/master/benchmarks/distributed/pipeline/pipe.py)

- [DeepSpeed](https://www.deepspeed.ai/tutorials/pipeline/)

- [Megatron-LM](https://github.com/NVIDIA/Megatron-LM) لديه تطبيق داخلي - لا يوجد واجهة برمجة تطبيقات.

- [Varuna](https://github.com/microsoft/varuna)

- [SageMaker](https://arxiv.org/abs/2111.05972) - هذا حل مملوك لا يمكن استخدامه إلا على AWS.

- [OSLO](https://github.com/tunib-ai/oslo) - تم تنفيذه بناءً على محولات Hugging Face.

حالة محولات 🤗: اعتبارًا من وقت كتابة هذا التقرير، لا يدعم أي من النماذج التوازي الأنبوبي الكامل. وتدعم نماذج GPT2 وT5 النموذج الساذج للتوازي.

العقبة الرئيسية هي عدم القدرة على تحويل النماذج إلى "nn.Sequential" وجعل جميع الإدخالات Tensors. ويرجع ذلك إلى أن النماذج تحتوي حاليًا على العديد من الميزات التي تجعل التحويل معقدًا للغاية، وسيتعين إزالتها لتحقيق ذلك.

تكاملات DeepSpeed وMegatron-LM متوفرة في [🤗 Accelerate](https://huggingface.co/docs/accelerate/main/en/usage_guides/deepspeed).

الأساليب الأخرى:

تكاملات DeepSpeed وMegatron-LM متوفرة في [🤗 Accelerate](https://huggingface.co/docs/accelerate/main/en/usage_guides/deepspeed).

## التوازي الأنبوبي + التوازي في البيانات:

يوضح الرسم التخطيطي التالي من تعليمي خط أنابيب DeepSpeed كيفية الجمع بين التوازي في البيانات والتوازي الأنبوبي.

من المهم هنا ملاحظة كيف أن ترتيب DP 0 لا يرى GPU2 وترتيب DP 1 لا يرى GPU3. بالنسبة إلى DP، هناك فقط وحدات GPU 0 و1 حيث يتم إدخال البيانات كما لو كان هناك وحدتي GPU فقط. وتقوم وحدة GPU0 بتفريغ بعض عبئها سرًا على وحدة GPU2 باستخدام التوازي الأنبوبي. وتقوم وحدة GPU1 بنفس الشيء عن طريق الاستعانة بوحدة GPU3.

نظرًا لأن كل بُعد يتطلب وحدتي GPU على الأقل، فأنت بحاجة إلى 4 وحدات GPU على الأقل هنا.

التطبيقات:

- [DeepSpeed](https://github.com/microsoft/DeepSpeed)

- [Megatron-LM](https://github.com/NVIDIA/Megatron-LM)

- [Varuna](https://github.com/microsoft/varuna)

- [SageMaker](https://arxiv.org/abs/2111.05972)

- [OSLO](https://github.com/tunib-ai/oslo)

حالة محولات 🤗: لم يتم التنفيذ بعد

## التوازي في البيانات + التوازي الأنبوبي + التوازي في المصفوفات:

للحصول على تدريب أكثر كفاءة، يتم استخدام التوازي ثلاثي الأبعاد حيث يتم الجمع بين التوازي الأنبوبي والتوازي في المصفوفات والتوازي في البيانات. ويمكن رؤية ذلك في الرسم التخطيطي التالي.

هذا الرسم التخطيطي مأخوذ من منشور مدونة بعنوان "التوازي ثلاثي الأبعاد: التدرج على نماذج ذات معلمات التريليون" https://www.microsoft.com/en-us/research/blog/deepspeed-extreme-scale-model-training-for-everyone/)، وهو أيضًا قراءة جيدة.

نظرًا لأن كل بُعد يتطلب وحدتي GPU على الأقل، فأنت بحاجة إلى 8 وحدات GPU على الأقل هنا.

التطبيقات:

- [DeepSpeed](https://github.com/microsoft/DeepSpeed) - يتضمن DeepSpeed أيضًا توازيًا أكثر كفاءة في البيانات، والذي يطلقون عليه اسم ZeRO-DP.

- [Megatron-LM](https://github.com/NVIDIA/Megatron-LM)

- [Varuna](https://github.com/microsoft/varuna)

- [SageMaker](https://arxiv.org/abs/2111.05972)

- [OSLO](https://github.com/tunib-ai/oslo)

حالة محولات 🤗: لم يتم التنفيذ بعد، حيث لا يوجد توازي أنبوبي أو توازي في المصفوفات.

## التوازي في البيانات باستخدام ZeRO + التوازي الأنبوبي + التوازي في المصفوفات:

تتمثل إحدى الميزات الرئيسية لـ DeepSpeed في ZeRO، وهو امتداد قابل للتطوير للغاية للتوازي في البيانات. وقد تمت مناقشته بالفعل في قسم [التوازي في البيانات باستخدام ZeRO](#zero-data-parallelism). وعادة ما يكون ميزة مستقلة لا تتطلب التوازي الأنبوبي أو التوازي في المصفوفات. ولكن يمكن الجمع بينه وبين التوازي الأنبوبي والتوازي في المصفوفات.

عندما يتم الجمع بين ZeRO-DP والتوازي الأنبوبي (وبشكل اختياري التوازي في المصفوفات)، فإنه يمكّن عادةً مرحلة ZeRO 1 فقط (تجزئة مُحسن الخسارة).

على الرغم من أنه من الممكن نظريًا استخدام مرحلة ZeRO 2 (تجزئة التدرج) مع التوازي الأنبوبي، إلا أنه سيكون له تأثيرات سلبية على الأداء. سيتعين إجراء عملية تجميع نثرية إضافية لكل دفعة صغيرة لتجميع التدرجات قبل التجزئة، مما يضيف تكلفة اتصال محتملة كبيرة. وبسبب التوازي الأنبوبي، يتم استخدام دفعات صغيرة ويتم بدلاً من ذلك التركيز على محاولة موازنة كثافة الحساب (حجم الدفعة الصغيرة) مع تقليل "فقاعة" خط الأنابيب (عدد الدفعات الصغيرة). وبالتالي، فإن تكاليف الاتصال هذه ستؤثر على الأداء.

بالإضافة إلى ذلك، هناك بالفعل عدد أقل من الطبقات بسبب التوازي الأنبوبي، لذلك لن تكون وفورات الذاكرة كبيرة. ويقلل التوازي الأنبوبي بالفعل من حجم التدرج عن طريق "1/PP"، لذلك فإن وفورات التجزئة في التدرج بالإضافة إلى ذلك أقل أهمية من التوازي في البيانات النقي.

المرحلة 3 من ZeRO ليست خيارًا جيدًا لنفس السبب - الحاجة إلى مزيد من الاتصالات بين العقد.

وبما أننا لدينا ZeRO، فإن الفائدة الأخرى هي ZeRO-Offload. وبما أن هذه هي مرحلة 1، فيمكن تفريغ حالات مُحسن الخسارة إلى وحدة المعالجة المركزية.

التطبيقات:

- [Megatron-DeepSpeed](https://github.com/microsoft/Megatron-DeepSpeed) و [Megatron-Deepspeed من BigScience](https://github.com/bigscience-workshop/Megatron-DeepSpeed)، وهو فرع من المستودع السابق.

- [OSLO](https://github.com/tunib-ai/oslo)

الأوراق البحثية المهمة:

- [Using DeepSpeed and Megatron to Train Megatron-Turing NLG 530B, A Large-Scale Generative Language Model](https://arxiv.org/abs/2201.11990)

حالة محولات 🤗: لم يتم التنفيذ بعد، حيث لا يوجد توازي أنبوبي أو توازي في المصفوفات.

## FlexFlow:

يحل FlexFlow أيضًا مشكلة التوازي بطريقة مختلفة قليلاً.

الورقة البحثية: ["Beyond Data and Model Parallelism for Deep Neural Networks" by Zhihao Jia, Matei Zaharia, Alex Aiken](https://arxiv.org/abs/1807.05358)

ويؤدي نوعًا ما من التوازي رباعي الأبعاد على Sample-Operator-Attribute-Parameter.

1. العينة = التوازي في البيانات (توازي على مستوى العينة)

2. المشغل = توازي عملية واحدة في عدة عمليات فرعية

3. السمة = التوازي في البيانات (توازي على مستوى الطول)

4. المعلمة = التوازي في النموذج (بغض النظر عن البُعد - أفقي أو رأسي)

أمثلة:

* العينة

لنأخذ 10 دفعات من طول التسلسل 512. إذا قمنا بتوازيها حسب بُعد العينة إلى جهازي كمبيوتر، فسنحصل على 10 × 512 والتي تصبح 5 × 2 × 512.

* المشغل

إذا قمنا بتطبيع الطبقة، فإننا نحسب الانحراف المعياري أولاً ثم المتوسط، وبعد ذلك يمكننا تطبيع البيانات. يسمح توازي المشغل بحساب الانحراف المعياري والمتوسط في نفس الوقت. لذلك، إذا قمنا بتوازيها حسب بُعد المشغل إلى جهازي كمبيوتر (cuda:0، cuda:1)، فنحن نقوم بنسخ بيانات الإدخال
يعد الوعد جذابًا للغاية - فهو يعمل على محاكاة لمدة 30 دقيقة على مجموعة الخيارات الخاصة بك ويخرج بأفضل استراتيجية لاستغلال هذه البيئة المحددة. إذا قمت بإضافة/إزالة/استبدال أي أجزاء، فسيقوم بالتشغيل وإعادة تحسين الخطة لذلك. وبعد ذلك يمكنك التدريب. سيكون لكل إعداد مختلف تحسينه الخاص.

🤗 حالة المحولات: يمكن تتبع نماذج المحولات عبر [transformers.utils.fx](https://github.com/huggingface/transformers/blob/master/src/transformers/utils/fx.py)، وهو شرط أساسي لـ FlexFlow، ومع ذلك، هناك حاجة إلى تغييرات على جانب FlexFlow لجعله يعمل مع نماذج المحولات.

## اختيار وحدة معالجة الرسومات (GPU)

عند التدريب على وحدات معالجة رسومات متعددة، يمكنك تحديد عدد وحدات معالجة الرسومات التي تريد استخدامها والترتيب الذي تريد استخدامها به. يمكن أن يكون هذا مفيدًا، على سبيل المثال، عندما يكون لديك وحدات معالجة رسومات ذات قدرة حوسبة مختلفة وتريد استخدام وحدة معالجة الرسومات الأسرع أولاً. تنطبق عملية الاختيار على كل من [DistributedDataParallel](https://pytorch.org/docs/stable/generated/torch.nn.parallel.DistributedDataParallel.html) و [DataParallel](https://pytorch.org/docs/stable/generated/torch.nn.DataParallel.html) لاستخدام مجموعة فرعية فقط من وحدات معالجة الرسومات المتوفرة، ولا تحتاج إلى Accelerate أو [تكامل DeepSpeed](./main_classes/deepspeed).

### عدد وحدات معالجة الرسومات

على سبيل المثال، إذا كان لديك 4 وحدات معالجة رسومات وتريد استخدام أول اثنتين فقط:

<hfoptions id="select-gpu">
<hfoption id="torchrun">

استخدم `--nproc_per_node` لاختيار عدد وحدات معالجة الرسومات التي تريد استخدامها.

```bash
torchrun --nproc_per_node=2  trainer-program.py ...
```

</hfoption>
<hfoption id="Accelerate">

استخدم `--num_processes` لاختيار عدد وحدات معالجة الرسومات التي تريد استخدامها.

```bash
accelerate launch --num_processes 2 trainer-program.py ...
```

</hfoption>
<hfoption id="DeepSpeed">

استخدم `--num_gpus` لاختيار عدد وحدات معالجة الرسومات التي تريد استخدامها.

```bash
deepspeed --num_gpus 2 trainer-program.py ...
```

</hfoption>
</hfoptions>

### ترتيب وحدات معالجة الرسومات

الآن، لاختيار وحدات معالجة الرسومات التي تريد استخدامها وترتيبها، استخدم متغير البيئة `CUDA_VISIBLE_DEVICES`. من الأسهل تعيين متغير البيئة في `~/bashrc` أو ملف تهيئة آخر. يستخدم `CUDA_VISIBLE_DEVICES` لتعيين خريطة وحدات معالجة الرسومات المستخدمة. على سبيل المثال، إذا كان لديك 4 وحدات معالجة رسومات (0، 1، 2، 3) وتريد تشغيل وحدتي معالجة الرسومات 0 و 2 فقط:

```bash
CUDA_VISIBLE_DEVICES=0,2 torchrun trainer-program.py ...
```

تكون وحدتا معالجة الرسومات الفعليتان (0 و 2) "مرئيتين" فقط لـ PyTorch، ويتم تعيينهما إلى `cuda:0` و `cuda:1` على التوالي. يمكنك أيضًا عكس ترتيب وحدات معالجة الرسومات لاستخدام 2 أولاً. الآن، يتم تعيين الخريطة إلى `cuda:1` لوحدة معالجة الرسومات 0 و `cuda:0` لوحدة معالجة الرسومات 2.

```bash
CUDA_VISIBLE_DEVICES=2,0 torchrun trainer-program.py ...
```

يمكنك أيضًا تعيين متغير البيئة `CUDA_VISIBLE_DEVICES` إلى قيمة فارغة لإنشاء بيئة بدون وحدات معالجة رسومات.

```bash
CUDA_VISIBLE_DEVICES= python trainer-program.py ...
```

<Tip warning={true}>

كما هو الحال مع أي متغير بيئي، يمكن تصديرها بدلاً من إضافتها إلى سطر الأوامر. ومع ذلك، لا يوصى بذلك لأنه يمكن أن يكون مربكًا إذا نسيت كيفية إعداد متغير البيئة وتستخدم وحدات معالجة الرسومات الخطأ. بدلاً من ذلك، من الشائع تعيين متغير البيئة لتشغيل تدريب محدد على نفس سطر الأوامر.

</Tip>

`CUDA_DEVICE_ORDER` هو متغير بيئي بديل يمكنك استخدامه للتحكم في كيفية ترتيب وحدات معالجة الرسومات. يمكنك ترتيبها إما عن طريق:

1. معرفات حافلة PCIe التي تتطابق مع ترتيب ["nvidia-smi"](https://developer.nvidia.com/nvidia-system-management-interface) و ["rocm-smi"](https://rocm.docs.amd.com/projects/rocm_smi_lib/en/latest/.doxygen/docBin/html/index.html) لوحدات معالجة الرسومات NVIDIA و AMD على التوالي

```bash
export CUDA_DEVICE_ORDER=PCI_BUS_ID
```

2. قدرة الحوسبة لوحدة معالجة الرسومات

```bash
export CUDA_DEVICE_ORDER=FASTEST_FIRST
```
```bash
export CUDA_DEVICE_ORDER=PCI_BUS_ID
```

2. قدرة الحوسبة لوحدة معالجة الرسومات

```bash
export CUDA_DEVICE_ORDER=FASTEST_FIRST
```

`CUDA_DEVICE_ORDER` مفيد بشكل خاص إذا كانت مجموعة التدريب الخاصة بك تتكون من وحدة معالجة رسومات أقدم وأخرى أحدث، حيث تظهر وحدة معالجة الرسومات الأقدم أولاً، ولكن لا يمكنك استبدال البطاقات فعليًا لجعل وحدة معالجة الرسومات الأحدث تظهر أولاً. في هذه الحالة، قم بتعيين `CUDA_DEVICE_ORDER=FASTEST_FIRST` لاستخدام وحدة معالجة الرسومات الأحدث والأسرع أولاً (`nvidia-smi` أو `rocm-smi` لا يزال يبلغ عن وحدات معالجة الرسومات الخاصة بهم في ترتيب PCIe). أو يمكنك أيضًا تعيين `export CUDA_VISIBLE_DEVICES=1,0`.