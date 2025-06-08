# 2.3. Криптографическая защита биометрических шаблонов

Защита биометрических данных требует специальных криптографических подходов, учитывающих уникальные особенности биометрической информации. В отличие от традиционных данных, биометрические шаблоны должны оставаться пригодными для сравнения даже в зашифрованном виде, что создает дополнительные технические требования к системе шифрования.

## Архитектура криптографической защиты

Система криптографической защиты биометрических данных строится на принципе многоуровневой защиты, где различные компоненты данных защищаются разными методами в зависимости от их назначения и требований к доступу.

```python
class BiometricCryptographicSystem:
    def __init__(self, master_key, key_derivation_salt):
        self.master_key = master_key
        self.kdf_salt = key_derivation_salt
        
        # Генерация производных ключей для различных целей
        self.template_key = self._derive_key("TEMPLATE_ENCRYPTION")
        self.integrity_key = self._derive_key("INTEGRITY_PROTECTION") 
        self.index_key = self._derive_key("SEARCHABLE_ENCRYPTION")
        
        # Инициализация криптографических компонентов
        self.symmetric_cipher = Fernet(self.template_key)
        self.integrity_hmac = hmac.new(self.integrity_key, digestmod=hashlib.sha256)
        
    def _derive_key(self, purpose):
        """Получение производного ключа для конкретной цели"""
        return PBKDF2(
            self.master_key,
            self.kdf_salt + purpose.encode(),
            dkLen=32,
            count=100000,
            hmac_hash_module=SHA256
        )
```

Первый уровень защиты обеспечивает шифрование самих биометрических шаблонов с использованием симметричных алгоритмов. Это защищает данные от прямого доступа злоумышленников, но требует расшифровки для выполнения операций сравнения.

Второй уровень включает защиту целостности данных с помощью криптографических хэш-функций и кодов аутентификации сообщений. Это позволяет обнаруживать любые несанкционированные изменения биометрических шаблонов.

Третий уровень предоставляет возможности поиска в зашифрованных данных без их расшифровки, что критически важно для производительности системы при большом количестве пользователей.

## Шифрование биометрических шаблонов

Шифрование биометрических шаблонов должно обеспечивать максимальную защиту при минимальном влиянии на производительность системы. Для этого используется гибридный подход, объединяющий симметричное и асимметричное шифрование.

```python
class BiometricTemplateEncryption:
    def __init__(self, public_key, private_key):
        self.public_key = public_key
        self.private_key = private_key
        self.rng = SystemRandom()
        
    def encrypt_template(self, biometric_template, user_id):
        """Шифрование биометрического шаблона"""
        # Генерация случайного ключа для симметричного шифрования
        session_key = os.urandom(32)
        
        # Преобразование шаблона в байты с добавлением метаданных
        template_data = {
            'template': biometric_template.tolist(),
            'user_id': user_id,
            'timestamp': datetime.now().isoformat(),
            'version': '1.0'
        }
        
        template_bytes = json.dumps(template_data).encode('utf-8')
        
        # Добавление случайной соли для защиты от атак по словарю
        salt = self.rng.randbytes(16)
        salted_template = salt + template_bytes
        
        # Симметричное шифрование данных
        cipher = ChaCha20_Poly1305.new(key=session_key)
        ciphertext, auth_tag = cipher.encrypt_and_digest(salted_template)
        
        # Асимметричное шифрование ключа сессии
        cipher_rsa = PKCS1_OAEP.new(self.public_key)
        encrypted_session_key = cipher_rsa.encrypt(session_key)
        
        # Объединение всех компонентов
        encrypted_template = {
            'encrypted_key': encrypted_session_key,
            'nonce': cipher.nonce,
            'ciphertext': ciphertext,
            'auth_tag': auth_tag
        }
        
        return encrypted_template
    
    def decrypt_template(self, encrypted_template):
        """Расшифровка биометрического шаблона"""
        try:
            # Расшифровка ключа сессии
            cipher_rsa = PKCS1_OAEP.new(self.private_key)
            session_key = cipher_rsa.decrypt(encrypted_template['encrypted_key'])
            
            # Расшифровка данных
            cipher = ChaCha20_Poly1305.new(
                key=session_key,
                nonce=encrypted_template['nonce']
            )
            
            plaintext = cipher.decrypt_and_verify(
                encrypted_template['ciphertext'],
                encrypted_template['auth_tag']
            )
            
            # Удаление соли и восстановление данных
            template_bytes = plaintext[16:]
            template_data = json.loads(template_bytes.decode('utf-8'))
            
            return np.array(template_data['template']), template_data
            
        except Exception as e:
            raise DecryptionError(f"Ошибка расшифровки шаблона: {e}")
```

## Защита целостности данных

Обеспечение целостности биометрических данных критически важно для корректной работы системы распознавания. Любые изменения в биометрических шаблонах могут привести к ложным результатам идентификации.

```python
class BiometricIntegrityProtection:
    def __init__(self, integrity_key):
        self.integrity_key = integrity_key
        
    def create_integrity_proof(self, template_data, metadata):
        """Создание доказательства целостности"""
        # Объединение данных для хэширования
        combined_data = {
            'template': template_data.tolist() if isinstance(template_data, np.ndarray) else template_data,
            'metadata': metadata,
            'timestamp': datetime.now().isoformat()
        }
        
        data_bytes = json.dumps(combined_data, sort_keys=True).encode('utf-8')
        
        # Создание HMAC для защиты от модификации
        integrity_hmac = hmac.new(
            self.integrity_key,
            data_bytes,
            hashlib.sha256
        )
        
        # Создание контрольной суммы
        checksum = hashlib.sha256(data_bytes).hexdigest()
        
        integrity_proof = {
            'hmac': integrity_hmac.hexdigest(),
            'checksum': checksum,
            'algorithm': 'HMAC-SHA256',
            'timestamp': combined_data['timestamp']
        }
        
        return integrity_proof
    
    def verify_integrity(self, template_data, metadata, integrity_proof):
        """Проверка целостности данных"""
        # Воссоздание данных для проверки
        combined_data = {
            'template': template_data.tolist() if isinstance(template_data, np.ndarray) else template_data,
            'metadata': metadata,
            'timestamp': integrity_proof['timestamp']
        }
        
        data_bytes = json.dumps(combined_data, sort_keys=True).encode('utf-8')
        
        # Проверка HMAC
        expected_hmac = hmac.new(
            self.integrity_key,
            data_bytes,
            hashlib.sha256
        ).hexdigest()
        
        hmac_valid = hmac.compare_digest(expected_hmac, integrity_proof['hmac'])
        
        # Проверка контрольной суммы
        expected_checksum = hashlib.sha256(data_bytes).hexdigest()
        checksum_valid = (expected_checksum == integrity_proof['checksum'])
        
        return hmac_valid and checksum_valid
```

## Поиск в зашифрованных данных

Для обеспечения эффективного поиска пользователей без расшифровки всех биометрических шаблонов используется схема поиска в зашифрованных данных, основанная на создании защищенных индексов.

```python
class SearchableEncryption:
    def __init__(self, search_key):
        self.search_key = search_key
        
    def create_search_index(self, user_id, biometric_features):
        """Создание индекса для поиска в зашифрованных данных"""
        # Извлечение ключевых характеристик для индексирования
        feature_hash = self._hash_features(biometric_features)
        
        # Создание защищенного индекса
        index_data = f"{user_id}:{feature_hash}"
        
        # Шифрование индекса
        cipher = Fernet(self.search_key)
        encrypted_index = cipher.encrypt(index_data.encode())
        
        return {
            'encrypted_index': encrypted_index,
            'feature_signature': feature_hash[:16]  # Частичная подпись для быстрого поиска
        }
    
    def _hash_features(self, features):
        """Создание хэша биометрических признаков"""
        # Нормализация и квантование признаков для устойчивости к шуму
        normalized_features = self._normalize_features(features)
        quantized_features = self._quantize_features(normalized_features)
        
        # Создание хэша
        feature_bytes = quantized_features.tobytes()
        return hashlib.sha256(feature_bytes).hexdigest()
```

Данная архитектура криптографической защиты обеспечивает комплексную безопасность биометрических данных, сохраняя при этом возможность эффективного поиска и сравнения шаблонов. Использование современных криптографических алгоритмов и принципов обеспечивает защиту от различных типов атак на биометрические данные.