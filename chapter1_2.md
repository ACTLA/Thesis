# 1.2. Угрозы и уязвимости биометрических систем распознавания лиц

Биометрические системы распознавания лиц, несмотря на высокую точность в контролируемых условиях, подвержены специфическим угрозам безопасности, принципиально отличающимся от традиционных атак на информационные системы. Понимание этих угроз критически важно для разработки эффективных мер защиты и оценки рисков при внедрении биометрических технологий.

## Классификация презентационных атак

Презентационные атаки представляют собой наиболее распространенный тип угроз для систем распознавания лиц. Международный стандарт ISO/IEC 30107 определяет презентационную атаку как предъявление поддельного биометрического образца биометрической системе захвата с целью нарушения её работы.

Фотографические атаки используют печатные изображения лица для обмана системы распознавания. Исследования показывают, что простые печатные фотографии высокого разрешения успешно обманывают до 75% коммерческих систем, не оснащенных средствами обнаружения подделок. Усложненные варианты таких атак включают вырезание областей глаз на фотографии для имитации моргания или использование изогнутых поверхностей для создания иллюзии трехмерности.

Видео-атаки предполагают воспроизведение записанных видеофайлов с изображением целевого пользователя на экранах мониторов, планшетов или смартфонов. Такие атаки более сложны в исполнении, но демонстрируют высокую эффективность против систем, требующих обнаружения движения или изменения выражения лица.

Трехмерные атаки используют маски, макеты или силиконовые копии лица для создания физически правдоподобных подделок. Современные технологии 3D-печати и фотограмметрии позволяют создавать высококачественные трехмерные модели лиц на основе фотографий из социальных сетей или публичных источников.

В контексте разработанной системы все перечисленные типы атак представляют критическую угрозу, поскольку алгоритм обработки кадров в `face_recognition_engine.py` не включает проверки подлинности биометрических образцов:

```python
def _detect_faces(self, frame):
    # Система уязвима ко всем типам презентационных атак
    face_locations = face_recognition.face_locations(rgb_frame, model='hog')
    face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)
    # Отсутствует анализ признаков живого лица
```

## Атаки на биометрические шаблоны

Компрометация биометрических шаблонов представляет особую категорию угроз из-за невозможности замены скомпрометированных биометрических характеристик. Атаки на базы данных биометрических шаблонов могут иметь долгосрочные последствия для безопасности пользователей.

Прямые атаки на хранилища данных направлены на получение несанкционированного доступа к базам данных с биометрическими шаблонами. Успешная атака такого типа предоставляет злоумышленнику доступ ко всем биометрическим данным пользователей системы. В разработанной системе биометрические шаблоны хранятся в незашифрованном JSON формате, что делает их полностью доступными при компрометации файла базы данных:

```python
# Критическая уязвимость - открытое хранение биометрических данных
cursor.execute('''
    INSERT INTO users (user_id, full_name, photo_path, face_encoding, created_by)
    VALUES (?, ?, ?, ?, ?)
''', (
    user_data['user_id'],
    user_data['full_name'],
    user_data.get('photo_path', ''),
    json.dumps(user_data.get('face_encoding', [])),  # Незащищенные данные
    created_by
))
```

Атаки через каналы утечки информации эксплуатируют побочные эффекты работы биометрических систем. Анализ времени выполнения операций сравнения, энергопотребления или электромагнитного излучения может предоставить информацию о биометрических шаблонах без прямого доступа к данным.

Атаки типа "хилл-климбинг" используют итеративное улучшение поддельных биометрических образцов на основе обратной связи от системы. Злоумышленник постепенно модифицирует подделку, анализируя реакцию системы, пока не достигнет успешной идентификации.

## Архитектурные уязвимости

Централизованная архитектура многих биометрических систем создает единые точки отказа, компрометация которых может привести к полной потере безопасности. В разработанной системе паттерн Синглтон для менеджера камеры создает критическую точку уязвимости:

```python
class CameraManager(QObject):
    _instance = None
    _lock = threading.Lock()
    
    # Единственный экземпляр - потенциальная цель атак
    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance
```

Недостаточное разделение привилегий между компонентами системы позволяет эскалацию прав при компрометации одного из модулей. Отсутствие строгих границ безопасности между компонентами увеличивает потенциальный ущерб от успешных атак.

Слабые механизмы аутентификации и авторизации создают возможности для несанкционированного доступа к административным функциям. В анализируемой системе все аутентифицированные администраторы получают полный доступ без дифференциации прав.

## Угрозы конфиденциальности данных

Биометрические данные содержат чувствительную информацию о физических характеристиках человека, что создает специфические угрозы конфиденциальности. Современные исследования показывают возможность извлечения дополнительной информации из биометрических шаблонов, включая этническую принадлежность, состояние здоровья и даже генетические характеристики.

Межсистемная корреляция биометрических данных позволяет связывать активность пользователя в различных биометрических системах. Использование одинаковых биометрических характеристик в множественных системах создает возможности для профилирования и отслеживания поведения пользователей.

Атаки на конфиденциальность через машинное обучение используют алгоритмы анализа данных для извлечения скрытой информации из биометрических шаблонов. Membership inference attacks позволяют определить, использовались ли данные конкретного человека для обучения модели распознавания.

## Уязвимости алгоритмического уровня

Adversarial examples представляют класс специально созданных входных данных, вызывающих некорректную работу алгоритмов машинного обучения. Для систем распознавания лиц такие атаки могут включать незаметные изменения изображений, приводящие к ошибочной идентификации или отказу в распознавании.

Model inversion attacks направлены на восстановление обучающих данных на основе анализа поведения обученной модели. Успешная атака такого типа может позволить воссоздать изображения лиц из биометрических шаблонов.

Backdoor attacks предполагают внедрение скрытой функциональности в модели машинного обучения на этапе обучения. Такие атаки могут быть активированы специальными триггерами и приводить к контролируемому злоумышленником поведению системы.

## Угрозы доступности системы

Denial of Service атаки на биометрические системы могут быть направлены на перегрузку вычислительных ресурсов или создание ситуаций, при которых система не может корректно функционировать. Подача большого количества сложных для обработки изображений может привести к исчерпанию ресурсов сервера.

Физические атаки на инфраструктуру включают повреждение камер, освещения или других компонентов системы захвата биометрических данных. Такие атаки могут быть направлены на создание условий, при которых биометрическая система не может функционировать.

Environmental attacks эксплуатируют изменения в условиях работы системы. Изменение освещения, создание теней или использование инфракрасных источников может нарушить работу алгоритмов обнаружения и распознавания лиц.

## Социальные и этические аспекты уязвимостей

Биометрические системы создают уникальные социальные риски, связанные с возможностью массового наблюдения и нарушения приватности граждан. Неконтролируемое развертывание систем распознавания лиц может привести к созданию инфраструктуры тотального контроля.

Алгоритмическая предвзятость в моделях машинного обучения приводит к неравномерной точности распознавания для различных демографических групп. Исследования показывают значительные различия в частоте ошибок для представителей разных рас, возрастных групп и полов, что создает проблемы справедливости использования технологии.

Проблема согласия и информированности пользователей становится критической при широком внедрении биометрических технологий. Многие пользователи не осознают рисков, связанных с предоставлением своих биометрических данных, и долгосрочных последствий их возможной компрометации.

## Анализ уязвимостей реализованной системы

Разработанная система демонстрирует типичные уязвимости биометрических приложений, не учитывающих требования безопасности на этапе проектирования. Основные проблемы включают:

Полное отсутствие защиты от презентационных атак в алгоритме обработки видеопотока. Система принимает любые изображения с обнаруженными лицами без проверки их подлинности.

Незащищенное хранение биометрических шаблонов в базе данных SQLite в формате JSON без шифрования или других мер защиты конфиденциальности.

Недостаточное журналирование событий безопасности, что затрудняет обнаружение атак и расследование инцидентов.

```python
# Пример уязвимого кода - отсутствие проверок безопасности
def on_face_recognized(self, match):
    # Нет проверки на презентационные атаки
    user = self.db.get_user_by_id(match.user_id)
    # Нет анализа подозрительных паттернов
    self.db.add_recognition_log(match.user_id, match.confidence, 'SUCCESS')
```

Архитектурные решения, создающие единые точки отказа и не обеспечивающие изоляцию критических компонентов системы.

## Методология оценки уязвимостей

Систематическая оценка уязвимостей биометрических систем требует применения специализированных методологий, учитывающих специфику биометрических данных. Модель STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) может быть адаптирована для анализа биометрических систем.

Презентационные атаки соответствуют категории Spoofing, атаки на биометрические шаблоны - категории Tampering и Information Disclosure. Отказ в обслуживании и эскалация привилегий также представляют значимые угрозы для биометрических систем.

Количественная оценка рисков должна учитывать как вероятность успешной реализации атаки, так и потенциальный ущерб от её последствий. Для биометрических данных ущерб от компрометации может быть особенно значительным из-за невозможности их замены.

Тестирование на проникновение биометрических систем требует специализированных инструментов и методик. Это включает создание наборов поддельных биометрических образцов, анализ устойчивости к adversarial examples и оценку эффективности мер защиты данных.

## Эволюция угроз и будущие вызовы

Развитие технологий искусственного интеллекта создает новые типы угроз для биометрических систем. Генеративные adversarial networks позволяют создавать высококачественные синтетические изображения лиц, которые могут быть использованы для атак на системы распознавания.

Deepfake технологии обеспечивают создание реалистичных видео с поддельными лицами, что представляет новый уровень угроз для видео-биометрических систем. Развитие этих технологий опережает создание эффективных методов их обнаружения.

Квантовые вычисления в перспективе могут нарушить криптографические основы защиты биометрических данных. Алгоритм Шора позволит эффективно взламывать современные асимметричные криптосистемы, что потребует перехода на квантово-устойчивые методы защиты.

Интеграция биометрических систем с технологиями Интернета вещей и облачными сервисами расширяет поверхность атак и создает новые векторы угроз. Распределенная обработка биометрических данных требует обеспечения безопасности на всех этапах их жизненного цикла.

Понимание современных угроз и уязвимостей биометрических систем является необходимой основой для разработки эффективных мер защиты. Анализ реализованной системы демонстрирует типичные проблемы безопасности, которые должны быть устранены при создании промышленных биометрических решений.