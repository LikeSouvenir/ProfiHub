# Фронтенд: как React-приложение работает с нодой напрямую

В отличие от связки "фронтенд + Ethereum" (MetaMask, ethers.js/viem, подпись в браузере), в примере из учебной базы фронтенд **не подписывает транзакции сам** и не использует никакой специальной web3-библиотеки — он просто дёргает REST API ноды напрямую через `fetch`, а подпись и отправку берёт на себя сама нода (`/transactions/signAndBroadcast`, ключ уже лежит в её keystore). Это самый быстрый способ показать рабочий прототип на хакатоне: не нужен кошелёк в браузере, не нужна библиотека для подписи — только `fetch` и REST API.

## Стек
```json
{
  "react": "^19",
  "react-dom": "^19",
  "react-router-dom": "^7",
  "react-bootstrap": "^2.10",
  "bootstrap": "^5.3",
  "vite": "^8"
}
```
Обычный Vite + React + React Router проект, `react-bootstrap` — для готовых компонентов форм и карточек без написания своего CSS.

## Общая архитектура
```
src/
├── main.jsx            # точка входа, подключает bootstrap.css
├── App.jsx             # оборачивает всё в AppProvider + RouterProvider
├── core/
│   ├── context.jsx      # React Context: хранит адрес ноды, sender, contractId, метод post()
│   └── routing.jsx      # список страниц-маршрутов
└── ui/
    ├── component/
    │   ├── Header.jsx        # навигация, меняется в зависимости от авторизации
    │   ├── Authorization.jsx # форма подключения к ноде/контракту
    │   └── func/              # одна форма = один @ContractAction-метод контракта
    └── pages/                # страницы для чтения состояния контракта
```
Главная идея: **один компонент в `func/` — это один метод контракта**. Компонентов ровно столько же, сколько `@ContractAction`-методов в Java-контракте — прямое соответствие фронтенда и бизнес-логики контракта.

## core/context.jsx — сердце интеграции
```jsx
const AppProvider = ({ children }) => {
    const [url, setUrl] = useState("")
    const [sender, setSender] = useState("")
    const [keypairPassword, setKeypairPassword] = useState("")
    const [contractId, setContractId] = useState("")
    const [isAuthorized, setIsAuthorized] = useState();

    const login = ({ url, sender, password, contractId }) => {
        setUrl(url); setSender(sender);
        setKeypairPassword(password); setContractId(contractId);
        setIsAuthorized(true);
    };

    const post = async (params) => {
        try {
            const res = await fetch(`http://localhost:${url}/transactions/signAndBroadcast`, {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({
                    contractId, fee: 10, sender, password: keypairPassword,
                    type: 104, params, version: 2, contractVersion: 1,
                }),
            });
            const result = await res.json();
            console.log(result);
            alert("успешно");
        } catch {
            alert("не получилось выполнить транзакцию")
        }
    };

    // ...
    return <AtlantContext.Provider value={{ url, sender, keypairPassword, isAuthorized, contractId, login, logout, post }}>
        {children}
    </AtlantContext.Provider>
}
```
Разберём ключевые решения:
- **Никакого Redux/Zustand** — состояние подключения (адрес ноды, отправитель, пароль пары ключей, id контракта) хранится в обычном React Context. Для учебного/хакатонного прототипа этого достаточно: данные нужны почти всем страницам, а сложной логики обновления состояния нет.
- **Один универсальный `post()`** на все методы контракта. Он уже знает `contractId` и `sender` из контекста — компонентам форм остаётся только передать нужные `params`. Это устраняет дублирование: не нужно писать fetch-запрос в каждой форме заново.
- **`type: 104`, `version: 2`, `contractVersion: 1`** — зашиты как константы, потому что в рамках одного контракта они не меняются между вызовами.
- **`fee: 10`** — фиксированная комиссия. На своей приватной песочнице этого достаточно, но для продакшена лучше брать актуальную минимальную комиссию через `GET /node/config` (поле `minimumFee[104]`), а не хардкодить.

Слабое место такого подхода: **пароль от пары ключей (`keypairPassword`) хранится в состоянии React-приложения в браузере** — то есть буквально во фронтенд-коде, доступном через DevTools. Это нормально для локальной демонстрации на закрытой песочнице, но не для реального продакшена — там подпись должна происходить либо на защищённом бэкенде, либо в отдельном кошельке пользователя, а не передаваться на фронт в открытом виде.

## Authorization.jsx — "подключение", а не логин
```jsx
const Authorization = () => {
    const { login } = useContext(AtlantContext);
    const [form, setForm] = useState({ url: "", sender: "", password: "", contractId: "" });

    const handleSubmit = (e) => {
        e.preventDefault();
        login(form);
    };

    return (
        <Form onSubmit={handleSubmit}>
            <FormControl placeholder="url" onChange={(e) => setForm({...form, url: e.target.value})} />
            <FormControl placeholder="sender" onChange={(e) => setForm({...form, sender: e.target.value})} />
            <FormControl placeholder="password" onChange={(e) => setForm({...form, password: e.target.value})} />
            <FormControl placeholder="contractId" onChange={(e) => setForm({...form, contractId: e.target.value})} />
            <Button type="submit">Войти</Button>
        </Form>
    );
}
```
Важно понимать: это **не логин в привычном смысле** (нет сервера авторизации, нет проверки пароля) — это просто форма, которая сохраняет в контекст четыре значения, нужных для дальнейших запросов к ноде: порт ноды, адрес отправителя, пароль его пары ключей в keystore этой ноды и id контракта. Реальная "авторизация" (имеет ли этот `sender` право что-то делать) происходит уже внутри контракта — в проверках `user.role` (см. файл про пользователей и роли).

## Чтение состояния — простой fetch без авторизации
```jsx
// UserInfo.jsx — личный кабинет
useEffect(() => {
    if (!sender || !contractId || !url) return;
    const load = async () => {
        const res = await fetch(`http://localhost:${url}/contracts/${contractId}`);
        const json = await res.json();
        setData(json);
    };
    load();
}, [contractId, sender, url]);

// дальше просто фильтруем весь стейт на клиенте:
data.filter(i => i.value.includes(sender)).map(...)
```
Чтение состояния контракта (`GET /contracts/{contractId}`) — публичный метод, не требует `X-API-Key`, поэтому здесь обычный `fetch` без заголовков авторизации. Фильтрация "покажи только записи, где встречается мой адрес" делается **на клиенте**, после того как весь стейт уже скачан — для песочницы с небольшим объёмом данных это ок, но на реальных объёмах правильнее использовать серверную фильтрацию через `POST /contracts/{contractId}` с параметром `matches` (регулярное выражение по ключам), чтобы не гонять по сети весь стейт целиком.

`UserCard.jsx` показывает более гибкий вариант того же паттерна — фильтрация сразу по двум полям (ключ и значение), заданным пользователем через два `FormControl`:
```jsx
data.filter(i => i.key.includes(role) && i.value.includes(wallet))
```

## func/ — форма на каждый метод контракта
Общий шаблон одинаков для всех файлов в `func/`:
```jsx
const RegisterOrg = () => {
    const { post } = useContext(AtlantContext);
    const [orgWallet, setOrgWallet] = useState("");
    // ...остальные поля состояния формы, по одному на каждый аргумент метода

    const handleSubmit = async (e) => {
        e.preventDefault();
        await post([
            { key: "action", type: "string", value: "registerOrg" },   // имя Java-метода
            { key: "orgWallet", type: "string", value: orgWallet },     // дальше — его аргументы
            { key: "name", type: "string", value: name },
            // ...
        ]);
    };

    return (
        <Form onSubmit={handleSubmit}>
            <FormControl placeholder="Адрес организации" onChange={(e) => setOrgWallet(e.target.value)} />
            {/* ...остальные поля формы */}
            <Button type="submit">Отправить</Button>
        </Form>
    );
};
```
Ключевой момент — первый элемент массива `params`, который передаётся в `post()`, это **всегда** `{ key: "action", value: "<имя метода>" }` (см. файл про написание контракта — так диспетчер контракта понимает, какой именно `@ContractAction`-метод вызывать). Дальше — просто по одному полю формы на каждый аргумент метода, с тем же именем `key`, что и имя параметра в Java-сигнатуре.

Эта прямолинейность — специально: чтобы добавить в интерфейс новый метод контракта, не нужно ничего "изобретать" на фронтенде — копируется один из существующих файлов `func/*.jsx`, меняется список полей формы под новую сигнатуру метода и имя в `action`.

## Function.jsx — страница со всеми формами сразу
```jsx
<Tabs>
    <Tab title="Процесс 1: учетные записи">
        <RegisterOrg /><UpdateAccount /><SetActive />
    </Tab>
    <Tab title="Процесс 2: продукция">
        <AddProduct /><ApproveProduct />
    </Tab>
    {/* ...остальные процессы */}
</Tabs>
```
Формы сгруппированы вкладками по тем же пяти бизнес-процессам, что описаны в комментариях самого Java-контракта (`Contract.java`) — учётные записи → продукция → заявки → исполнение поставщиком → приёмка/оплата. Такая группировка — хороший приём и для презентации на защите кейса: она сразу показывает, что фронтенд не набор разрозненных форм, а прямое отражение бизнес-процесса, который реализует контракт.

## Итоговая цепочка запроса
1. Пользователь заполняет форму в `func/RegisterOrg.jsx` и жмёт "Отправить".
2. Форма вызывает общий `post()` из контекста с массивом `params` (первый элемент — `action`).
3. `post()` собирает полную JSON-транзакцию 104 (добавляя `contractId`, `sender`, `password`, `fee`, `version`) и отправляет её на `http://localhost:<port>/transactions/signAndBroadcast`.
4. Нода подписывает транзакцию ключом `sender` из своего keystore и отправляет её в сеть.
5. Контракт исполняет метод `registerOrg`, меняет своё состояние.
6. Любая страница чтения (`UserInfo`, `BlockchainState`, `UserCard`) в следующий раз, когда сделает `GET /contracts/{contractId}`, увидит уже обновлённые данные.

Это тот же цикл "деплой → вызов → проверка", что описан в предыдущих файлах про API — просто обёрнутый в формы вместо ручных curl-команд.
