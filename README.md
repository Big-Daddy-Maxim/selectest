# selectest

**Шаг 1. Исправление ошибки в requirements.txt**  
Что сделал:
Проанализировал файл requirements.txt . Обнаружил строку: fastapi==999.0.0; python_version < "3.8"  
 
Проблема:
Указана явно несуществующая версия 999.0.0 
Маркер python_version < "3.8"  может вызвать ошибку при попытке установки
Кроме того строка дублирует пакет fastapi, который уже указан в первой строке (без версии).  
Решение:
Удалили строку fastapi==999.0.0; python_version < "3.8"  


**Шаг 2. Ошибка валидации Pydantic: extra_forbidden для поля database_url**  
Проблема:
При запуске приложение падало с ошибкой:  
 
Модель не принимала переданное значение DATABASE_URL, считая его лишним полем.
Анализ:
В классе Settings  поле database_url было с опечаткой и Pydantic по умолчанию запрещает передачу полей, не объявленных в модели.  
 
Решение:

database_url: str = Field(
    "postgresql+asyncpg://postgres:postgres@db:5432/postgres",      validation_alias="DATABASE_URL",  

 
**Шаг 3: Исправление интервала фонового парсинга**  
Что сделал: Фоновый парсинг выполняется каждые 5 секунд, вместо 5 мин в  .env.example (PARSE_SCHEDULE_MINUTES=5).  
Проблема: После проверки обнаружил, что в app/core/config.py поле parse_schedule_minutes не было связано с переменной окружения, либо в вызове create_scheduler в main.py не передавался параметр minutes. В результате использовалось значение по умолчанию 5, но планировщик интерпретировал его как секунды.  
 
 
Решение:
log_level: str = "INFO"
parse_schedule_minutes: int = Field(5, validation_alias="PARSE_SCHEDULE_MINUTES")

from app.core.config import settings
…
…
…
@app.on_event("startup")
async def on_startup() -> None:
    logger.info("Запуск приложения")
    await _run_parse_job()
    global _scheduler
    _scheduler = create_scheduler(
        _run_parse_job,
        minutes=settings.parse_schedule_minutes
    )
    _scheduler.start()  



 
**Шаг 4. Обработка отсутствующего города в парсере**  
Что сделал: Запустил приложение и увидел в логах повторяющуюся ошибку при парсинге вакансий.  
Проблема: В ответе API Selectel у некоторых вакансий поле city отсутствует или равно null. Код пытается обратиться item.city.name.strip(), что вызывает ошибку AttributeError: 'NoneType' object has no attribute 'name'.  
 
Решение:
"tag_name": item.tag.name,
"city_name": item.city.name.strip() if item.city else None,
"published_at": item.published_at,  

 
**Шаг 5: Утечка HTTP-соединений в парсере**  
Что сделал: Просмотрел функцию parse_and_store в app/services/parser.py.  
Проблема: HTTP-клиент создаётся через httpx.AsyncClient(...), но не закрывается после использования. Это может привести к утечке соединений и истощению ресурсов.  
 
Решение: Использовать async with для автоматического закрытия
try:
    async with httpx.AsyncClient(timeout=timeout) as client:
        page = 1
        while True:

 
**Шаг 6: Исправление несоответствия параметра функции create_scheduler**  
Что сделал: Запустил приложение и получил ошибку при старте.   
Проблема: В  main.py при создании планировщика передаётся аргумент minutes, но функция create_scheduler в app/services/scheduler.py  он называется interval. Из-за этого вызов падает с TypeError и приложение не запускается.  
 
Решение:
def create_scheduler(job: Callable[[], Awaitable[None]], minutes: int = settings.parse_schedule_minutes) -> AsyncIOScheduler:
    scheduler = AsyncIOScheduler()
    scheduler.add_job(
        job,
        trigger=IntervalTrigger(minutes=minutes),
        coalesce=True,
        max_instances=1,
    )
    return scheduler  


 
**Шаг 7: Исправление возвращаемого статуса при дубликате external_id**  
Что сделал: Проверил эндпоинт создания вакансии в app/api/v1/vacancies.py. Обнаружил, что при попытке создать вакансию с уже существующим external_id сервер возвращает 200 OK с сообщением, что вакансия уже существует.  
Проблема: Должен возвращаться код 409 Conflict, а не 200 OK. Также ответ не соответствует объявленной модели VacancyRead, что может вызвать ошибки у клиентов.  
 
Решение: Заменил JSONResponse на исключение HTTPException 409.
if existing:
    raise HTTPException(
        status_code=status.HTTP_409_CONFLICT,
        detail="Vacancy with this external_id already exists",
    )  


 
**Шаг 8: Исправление схемы VacancyUpdate для поддержки частичного обновления**  
Что сделал: Проанализировал схему VacancyUpdate в app/schemas/vacancy.py и функцию обновления в crud/vacancy.py.  
Проблема: VacancyUpdate наследует VacancyBase, где все поля обязательны. Это делает невозможным частичное обновление и клиент должен передавать все поля, иначе они сбросятся в None.  
 
 

Решение: Сделать все поля опциональными, в функции обновления учитываю только переданные поля 
class VacancyUpdate(BaseModel):
    title: Optional[str] = None
    timetable_mode_name: Optional[str] = None
    tag_name: Optional[str] = None
    city_name: Optional[str] = None
    published_at: Optional[datetime] = None
    is_remote_available: Optional[bool] = None
    is_hot: Optional[bool] = None
    external_id: Optional[int] = None
в crud/vacancy.py

 session: AsyncSession, vacancy: Vacancy, data: VacancyUpdate
) -> Vacancy:
    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(vacancy, field, value)
    await session.commit()   

    
**Шаг 9: Добавление обработки ошибок в эндпоинт**  
Что сделал: Проверил файл app/api/v1/parse.py.  
Проблема: Эндпоинт POST /parse не содержит обработки исключений. В случае ошибки при парсинге или сохранении данных клиент получит неинформативный ответ или внутреннюю ошибку сервера без пояснений.  
 
Решение: 
from fastapi import APIRouter, Depends, HTTPException
…
…
…
try:
    created_count = await parse_and_store(session)
    return {"created": created_count}
except Exception as e:
    raise HTTPException(status_code=500, detail=f"Ошибка парсинга: {str(e)}")



