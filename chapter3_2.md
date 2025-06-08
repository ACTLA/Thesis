# 3.2. Модуль управления пользователями

Модуль управления пользователями объединяет функции регистрации, изменения и удаления пользователей системы распознавания лиц. Реализация включает интуитивный пользовательский интерфейс для захвата биометрических данных и надежные механизмы проверки качества биометрических образцов.

## Процесс добавления пользователей

Диалог добавления пользователей в файле `ui/add_user_dialog.py` построен на адаптивной архитектуре с разделением на секции ввода персональных данных и захвата биометрических характеристик. Интерфейс автоматически адаптируется под различные размеры экранов используя компонент `QSplitter`.

Форма персональных данных включает обязательные поля идентификатора пользователя и полного имени с валидацией в реальном времени. Система блокирует сохранение при незаполненных критических полях и предотвращает создание пользователей с дублирующимися идентификаторами.

```python
def add_user(self):
    if not self.user_id_input.text().strip():
        QMessageBox.warning(self, "Ошибка", "Введите идентификатор пользователя")
        return
    
    if not self.full_name_input.text().strip():
        QMessageBox.warning(self, "Ошибка", "Введите полное имя пользователя")
        return
    
    if not self.face_encoding:
        QMessageBox.warning(self, "Ошибка", "Добавьте фотографию пользователя")
        return
```

Система захвата биометрических данных реализована через вкладочный интерфейс с двумя режимами: загрузка изображения из файла и съемка с камеры. Каждый режим включает специализированную логику обработки и проверки качества биометрических образцов.

## Интеграция с камерой для регистрации

Режим съемки с камеры использует отдельный поток `CameraThread` для предотвращения блокировки пользовательского интерфейса. Поток обеспечивает непрерывный захват кадров и автоматическое обнаружение лиц с визуальной индикацией результатов.

```python
class CameraThread(QThread):
    frame_ready = pyqtSignal(np.ndarray)
    face_detected = pyqtSignal(np.ndarray, list)
    
    def run(self):
        self.is_running = True
        
        try:
            self.cap = cv2.VideoCapture(0, cv2.CAP_DSHOW)
            if not self.cap.isOpened():
                self.cap = cv2.VideoCapture(0)
        except Exception:
            return
        
        while self.is_running:
            ret, frame = self.cap.read()
            if ret and self.is_running:
                self.frame_ready.emit(frame.copy())
                
                rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
                face_locations = face_recognition.face_locations(rgb_frame)
                
                if face_locations:
                    self.face_detected.emit(frame.copy(), face_locations)
```

Система индикации состояния информирует пользователя о процессе регистрации через цветовые метки: "Поиск лица", "Лицо найдено", "Несколько лиц". Кнопка захвата активируется только при обнаружении ровно одного лица в кадре, что гарантирует качество биометрических данных.

Функция захвата лица создает временный файл с уникальным именем на основе временной метки и сохраняет кадр с выделенной областью лица. Биометрический шаблон извлекается из полного разрешения изображения для обеспечения максимальной точности.

## Обработка биометрических шаблонов

Извлечение биометрических характеристик выполняется через библиотеку face_recognition, которая создает 128-элементный вектор численных признаков лица. Система проверяет успешность создания кодировки и отклоняет некачественные образцы.

```python
def process_image_file(self, file_path):
    image = cv2.imread(file_path)
    if image is None:
        QMessageBox.warning(self, "Ошибка", "Не удалось загрузить изображение")
        return
    
    rgb_image = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)
    face_locations = face_recognition.face_locations(rgb_image)
    
    if len(face_locations) == 0:
        QMessageBox.warning(self, "Ошибка", "На фотографии не обнаружено лицо")
        return
    
    if len(face_locations) > 1:
        QMessageBox.warning(self, "Ошибка", "На фотографии обнаружено несколько лиц")
        return
    
    face_encodings = face_recognition.face_encodings(rgb_image, face_locations)
    if face_encodings:
        self.face_encoding = face_encodings[0].tolist()
        self.photo_path = file_path
        self.display_image_with_face_box(image, face_locations[0])
```

Сохранение пользователя в базе данных выполняется через метод `add_user` класса `Database` с автоматической обработкой ошибок дублирования. Биометрический шаблон сериализуется в JSON формат для совместимости с SQLite.

Система автоматически обновляет кэш движка распознавания при добавлении нового пользователя через вызов `reload_face_encodings()`. Это обеспечивает немедленную доступность новых пользователей для идентификации.

## Управление существующими пользователями

Главное окно системы отображает список пользователей в таблице с колонками основных данных и элементами управления. Метод `update_users_table` загружает информацию из базы данных и форматирует её для отображения.

```python
def update_users_table(self):
    try:
        users = self.db.get_all_users()
        
        self.users_table.setRowCount(len(users))
        for i, user in enumerate(users):
            self.users_table.setItem(i, 0, QTableWidgetItem(user['user_id']))
            self.users_table.setItem(i, 1, QTableWidgetItem(user['full_name']))
            self.users_table.setItem(i, 2, QTableWidgetItem(user.get('email', '')))
            
            # Форматирование даты
            created_at = user['created_at']
            if isinstance(created_at, str):
                try:
                    dt = datetime.fromisoformat(created_at.replace('Z', '+00:00'))
                    formatted_date = dt.strftime('%d.%m.%Y %H:%M')
                except:
                    formatted_date = created_at
            else:
                formatted_date = str(created_at)
            
            self.users_table.setItem(i, 3, QTableWidgetItem(formatted_date))
            
            # Кнопка удаления
            delete_btn = QPushButton("Удалить")
            delete_btn.clicked.connect(lambda _, user_id=user['id']: self.delete_user(user_id))
            self.users_table.setCellWidget(i, 4, delete_btn)
    except Exception as e:
        pass
```

Функция удаления пользователей реализована как мягкое удаление через установку флага неактивности вместо физического удаления записи. Это сохраняет исторические данные для аудита при необходимости восстановления.

Каждая операция удаления требует подтверждения через диалоговое окно для предотвращения случайных действий. При подтверждении система обновляет кэш распознавания и перезагружает таблицу для отражения изменений.

## Контроль качества биометрических данных

Система включает многоуровневую проверку качества биометрических образцов. Проверка на уровне алгоритмов face_recognition отклоняет изображения без обнаруженных лиц или с неопределенными характеристиками.

Контроль множественных лиц предотвращает регистрацию неоднозначных биометрических образцов. Система требует изоляции одного пользователя в кадре и отображает соответствующие предупреждения при обнаружении нескольких лиц.

Автоматическая очистка временных файлов выполняется при закрытии диалога или отмене операции. Переопределенные методы `closeEvent` и `reject` обеспечивают удаление временных файлов, созданных при съемке с камеры.

```python
def closeEvent(self, event):
    self.stop_camera()
    
    if self.photo_path and 'temp_capture_' in self.photo_path:
        try:
            os.remove(self.photo_path)
        except:
            pass
    
    event.accept()
```

Такая реализация модуля управления пользователями обеспечивает надежную регистрацию биометрических данных с контролем качества и создает основу для интеграции дополнительных проверок безопасности в процесс добавления пользователей.