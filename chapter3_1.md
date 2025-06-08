# 3.1. Архитектура и структура программного комплекса

Разработанная автоматизированная система распознавания лиц построена на модульной архитектуре, обеспечивающей гибкость, масштабируемость и безопасность. Система состоит из нескольких взаимосвязанных компонентов, каждый из которых выполняет специфические функции и может быть модифицирован независимо от других модулей.

## Общая архитектура системы

Архитектура системы основана на принципе разделения ответственности, где каждый модуль имеет четко определенные функции и интерфейсы взаимодействия. Это обеспечивает простоту сопровождения, тестирования и развития системы.

```python
# main.py - Главный файл приложения
class FaceRecognitionApplication:
    def __init__(self):
        self.config = self._load_configuration()
        self.security_manager = SecurityManager(self.config.security)
        self.database = Database()
        self.camera_manager = CameraManager()
        self.recognition_engine = FaceRecognitionEngine(self.database)
        self.audit_system = SecurityAuditSystem(self.config.audit)
        
        # Инициализация модулей безопасности
        self.liveness_detector = LivenessDetector(self.config.liveness)
        self.crypto_manager = BiometricCryptographicSystem(
            self.config.crypto.master_key, 
            self.config.crypto.salt
        )
        
        self.logger = logging.getLogger(__name__)
        
    def initialize_system(self):
        """Инициализация всех компонентов системы"""
        try:
            # Проверка целостности системы
            if not self.security_manager.verify_system_integrity():
                raise SecurityError("Нарушена целостность системы")
            
            # Инициализация базы данных
            self.database.initialize()
            
            # Загрузка биометрических шаблонов
            self.recognition_engine.load_known_faces()
            
            # Проверка доступности камеры
            if not self.camera_manager.is_available():
                self.logger.warning("Камера недоступна")
            
            self.logger.info("Система успешно инициализирована")
            
        except Exception as e:
            self.logger.error(f"Ошибка инициализации системы: {e}")
            raise
```

Основными компонентами системы являются:

**Модуль управления пользовательским интерфейсом** обеспечивает взаимодействие с пользователем через графическую оболочку, построенную на библиотеке PyQt5. Он включает окна входа в систему, главное окно управления, диалоги добавления пользователей и виджет распознавания лиц в реальном времени.

**Модуль управления камерой** реализован как синглтон для предотвращения конфликтов доступа к аппаратным ресурсам. Он обеспечивает централизованное управление видеопотоком, подписку различных компонентов на получение кадров и надежное освобождение ресурсов.

**Движок распознавания лиц** выполняет основную функциональность идентификации пользователей. Он включает кэширование биометрических шаблонов, оптимизацию производительности через пропуск кадров и ограничение частоты распознавания одного пользователя.

## Структура модулей безопасности

Модули безопасности интегрированы в основную архитектуру системы и взаимодействуют с другими компонентами через четко определенные интерфейсы.

```python
# security/__init__.py
class SecurityManager:
    """Центральный менеджер безопасности"""
    def __init__(self, config):
        self.config = config
        self.liveness_detector = LivenessDetector(config.liveness)
        self.crypto_system = BiometricCryptographicSystem(
            config.master_key, config.salt
        )
        self.audit_logger = SecurityAuditLogger(config.audit)
        self.access_control = AccessControlManager(config.access)
        
    def authenticate_user(self, face_data, user_credentials=None):
        """Комплексная аутентификация пользователя"""
        auth_result = {
            'success': False,
            'user_id': None,
            'confidence': 0.0,
            'security_checks': {}
        }
        
        try:
            # Проверка подлинности лица
            liveness_result = self.liveness_detector.analyze_liveness(face_data)
            auth_result['security_checks']['liveness'] = liveness_result
            
            if not liveness_result['overall_score'] > 0.7:
                self.audit_logger.log_security_event(
                    'LIVENESS_CHECK_FAILED', 
                    details=liveness_result
                )
                return auth_result
            
            # Распознавание лица
            recognition_result = self._perform_face_recognition(face_data)
            auth_result.update(recognition_result)
            
            # Дополнительные проверки безопасности
            if auth_result['success']:
                access_granted = self.access_control.check_access_permissions(
                    auth_result['user_id']
                )
                auth_result['access_granted'] = access_granted
            
            # Регистрация события
            self.audit_logger.log_authentication_attempt(
                auth_result['user_id'], 
                auth_result['success'], 
                auth_result
            )
            
            return auth_result
            
        except Exception as e:
            self.audit_logger.log_security_event(
                'AUTHENTICATION_ERROR', 
                error=str(e)
            )
            raise
```

**Модуль обнаружения живого лица** интегрирован в процесс аутентификации и выполняет проверку подлинности биометрического образца перед передачей его в систему распознавания. Он анализирует последовательность кадров для выявления признаков живого лица.

**Криптографический модуль** обеспечивает защиту биометрических данных на всех этапах их обработки. Он автоматически шифрует новые биометрические шаблоны при добавлении пользователей и расшифровывает их при необходимости сравнения.

**Модуль аудита** работает в фоновом режиме и регистрирует все значимые события в системе. Он интегрирован во все критические операции и обеспечивает создание защищенных журналов событий.

## Структура базы данных и хранения

База данных системы спроектирована с учетом требований безопасности и производительности. Она включает таблицы для хранения информации о пользователях, администраторах, журналов распознавания и событий безопасности.

```python
# database.py
class Database:
    def __init__(self):
        self.db_path = str(DATABASE_PATH)
        self._lock = threading.RLock()
        self.crypto_manager = BiometricCryptographicSystem(
            get_master_key(), get_salt()
        )
        
    def add_user_secure(self, user_data, admin_id):
        """Безопасное добавление пользователя с шифрованием"""
        try:
            with self.get_connection() as conn:
                cursor = conn.cursor()
                
                # Шифрование биометрического шаблона
                if user_data.get('face_encoding'):
                    encrypted_encoding = self.crypto_manager.encrypt_template(
                        np.array(user_data['face_encoding']),
                        user_data['user_id']
                    )
                    
                    # Создание хэша целостности
                    integrity_hash = self.crypto_manager.create_integrity_hash(
                        user_data['face_encoding']
                    )
                    
                    user_data['face_encoding'] = json.dumps(encrypted_encoding)
                    user_data['integrity_hash'] = integrity_hash
                
                # Вставка данных пользователя
                cursor.execute('''
                    INSERT INTO users 
                    (user_id, full_name, email, phone, photo_path, 
                     face_encoding, integrity_hash, created_by)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                ''', (
                    user_data['user_id'],
                    user_data['full_name'],
                    user_data.get('email', ''),
                    user_data.get('phone', ''),
                    user_data.get('photo_path', ''),
                    user_data['face_encoding'],
                    user_data.get('integrity_hash', ''),
                    admin_id
                ))
                
                user_id = cursor.lastrowid
                conn.commit()
                
                # Регистрация события добавления пользователя
                audit_logger.log_admin_action(
                    admin_id, 
                    'USER_ADDED', 
                    {'new_user_id': user_id, 'user_code': user_data['user_id']}
                )
                
                return user_id
                
        except Exception as e:
            logger.error(f"Ошибка добавления пользователя: {e}")
            raise
```

Таблица пользователей хранит зашифрованные биометрические шаблоны вместе с хэшами целостности для обнаружения несанкционированных изменений. Метаданные пользователей хранятся отдельно для оптимизации запросов, не требующих доступа к биометрическим данным.

Таблица журналов распознавания содержит детальную информацию о каждой попытке аутентификации, включая уровень уверенности, результаты проверки подлинности и контекстные данные.

## Обработка ошибок и восстановление

Система включает комплексные механизмы обработки ошибок и восстановления, обеспечивающие стабильную работу даже в условиях сбоев отдельных компонентов.

```python
class SystemRecoveryManager:
    def __init__(self, system_components):
        self.components = system_components
        self.health_monitor = HealthMonitor()
        
    def handle_component_failure(self, component_name, error):
        """Обработка сбоя компонента системы"""
        logger.error(f"Сбой компонента {component_name}: {error}")
        
        # Попытка восстановления компонента
        if self._attempt_component_recovery(component_name):
            logger.info(f"Компонент {component_name} восстановлен")
            return True
        
        # Переключение на резервный режим
        if self._enable_fallback_mode(component_name):
            logger.warning(f"Активирован резервный режим для {component_name}")
            return True
        
        # Критическая ошибка
        self._handle_critical_failure(component_name, error)
        return False
```

Модульная архитектура системы обеспечивает высокую надежность, безопасность и простоту сопровождения. Каждый компонент может быть модифицирован или заменен независимо, что позволяет легко адаптировать систему под изменяющиеся требования безопасности.