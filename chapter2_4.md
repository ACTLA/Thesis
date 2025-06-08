# 2.4. Система аудита и мониторинга безопасности

Комплексная система аудита и мониторинга является критически важным компонентом защищенной биометрической системы. Она обеспечивает непрерывный контроль за всеми операциями в системе, своевременное обнаружение подозрительной активности и возможность расследования инцидентов безопасности.

## Архитектура системы аудита

Система аудита проектируется как распределенная архитектура, обеспечивающая надежность и защищенность журналов событий. Основными компонентами являются агенты сбора событий, центральный сервер обработки журналов и защищенное хранилище аудиторских данных.

```python
class SecurityAuditSystem:
    def __init__(self, config):
        self.config = config
        self.encryption_manager = AuditEncryptionManager(config.audit_key)
        self.event_processor = EventProcessor(config.processing_rules)
        self.storage_manager = SecureStorageManager(config.storage_config)
        self.alert_system = AlertSystem(config.alert_rules)
        
        # Буфер для батчевой обработки событий
        self.event_buffer = []
        self.buffer_lock = threading.Lock()
        
    def log_event(self, event_type, user_id, details, risk_level='LOW'):
        """Регистрация события безопасности"""
        event = {
            'event_id': self._generate_event_id(),
            'timestamp': datetime.now(timezone.utc).isoformat(),
            'event_type': event_type,
            'user_id': user_id,
            'details': details,
            'risk_level': risk_level,
            'source_ip': self._get_client_ip(),
            'session_id': self._get_session_id(),
            'system_info': self._get_system_info()
        }
        
        # Добавление цифровой подписи
        event['signature'] = self.encryption_manager.sign_event(event)
        
        # Асинхронная обработка события
        self._queue_event_for_processing(event)
        
        # Немедленная обработка критических событий
        if risk_level in ['HIGH', 'CRITICAL']:
            self._process_critical_event(event)
    
    def _process_critical_event(self, event):
        """Немедленная обработка критических событий"""
        # Отправка уведомления администраторам
        self.alert_system.send_immediate_alert(event)
        
        # Запись в защищенный журнал высокого приоритета
        self.storage_manager.store_critical_event(event)
        
        # Активация дополнительных мер безопасности при необходимости
        if event['event_type'] == 'MULTIPLE_FAILED_ATTEMPTS':
            self._trigger_enhanced_security(event['user_id'])
```

Агенты сбора событий интегрируются в каждый компонент системы и регистрируют все значимые операции. Они обеспечивают первичную фильтрацию событий и их предварительную обработку перед передачей в центральную систему.

Центральный сервер обработки журналов выполняет корреляцию событий, анализ паттернов активности и обнаружение аномалий. Он также отвечает за применение правил безопасности и генерацию предупреждений.

## Регистрация событий биометрической аутентификации

Система аудита должна регистрировать все попытки биометрической аутентификации с детальной информацией, необходимой для анализа безопасности и расследования инцидентов. Каждое событие аутентификации содержит множество параметров, характеризующих контекст и результат операции.

```python
class BiometricAuditLogger:
    def __init__(self, audit_system, anomaly_detector):
        self.audit_system = audit_system
        self.anomaly_detector = anomaly_detector
        self.failed_attempts_cache = {}
        
    def log_authentication_attempt(self, user_id, face_data, recognition_result):
        """Регистрация попытки биометрической аутентификации"""
        
        # Анализ качества биометрических данных
        quality_metrics = self._analyze_biometric_quality(face_data)
        
        # Оценка подозрительности попытки
        suspicion_score = self._calculate_suspicion_score(user_id, recognition_result, quality_metrics)
        
        event_details = {
            'authentication_result': recognition_result['success'],
            'confidence_score': recognition_result.get('confidence', 0.0),
            'liveness_score': recognition_result.get('liveness_score', 0.0),
            'quality_metrics': quality_metrics,
            'processing_time': recognition_result.get('processing_time', 0.0),
            'suspicion_score': suspicion_score,
            'face_location': recognition_result.get('face_location'),
            'multiple_faces_detected': recognition_result.get('multiple_faces', False),
            'device_info': self._get_device_info(),
            'environmental_conditions': self._assess_environment(face_data)
        }
        
        # Определение уровня риска
        risk_level = self._determine_risk_level(recognition_result, suspicion_score)
        
        # Регистрация события
        self.audit_system.log_event(
            event_type='BIOMETRIC_AUTHENTICATION',
            user_id=user_id,
            details=event_details,
            risk_level=risk_level
        )
        
        # Обновление статистики неудачных попыток
        if not recognition_result['success']:
            self._update_failed_attempts(user_id)
        else:
            self._reset_failed_attempts(user_id)
    
    def _analyze_biometric_quality(self, face_data):
        """Анализ качества биометрических данных"""
        return {
            'image_resolution': face_data.get('resolution', 'unknown'),
            'brightness_level': face_data.get('brightness', 0),
            'contrast_level': face_data.get('contrast', 0),
            'blur_level': face_data.get('blur_score', 0),
            'face_size_pixels': face_data.get('face_size', 0),
            'pose_angle': face_data.get('pose_estimation', {}),
            'occlusion_detected': face_data.get('occlusion', False)
        }
    
    def _calculate_suspicion_score(self, user_id, result, quality):
        """Расчет оценки подозрительности попытки"""
        suspicion_factors = []
        
        # Низкое качество изображения
        if quality['blur_level'] > 0.7:
            suspicion_factors.append(('high_blur', 0.3))
        
        # Необычные условия освещения
        if quality['brightness_level'] < 50 or quality['brightness_level'] > 200:
            suspicion_factors.append(('unusual_lighting', 0.2))
        
        # Низкая уверенность при успешном распознавании
        if result['success'] and result.get('confidence', 1.0) < 0.7:
            suspicion_factors.append(('low_confidence_success', 0.4))
        
        # Множественные неудачные попытки
        failed_count = self.failed_attempts_cache.get(user_id, 0)
        if failed_count > 3:
            suspicion_factors.append(('repeated_failures', min(0.8, failed_count * 0.1)))
        
        # Расчет общей оценки
        total_score = sum(score for _, score in suspicion_factors)
        return min(1.0, total_score)
```

## Мониторинг аномальной активности

Система мониторинга должна автоматически выявлять паттерны поведения, указывающие на возможные атаки или нарушения безопасности. Анализ проводится в реальном времени с использованием статистических методов и машинного обучения.

```python
class AnomalyDetectionSystem:
    def __init__(self, config):
        self.config = config
        self.baseline_models = {}
        self.alert_thresholds = config.alert_thresholds
        self.time_windows = config.time_windows
        
    def analyze_user_behavior(self, user_id, recent_events):
        """Анализ поведения пользователя на предмет аномалий"""
        anomalies = []
        
        # Анализ временных паттернов
        time_anomalies = self._detect_temporal_anomalies(user_id, recent_events)
        anomalies.extend(time_anomalies)
        
        # Анализ геолокационных данных
        location_anomalies = self._detect_location_anomalies(user_id, recent_events)
        anomalies.extend(location_anomalies)
        
        # Анализ биометрических характеристик
        biometric_anomalies = self._detect_biometric_anomalies(user_id, recent_events)
        anomalies.extend(biometric_anomalies)
        
        # Анализ паттернов неудач
        failure_anomalies = self._detect_failure_patterns(user_id, recent_events)
        anomalies.extend(failure_anomalies)
        
        return anomalies
    
    def _detect_temporal_anomalies(self, user_id, events):
        """Обнаружение временных аномалий"""
        anomalies = []
        
        # Получение исторических данных пользователя
        historical_times = self._get_user_access_times(user_id)
        
        if len(historical_times) < 10:  # Недостаточно данных для анализа
            return anomalies
        
        # Анализ времени доступа
        for event in events:
            access_time = datetime.fromisoformat(event['timestamp'])
            hour = access_time.hour
            day_of_week = access_time.weekday()
            
            # Проверка на необычное время доступа
            if not self._is_typical_access_time(user_id, hour, day_of_week, historical_times):
                anomalies.append({
                    'type': 'UNUSUAL_ACCESS_TIME',
                    'severity': 'MEDIUM',
                    'description': f'Доступ в нетипичное время: {access_time.strftime("%H:%M")}',
                    'event_id': event['event_id']
                })
        
        return anomalies
    
    def _detect_biometric_anomalies(self, user_id, events):
        """Обнаружение аномалий в биометрических данных"""
        anomalies = []
        
        # Получение базовых биометрических характеристик пользователя
        baseline_metrics = self._get_user_biometric_baseline(user_id)
        
        for event in events:
            if event['event_type'] == 'BIOMETRIC_AUTHENTICATION':
                current_metrics = event['details']['quality_metrics']
                
                # Сравнение с базовыми характеристиками
                deviation_score = self._calculate_biometric_deviation(
                    baseline_metrics, current_metrics
                )
                
                if deviation_score > self.alert_thresholds['biometric_deviation']:
                    anomalies.append({
                        'type': 'BIOMETRIC_DEVIATION',
                        'severity': 'HIGH',
                        'description': f'Существенное отклонение биометрических характеристик',
                        'deviation_score': deviation_score,
                        'event_id': event['event_id']
                    })
        
        return anomalies
```

## Защищенное хранение журналов аудита

Журналы аудита должны быть защищены от модификации и удаления, поскольку они могут содержать критически важную информацию для расследования инцидентов безопасности. Система использует криптографические методы для обеспечения целостности и неотрекаемости записей.

```python
class SecureAuditStorage:
    def __init__(self, storage_config, signing_key):
        self.storage_config = storage_config
        self.signing_key = signing_key
        self.merkle_tree = MerkleTree()
        
    def store_audit_record(self, audit_record):
        """Безопасное сохранение записи аудита"""
        # Добавление временной метки и порядкового номера
        enriched_record = self._enrich_record(audit_record)
        
        # Создание цифровой подписи
        signature = self._sign_record(enriched_record)
        enriched_record['signature'] = signature
        
        # Добавление в дерево Меркла для защиты целостности
        record_hash = self._hash_record(enriched_record)
        self.merkle_tree.add_leaf(record_hash)
        
        # Сохранение в основное хранилище
        storage_id = self._store_in_primary_storage(enriched_record)
        
        # Создание резервной копии в неизменяемом хранилище
        self._store_in_immutable_storage(enriched_record, storage_id)
        
        return storage_id
    
    def verify_audit_integrity(self, storage_id):
        """Проверка целостности записи аудита"""
        # Получение записи из хранилища
        record = self._retrieve_from_storage(storage_id)
        
        if not record:
            return False, "Запись не найдена"
        
        # Проверка цифровой подписи
        signature_valid = self._verify_signature(record)
        
        if not signature_valid:
            return False, "Недействительная цифровая подпись"
        
        # Проверка целостности через дерево Меркла
        record_hash = self._hash_record(record)
        merkle_valid = self.merkle_tree.verify_leaf(record_hash)
        
        if not merkle_valid:
            return False, "Нарушена целостность в дереве Меркла"
        
        return True, "Запись аудита корректна"
```

Система аудита и мониторинга обеспечивает комплексную защиту биометрической системы путем непрерывного наблюдения за всеми операциями, автоматического обнаружения аномалий и надежного сохранения доказательств для последующего анализа. Это позволяет не только предотвращать атаки, но и проводить эффективное расследование инцидентов безопасности.