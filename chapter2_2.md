# 2.2. Проектирование модулей защиты от спуфинга и презентационных атак

Разработка эффективной защиты от презентационных атак требует комплексного подхода, объединяющего несколько методов обнаружения попыток обмана. Модуль защиты от спуфинга должен интегрироваться в основной процесс распознавания лиц, не нарушая его работу, но обеспечивая надежную защиту от различных типов атак.

## Архитектура модуля обнаружения живого лица

Модуль обнаружения живого лица проектируется как независимый компонент, который может быть интегрирован в существующую систему распознавания. Архитектура основана на принципе многоуровневой проверки, где каждый уровень анализирует различные аспекты представленного биометрического образца.

```python
class LivenessDetector:
    def __init__(self, config):
        self.blink_detector = BlinkDetector(config.blink_threshold)
        self.texture_analyzer = TextureAnalyzer(config.texture_model)
        self.movement_tracker = MovementTracker(config.movement_params)
        self.quality_assessor = QualityAssessor(config.quality_thresholds)
        
    def analyze_liveness(self, frame_sequence):
        """Комплексный анализ подлинности лица"""
        results = {
            'blink_detected': False,
            'texture_valid': False,
            'movement_natural': False,
            'quality_sufficient': False,
            'overall_score': 0.0
        }
        
        # Проверка качества изображения
        results['quality_sufficient'] = self.quality_assessor.assess(frame_sequence)
        if not results['quality_sufficient']:
            return results
        
        # Анализ моргания
        results['blink_detected'] = self.blink_detector.detect_blink(frame_sequence)
        
        # Анализ текстуры
        results['texture_valid'] = self.texture_analyzer.analyze_texture(frame_sequence)
        
        # Анализ движения
        results['movement_natural'] = self.movement_tracker.analyze_movement(frame_sequence)
        
        # Расчет общей оценки подлинности
        results['overall_score'] = self._calculate_liveness_score(results)
        
        return results
```

Первый уровень проверки включает анализ качества изображения. Система оценивает разрешение, четкость, освещенность и другие технические характеристики кадра. Изображения низкого качества, характерные для печатных фотографий или экранов мониторов, отфильтровываются на этом этапе.

Второй уровень осуществляет детекцию моргания как наиболее простого и эффективного индикатора живого лица. Алгоритм анализирует изменения в геометрии глаз на протяжении нескольких кадров, выявляя характерные паттерны моргания.

## Алгоритм детекции моргания

Детекция моргания основана на анализе соотношения сторон глаз (Eye Aspect Ratio, EAR), которое значительно изменяется в процессе моргания. Алгоритм отслеживает ключевые точки глаз и вычисляет динамику их изменения.

```python
class BlinkDetector:
    def __init__(self, threshold=0.25, consecutive_frames=3):
        self.ear_threshold = threshold
        self.consecutive_frames = consecutive_frames
        self.frame_buffer = []
        
    def calculate_eye_aspect_ratio(self, eye_landmarks):
        """Вычисление соотношения сторон глаза"""
        # Вертикальные расстояния
        vertical_1 = np.linalg.norm(eye_landmarks[1] - eye_landmarks[5])
        vertical_2 = np.linalg.norm(eye_landmarks[2] - eye_landmarks[4])
        
        # Горизонтальное расстояние
        horizontal = np.linalg.norm(eye_landmarks[0] - eye_landmarks[3])
        
        # Соотношение сторон
        ear = (vertical_1 + vertical_2) / (2.0 * horizontal)
        return ear
    
    def detect_blink(self, frame_sequence):
        """Обнаружение моргания в последовательности кадров"""
        ear_values = []
        blink_detected = False
        
        for frame in frame_sequence:
            landmarks = self._extract_facial_landmarks(frame)
            if landmarks is not None:
                left_eye_ear = self.calculate_eye_aspect_ratio(landmarks['left_eye'])
                right_eye_ear = self.calculate_eye_aspect_ratio(landmarks['right_eye'])
                avg_ear = (left_eye_ear + right_eye_ear) / 2.0
                ear_values.append(avg_ear)
        
        if len(ear_values) >= self.consecutive_frames * 2:
            # Поиск паттерна моргания (снижение и восстановление EAR)
            blink_detected = self._analyze_blink_pattern(ear_values)
        
        return blink_detected
    
    def _analyze_blink_pattern(self, ear_values):
        """Анализ паттерна моргания"""
        baseline = np.mean(ear_values)
        
        for i in range(len(ear_values) - self.consecutive_frames):
            window = ear_values[i:i + self.consecutive_frames]
            if all(ear < baseline * (1 - self.ear_threshold) for ear in window):
                # Найдено закрытие глаза, проверяем открытие
                for j in range(i + self.consecutive_frames, len(ear_values)):
                    if ear_values[j] > baseline * (1 - self.ear_threshold / 2):
                        return True
        
        return False
```

## Анализ текстурных характеристик

Анализ текстуры позволяет различать живую кожу от печатных изображений или экранов мониторов. Живая кожа обладает уникальными характеристиками: микро-текстурой, естественными неровностями и специфическими световыми свойствами.

```python
class TextureAnalyzer:
    def __init__(self, model_path):
        self.texture_model = self._load_texture_model(model_path)
        self.feature_extractor = LocalBinaryPatternExtractor()
        
    def analyze_texture(self, frame_sequence):
        """Анализ текстурных характеристик лица"""
        texture_scores = []
        
        for frame in frame_sequence:
            face_region = self._extract_face_region(frame)
            if face_region is not None:
                # Извлечение локальных бинарных паттернов
                lbp_features = self.feature_extractor.extract_lbp(face_region)
                
                # Анализ градиентов
                gradient_features = self._analyze_gradients(face_region)
                
                # Анализ частотных характеристик
                frequency_features = self._analyze_frequency_domain(face_region)
                
                # Объединение признаков
                combined_features = np.concatenate([
                    lbp_features, gradient_features, frequency_features
                ])
                
                # Классификация с помощью обученной модели
                score = self.texture_model.predict_proba([combined_features])[0][1]
                texture_scores.append(score)
        
        # Усреднение результатов по всем кадрам
        return np.mean(texture_scores) > 0.5 if texture_scores else False
    
    def _analyze_gradients(self, face_region):
        """Анализ градиентов изображения"""
        gray = cv2.cvtColor(face_region, cv2.COLOR_BGR2GRAY)
        
        # Градиенты по Собелю
        grad_x = cv2.Sobel(gray, cv2.CV_64F, 1, 0, ksize=3)
        grad_y = cv2.Sobel(gray, cv2.CV_64F, 0, 1, ksize=3)
        
        # Магнитуда и направление градиентов
        magnitude = np.sqrt(grad_x**2 + grad_y**2)
        direction = np.arctan2(grad_y, grad_x)
        
        # Статистические характеристики
        features = [
            np.mean(magnitude), np.std(magnitude),
            np.mean(direction), np.std(direction),
            np.percentile(magnitude, [25, 50, 75])
        ]
        
        return np.array(features).flatten()
```

## Анализ движения и поведенческих паттернов

Анализ естественности движений лица помогает выявить искусственные паттерны, характерные для воспроизведения видеозаписей или использования масок. Естественные движения человека обладают специфическими характеристиками: плавностью, непредсказуемостью и согласованностью различных частей лица.

Система отслеживает движения ключевых точек лица во времени, анализируя их скорость, ускорение и корреляцию между различными областями. Искусственные движения, характерные для масок или воспроизводимых видео, обычно имеют нехарактерные паттерны или чрезмерную синхронизацию.

Интеграция всех методов обнаружения в единую систему обеспечивает высокую надежность защиты от различных типов презентационных атак. Каждый метод вносит свой вклад в общую оценку подлинности, а итоговое решение принимается на основе комбинированного анализа всех факторов.