# Написание фронтенда для взаимодействия с Fabric

Фронтенд не подключается к сети Hyperledger Fabric напрямую — он обращается к написанному ранее Express-API (`enrollAdmin`, `enrollUser`, `addCar`, `getAllCars`), а всю работу с CA, wallet и Gateway берёт на себя сервер. С точки зрения фронтенда это обычное REST API — тот же подход, что и в теме про React + fetch/axios к любому другому backend.

## Структура фронтенд-проекта
```
src/
├── services/
│   └── CarsService.js       # Весь код обращения к API — в одном месте
├── context/
│   └── IdentityContext.jsx   # Текущая организация и пользователь, доступные всему приложению
├── components/
│   ├── CarCard.jsx
│   ├── AddCarForm.jsx
│   └── EnrollUserForm.jsx
├── pages/
│   ├── CarsPage.jsx
│   └── LoginPage.jsx
├── App.jsx
└── main.jsx
```

## Шаг 1. Переменная окружения с адресом API
Адрес API не стоит "зашивать" прямо в код компонентов — по той же причине, по которой в теме про backend конфигурация выносилась в `.env`:
```
# .env
VITE_API_URL=http://localhost:7000
```
*Обратите внимание: в проектах на Vite переменные окружения, которые должны быть доступны в браузерном коде, обязаны начинаться с префикса `VITE_` — иначе Vite не включит их в сборку.*

## Шаг 2. Сервисный слой — вся логика обращения к API в одном месте
```js
// src/services/CarsService.js
const API_URL = import.meta.env.VITE_API_URL;

async function request(endpoint, body) {
    const response = await fetch(`${API_URL}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
    });

    const data = await response.json();

    if (!response.ok || !data.success) {
        throw new Error(data.error || 'Неизвестная ошибка сервера');
    }

    return data.data ?? data.message;
}

class CarsService {
    enrollAdmin(organization) {
        return request('/enrollAdmin', { organization });
    }

    enrollUser(organization, userID) {
        return request('/enrollUser', { organization, userID });
    }

    addCar(organization, userID, id, color, brand, owner) {
        return request('/addCar', { organization, userID, id, color, brand, owner });
    }

    getAllCars(organization, userID) {
        return request('/getAllCars', { organization, userID });
    }
}

export default new CarsService();
```
Вынесение общей функции `request(...)` избавляет от дублирования `fetch` + разбора JSON + проверки ошибок в каждом методе отдельно — при необходимости (например, добавить заголовок авторизации) правки вносятся один раз, в одном месте, а не в каждом методе сервиса.

## Шаг 3. Контекст текущей личности (организация + пользователь)
Так как почти каждый запрос к API требует `organization` и `userID`, разумно хранить их не в каждом компоненте отдельно, а в общем контексте (см. тему про Context API):
```jsx
// src/context/IdentityContext.jsx
import { createContext, useState } from 'react'

export const IdentityContext = createContext(null);

export function IdentityProvider({ children }) {
    const [organization, setOrganization] = useState('org1');
    const [userID, setUserID] = useState('user1');

    return (
        <IdentityContext.Provider value={{ organization, setOrganization, userID, setUserID }}>
            {children}
        </IdentityContext.Provider>
    );
}
```
Такой контекст удобно дополнить сохранением значений в `localStorage`, чтобы выбранный пользователь не сбрасывался при перезагрузке страницы (в отличие от чувствительных данных вроде приватных ключей, сами по себе `organization`/`userID` не секретны — сертификат и ключ остаются на сервере, во внутреннем `wallet`, и никогда не передаются на фронтенд).

## Шаг 4. Форма регистрации пользователя
```jsx
// src/components/EnrollUserForm.jsx
import { useState, useContext } from 'react'
import { IdentityContext } from '../context/IdentityContext.jsx'
import CarsService from '../services/CarsService.js'

function EnrollUserForm() {
    const { organization } = useContext(IdentityContext);
    const [userID, setUserID] = useState('');
    const [status, setStatus] = useState(null);
    const [loading, setLoading] = useState(false);

    async function handleSubmit(e) {
        e.preventDefault();
        setLoading(true);
        setStatus(null);

        try {
            const message = await CarsService.enrollUser(organization, userID);
            setStatus({ type: 'success', text: message });
        } catch (error) {
            setStatus({ type: 'error', text: error.message });
        } finally {
            setLoading(false);
        }
    }

    return (
        <form onSubmit={handleSubmit}>
            <input
                value={userID}
                onChange={(e) => setUserID(e.target.value)}
                placeholder="Имя нового пользователя"
            />
            <button type="submit" disabled={loading}>
                {loading ? 'Регистрация...' : 'Зарегистрировать'}
            </button>

            {status && (
                <p style={{ color: status.type === 'error' ? 'red' : 'green' }}>
                    {status.text}
                </p>
            )}
        </form>
    );
}

export default EnrollUserForm;
```
Состояния `loading` и `status` — стандартный паттерн для любого запроса к серверу (уже встречался в темах про useState/useEffect и Promise): пока запрос выполняется, кнопка блокируется, а после завершения пользователь видит либо успех, либо текст ошибки, пришедший напрямую от API (например, "Пользователь user1 уже существует").

## Шаг 5. Форма добавления машины
```jsx
// src/components/AddCarForm.jsx
import { useState, useContext } from 'react'
import { IdentityContext } from '../context/IdentityContext.jsx'
import CarsService from '../services/CarsService.js'

function AddCarForm({ onCarAdded }) {
    const { organization, userID } = useContext(IdentityContext);
    const [form, setForm] = useState({ id: '', color: '', brand: '', owner: '' });
    const [error, setError] = useState(null);

    function handleChange(e) {
        setForm({ ...form, [e.target.name]: e.target.value });
    }

    async function handleSubmit(e) {
        e.preventDefault();
        setError(null);

        try {
            await CarsService.addCar(organization, userID, form.id, form.color, form.brand, form.owner);
            setForm({ id: '', color: '', brand: '', owner: '' }); // очищаем форму после успеха
            onCarAdded(); // сообщаем родителю, что список нужно обновить
        } catch (err) {
            setError(err.message);
        }
    }

    return (
        <form onSubmit={handleSubmit}>
            <input name="id" value={form.id} onChange={handleChange} placeholder="ID" />
            <input name="color" value={form.color} onChange={handleChange} placeholder="Цвет" />
            <input name="brand" value={form.brand} onChange={handleChange} placeholder="Марка" />
            <input name="owner" value={form.owner} onChange={handleChange} placeholder="Владелец" />
            <button type="submit">Добавить машину</button>

            {error && <p style={{ color: 'red' }}>{error}</p>}
        </form>
    );
}

export default AddCarForm;
```
Вычисляемый ключ `[e.target.name]` (см. тему про продвинутую работу с объектами) позволяет обрабатывать изменение любого из четырёх полей формы одной функцией `handleChange`, вместо четырёх отдельных обработчиков.

## Шаг 6. Список машин с загрузкой через useEffect
```jsx
// src/pages/CarsPage.jsx
import { useState, useEffect, useContext, useCallback } from 'react'
import { IdentityContext } from '../context/IdentityContext.jsx'
import CarsService from '../services/CarsService.js'
import AddCarForm from '../components/AddCarForm.jsx'
import CarCard from '../components/CarCard.jsx'

function CarsPage() {
    const { organization, userID } = useContext(IdentityContext);
    const [cars, setCars] = useState([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);

    const loadCars = useCallback(async () => {
        setLoading(true);
        setError(null);

        try {
            const data = await CarsService.getAllCars(organization, userID);
            setCars(data);
        } catch (err) {
            setError(err.message);
        } finally {
            setLoading(false);
        }
    }, [organization, userID]);

    // Загружаем список при первом рендере и при смене организации/пользователя
    useEffect(() => {
        loadCars();
    }, [loadCars]);

    return (
        <div>
            <h1>Машины в реестре</h1>

            <AddCarForm onCarAdded={loadCars} />

            {loading && <p>Загрузка...</p>}
            {error && <p style={{ color: 'red' }}>{error}</p>}

            <div className="cars-grid">
                {cars.map(car => (
                    <CarCard key={car.ID} car={car} />
                ))}
            </div>
        </div>
    );
}

export default CarsPage;
```
`onCarAdded={loadCars}` — после успешного добавления машины форма сообщает странице, что список устарел, и страница заново запрашивает актуальные данные из блокчейна через тот же `evaluateTransaction('GetAllCars')` на сервере — без этого пользователь увидел бы старые данные до ручного обновления страницы.

## CarCard.jsx — карточка машины
```jsx
function CarCard({ car }) {
    return (
        <div className="car-card">
            <h3>{car.Brand}</h3>
            <p>Цвет: {car.Color}</p>
            <p>Владелец: {car.Owner}</p>
            <span className="car-id">{car.ID}</span>
        </div>
    );
}

export default CarCard;
```

## Оптимизация фронтенд-слоя

1. **Не создавать новое подключение/запрос при каждом клике без нужды** — как и на бэкенде, стоит избегать избыточных запросов: например, обновлять список машин только после реального изменения (`onCarAdded`), а не по таймеру каждую секунду.
2. **Единая обработка ошибок сервиса** (функция `request` в `CarsService`) вместо повторения проверки `response.ok`/`data.success` в каждом компоненте — то же самое соображение, что и с `asyncHandler` на бэкенде: одна точка изменения вместо дублирования логики.
3. **Разделение "личности" и "данных"**: `IdentityContext` хранит только то, кто сейчас работает с приложением (`organization`, `userID`), а не сами данные блокчейна — данные (список машин) остаются локальным состоянием конкретной страницы и запрашиваются заново при необходимости, а не кэшируются бессрочно, чтобы не показывать пользователю устаревшую версию реестра.
4. **Понятные сообщения об ошибках для пользователя.** Поскольку API уже возвращает содержательные сообщения (`"Пользователь user1 уже существует"`, `"Машина car1 уже существует"`), фронтенду достаточно просто показать `error.message` — не нужно придумывать отдельный слой перевода технических ошибок в пользовательский текст, если бэкенд изначально спроектирован с понятными сообщениями (см. тему про backend/API).
5. **Индикация состояния запроса (`loading`)** обязательна для любых операций, взаимодействующих с блокчейном — в отличие от обычного запроса к базе данных, подтверждение транзакции в Fabric занимает заметное время (endorsement + ordering + commit), и пользователю важно видеть, что запрос обрабатывается, а не считать интерфейс "зависшим".
