# 13. Написание фронтенда

## 13.1. Принцип: фронтенд никогда не подключается к Fabric напрямую
Фронтенд обращается только к Express-API из конспекта 12 обычными HTTP-запросами — Fabric, wallet, сертификаты полностью скрыты на сервере. С точки зрения React-приложения это ничем не отличается от работы с любым другим REST API.

```
React (браузер)  →  fetch()  →  Express API (localhost:7000)  →  Fabric-сеть
```

## 13.2. Подготовка проекта
```bash
npm create vite@latest hl-frontend -- --template react
cd hl-frontend
npm install
```
```
# .env
VITE_API_URL=http://localhost:7000
```

## 13.3. Структура фронтенд-проекта
```
src/
├── services/
│   └── CarsService.js
├── context/
│   └── IdentityContext.jsx
├── components/
│   ├── CarCard.jsx
│   ├── AddCarForm.jsx
│   └── EnrollUserForm.jsx
├── pages/
│   └── CarsPage.jsx
├── App.jsx
└── main.jsx
```

## 13.4. Сервисный слой — CarsService.js
```js
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

    transferCar(organization, userID, id, newOwner) {
        return request('/transferCar', { organization, userID, id, newOwner });
    }
}

export default new CarsService();
```

## 13.5. Контекст текущей личности
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

## 13.6. Форма регистрации пользователя
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
            <input value={userID} onChange={(e) => setUserID(e.target.value)} placeholder="Имя нового пользователя" />
            <button type="submit" disabled={loading}>{loading ? 'Регистрация...' : 'Зарегистрировать'}</button>
            {status && <p style={{ color: status.type === 'error' ? 'red' : 'green' }}>{status.text}</p>}
        </form>
    );
}

export default EnrollUserForm;
```

## 13.7. Форма добавления машины
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
            setForm({ id: '', color: '', brand: '', owner: '' });
            onCarAdded();
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

## 13.8. Страница со списком машин
```jsx
// src/pages/CarsPage.jsx
import { useState, useEffect, useContext, useCallback } from 'react'
import { IdentityContext } from '../context/IdentityContext.jsx'
import CarsService from '../services/CarsService.js'
import AddCarForm from '../components/AddCarForm.jsx'
import EnrollUserForm from '../components/EnrollUserForm.jsx'
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

    useEffect(() => { loadCars(); }, [loadCars]);

    return (
        <div>
            <h1>Машины в реестре</h1>

            <EnrollUserForm />
            <AddCarForm onCarAdded={loadCars} />

            {loading && <p>Загрузка...</p>}
            {error && <p style={{ color: 'red' }}>{error}</p>}

            <div className="cars-grid">
                {cars.map(car => <CarCard key={car.ID} car={car} />)}
            </div>
        </div>
    );
}

export default CarsPage;
```

## 13.9. CarCard.jsx
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

## 13.10. App.jsx и main.jsx
```jsx
// App.jsx
import CarsPage from './pages/CarsPage.jsx'
import { IdentityProvider } from './context/IdentityContext.jsx'

function App() {
    return (
        <IdentityProvider>
            <CarsPage />
        </IdentityProvider>
    );
}

export default App;
```

## 13.11. Подписка на события реестра в реальном времени (опционально)
Использует эндпоинт `/events/:organization/:userID` из конспекта 12 через Server-Sent Events:
```jsx
useEffect(() => {
    const eventSource = new EventSource(
        `${import.meta.env.VITE_API_URL}/events/${organization}/${userID}`
    );

    eventSource.onmessage = () => {
        loadCars(); // при любом событии реестра — обновляем список
    };

    return () => eventSource.close();
}, [organization, userID, loadCars]);
```

## 13.12. Обязательный чек-лист перед запуском фронтенда
- [ ] Сеть Fabric поднята (`docker ps` показывает контейнеры peer/orderer/ca)
- [ ] Chaincode развёрнут (`peer lifecycle chaincode querycommitted`)
- [ ] API-сервер запущен (`npm run dev` в `application-javascript`, слушает `API_PORT`)
- [ ] `VITE_API_URL` во фронтенде указывает на реальный адрес и порт API
- [ ] CORS разрешён на API (пакет `cors`, уже подключён в конспекте 12)

## 13.13. Итоговая полная схема всего курса
```
Fabric-сеть (конспекты 01–07)
        │  connection-org1.json + TLS
Express API (конспект 12) — wallet, Gateway, submitTransaction/evaluateTransaction
        │  HTTP/JSON (fetch)
React-фронтенд (этот конспект) — CarsService, IdentityContext, формы и список
```
Каждый слой отвечает строго за своё: сеть — за консенсус и хранение, API — за мост между JSON-миром браузера и gRPC/TLS-миром Fabric, фронтенд — за пользовательский интерфейс поверх обычного REST API.
