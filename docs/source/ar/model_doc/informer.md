# Informer

## نظرة عامة

اقترح نموذج Informer في "Informer: Beyond Efficient Transformer for Long Sequence Time-Series Forecasting" من قبل هايوي تشو، وشانجهانغ جانغ، وجييكي بينغ، وشوواي جانغ، وجيانكسين لي، وهوي شيونغ، ووانكاي جانغ.

تقدم هذه الطريقة آلية اهتمام احتمالية لاختيار الاستعلامات "النشطة" بدلاً من الاستعلامات "الخاملة" وتوفر محولًا مبعثرًا، مما يخفف من متطلبات الحوسبة والذاكرة التربيعية لاهتمام الفانيلا.

الملخص من الورقة هو ما يلي:

تتطلب العديد من تطبيقات العالم الحقيقي التنبؤ بسلاسل زمنية طويلة، مثل تخطيط استهلاك الكهرباء. يتطلب التنبؤ بسلاسل زمنية طويلة (LSTF) قدرة تنبؤية عالية للنموذج، وهي القدرة على التقاط الاعتماد طويل المدى الدقيق بين الإخراج والإدخال بكفاءة. أظهرت الدراسات الحديثة الإمكانات التي يتمتع بها المحول لزيادة قدرة التنبؤ. ومع ذلك، هناك عدة مشكلات خطيرة مع المحول تمنعه من أن يكون قابلاً للتطبيق مباشرة على LSTF، بما في ذلك التعقيد الزمني التربيعي، والاستخدام العالي للذاكرة، والقيود المتأصلة في بنية الترميز فك الترميز. لمعالجة هذه القضايا، نقوم بتصميم نموذج محول كفء لـ LSTF، يسمى Informer، مع ثلاث خصائص مميزة: (1) آلية ProbSparse self-attention، والتي تحقق O (L logL) في التعقيد الزمني واستخدام الذاكرة، ولها أداء قابل للمقارنة في محاذاة تبعية التسلسلات. (2) تقطير الاهتمام الذاتي يسلط الضوء على الاهتمام المهيمن عن طريق تقسيم إدخال الطبقة المتتالية إلى النصف، ويتعامل بكفاءة مع تسلسلات الإدخال الطويلة للغاية. (3) فك تشفير النمط التوليدي، على الرغم من بساطته من الناحية المفاهيمية، فإنه يتنبأ بتسلسلات السلاسل الزمنية الطويلة في عملية توجيه واحدة بدلاً من طريقة الخطوة بخطوة، مما يحسن بشكل كبير من سرعة الاستدلال في التنبؤات ذات التسلسلات الطويلة. تُظهر التجارب واسعة النطاق على أربع مجموعات بيانات كبيرة الحجم أن Informer يتفوق بشكل كبير على الطرق الموجودة ويقدم حلاً جديدًا لمشكلة LSTF.

ساهم بهذا النموذج [elisim](https://huggingface.co/elisim) و [kashif](https://huggingface.co/kashif).

يمكن العثور على الكود الأصلي [هنا](https://github.com/zhouhaoyi/Informer2020).

## الموارد

فيما يلي قائمة بموارد Hugging Face الرسمية وموارد المجتمع (المشار إليها بـ 🌎) لمساعدتك في البدء. إذا كنت مهتمًا بتقديم مورد لإدراجه هنا، فالرجاء فتح طلب سحب وسنراجعه! يجب أن يوضح المورد المثالي شيئًا جديدًا بدلاً من تكرار مورد موجود.

- اطلع على منشور مدونة Informer في مدونة HuggingFace: [Multivariate Probabilistic Time Series Forecasting with Informer](https://huggingface.co/blog/informer)

## InformerConfig

[[autodoc]] InformerConfig

## نموذج Informer

[[autodoc]] InformerModel

- forword

## InformerForPrediction

[[autodoc]] InformerForPrediction

- forword