# 🚀 Frontend Development Starter Guide

## 📖 Для початківців: Покроковий план розробки

Цей гайд допоможе вам почати писати код для фронтенду, навіть якщо ви тільки трохи знаєте JavaScript.

## 📋 Зміст
1. [Підготовка середовища](#підготовка-середовища)
2. [Порядок розробки компонентів](#порядок-розробки-компонентів)
3. [Покроковий план MVP](#покроковий-план-mvp)
4. [Приклади коду для початку](#приклади-коду-для-початку)
5. [Часті помилки початківців](#часті-помилки-початківців)

## 🛠 Підготовка середовища

### 1. Встановіть необхідне ПЗ:
```bash
# Перевірте версії
node --version  # має бути 18+
npm --version   # має бути 8+

# Встановіть залежності
cd frontend
npm install
```

### 2. Встановіть корисні VS Code розширення:
- **ES7+ React/Redux/React-Native snippets** - швидкі сніпети для React
- **Prettier** - автоматичне форматування коду
- **ESLint** - перевірка коду на помилки
- **Auto Rename Tag** - автоматичне перейменування HTML/JSX тегів
- **Bracket Pair Colorizer** - підсвітка дужок

### 3. Запустіть проект:
```bash
npm run dev
# Відкрийте http://localhost:3000
```

## 📝 Порядок розробки компонентів

### ✅ ПРАВИЛЬНИЙ ПОРЯДОК (від простого до складного):

#### Етап 1: Базова структура (1-2 дні)
1. **main.jsx** - точка входу
2. **App.jsx** - головний компонент
3. **styles/globals.css** - базові стилі

#### Етап 2: Прості статичні компоненти (2-3 дні)
1. **Button** - кнопка
2. **Input** - поле вводу
3. **Card** - картка
4. **LoadingSpinner** - індикатор завантаження

#### Етап 3: Layout компоненти (2-3 дні)
1. **Header** - шапка сайту
2. **Sidebar** - бокове меню
3. **Footer** - підвал
4. **Layout** - обгортка для сторінок

#### Етап 4: Форми та валідація (3-4 дні)
1. **LoginForm** - форма входу
2. **RegisterForm** - форма реєстрації
3. **useForm** - хук для роботи з формами
4. **validators.js** - функції валідації

#### Етап 5: API та стан (3-4 дні)
1. **api.js** - налаштування axios
2. **authStore.js** - стан авторизації
3. **useAuth.js** - хук авторизації
4. **PrivateRoute** - захищені маршрути

#### Етап 6: Основні сторінки (1 тиждень)
1. **Login** - сторінка входу
2. **Dashboard** - головна панель
3. **Transactions** - список транзакцій
4. **AddTransaction** - додавання транзакції

#### Етап 7: Візуалізація даних (3-4 дні)
1. **StatCard** - картка статистики
2. **LineChart** - лінійний графік
3. **PieChart** - кругова діаграма
4. **TransactionList** - список транзакцій

## 🎯 Покроковий план MVP

### Тиждень 1: Основа
```javascript
// 1. Почніть з main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './styles/globals.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

// 2. Створіть простий App.jsx
import React from 'react';

function App() {
  return (
    <div className="app">
      <h1>Finance Dashboard</h1>
      <p>Welcome to your financial tracker!</p>
    </div>
  );
}

export default App;

// 3. Додайте базові стилі в globals.css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: 'Inter', sans-serif;
  background: #f5f5f5;
}

.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}
```

### Тиждень 2: Компоненти
```javascript
// 4. Створіть Button компонент
// components/common/Button/Button.jsx
import React from 'react';
import styles from './Button.module.css';

const Button = ({ children, onClick, variant = 'primary', disabled = false }) => {
  return (
    <button
      className={`${styles.button} ${styles[variant]}`}
      onClick={onClick}
      disabled={disabled}
    >
      {children}
    </button>
  );
};

export default Button;

// 5. Стилі для Button
// Button.module.css
.button {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.3s;
}

.primary {
  background: #4F46E5;
  color: white;
}

.primary:hover {
  background: #4338CA;
}

.secondary {
  background: #E5E7EB;
  color: #374151;
}
```

### Тиждень 3: Роутинг та сторінки
```javascript
// 6. Додайте роутинг в App.jsx
import React from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import Login from './pages/Login/Login';
import Dashboard from './pages/Dashboard/Dashboard';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/login" element={<Login />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/" element={<Navigate to="/login" />} />
      </Routes>
    </BrowserRouter>
  );
}

// 7. Створіть просту Login сторінку
// pages/Login/Login.jsx
import React, { useState } from 'react';
import Button from '../../components/common/Button';
import './Login.css';

const Login = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    console.log('Login:', { email, password });
    // TODO: Додати API виклик
  };

  return (
    <div className="login-container">
      <form className="login-form" onSubmit={handleSubmit}>
        <h2>Вхід</h2>

        <div className="form-group">
          <label>Email:</label>
          <input
            type="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </div>

        <div className="form-group">
          <label>Пароль:</label>
          <input
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </div>

        <Button type="submit">Увійти</Button>
      </form>
    </div>
  );
};

export default Login;
```

### Тиждень 4: API інтеграція
```javascript
// 8. Налаштуйте API
// services/api.js
import axios from 'axios';

const API = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8080/api',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Додати токен до кожного запиту
API.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Обробка помилок
API.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default API;

// 9. Створіть сервіс авторизації
// services/authService.js
import API from './api';

export const authService = {
  async login(email, password) {
    const response = await API.post('/auth/login', { email, password });
    const { token, user } = response.data;
    localStorage.setItem('token', token);
    return user;
  },

  async logout() {
    localStorage.removeItem('token');
  },

  async register(data) {
    const response = await API.post('/auth/register', data);
    return response.data;
  },

  isAuthenticated() {
    return !!localStorage.getItem('token');
  }
};
```

### Тиждень 5: State Management з Zustand
```javascript
// 10. Створіть store
// store/authStore.js
import { create } from 'zustand';
import { authService } from '../services/authService';

const useAuthStore = create((set) => ({
  user: null,
  isLoading: false,
  error: null,

  login: async (email, password) => {
    set({ isLoading: true, error: null });
    try {
      const user = await authService.login(email, password);
      set({ user, isLoading: false });
      return true;
    } catch (error) {
      set({ error: error.message, isLoading: false });
      return false;
    }
  },

  logout: () => {
    authService.logout();
    set({ user: null });
  },

  checkAuth: () => {
    const isAuth = authService.isAuthenticated();
    if (!isAuth) {
      set({ user: null });
    }
  }
}));

export default useAuthStore;

// 11. Використайте store в компоненті
// Оновіть Login.jsx
import useAuthStore from '../../store/authStore';
import { useNavigate } from 'react-router-dom';

const Login = () => {
  const navigate = useNavigate();
  const { login, isLoading, error } = useAuthStore();

  const handleSubmit = async (e) => {
    e.preventDefault();
    const success = await login(email, password);
    if (success) {
      navigate('/dashboard');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      {error && <div className="error">{error}</div>}
      {/* ... форма ... */}
      <Button type="submit" disabled={isLoading}>
        {isLoading ? 'Вхід...' : 'Увійти'}
      </Button>
    </form>
  );
};
```

### Тиждень 6: Dashboard та транзакції
```javascript
// 12. Створіть Dashboard
// pages/Dashboard/Dashboard.jsx
import React, { useEffect, useState } from 'react';
import useAuthStore from '../../store/authStore';
import StatCard from '../../components/StatCard';
import TransactionList from '../../components/TransactionList';
import { transactionService } from '../../services/transactionService';
import './Dashboard.css';

const Dashboard = () => {
  const { user } = useAuthStore();
  const [stats, setStats] = useState(null);
  const [transactions, setTransactions] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchData();
  }, []);

  const fetchData = async () => {
    try {
      const [statsData, transData] = await Promise.all([
        transactionService.getStatistics(),
        transactionService.getRecent()
      ]);
      setStats(statsData);
      setTransactions(transData);
    } catch (error) {
      console.error('Error fetching data:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) return <div>Завантаження...</div>;

  return (
    <div className="dashboard">
      <h1>Вітаємо, {user?.name}!</h1>

      <div className="stats-grid">
        <StatCard
          title="Доходи"
          value={stats?.income || 0}
          color="green"
        />
        <StatCard
          title="Витрати"
          value={stats?.expenses || 0}
          color="red"
        />
        <StatCard
          title="Баланс"
          value={stats?.balance || 0}
          color="blue"
        />
      </div>

      <div className="recent-transactions">
        <h2>Останні транзакції</h2>
        <TransactionList transactions={transactions} />
      </div>
    </div>
  );
};

export default Dashboard;
```

## 💡 Приклади коду для початку

### Простий custom hook
```javascript
// hooks/useLocalStorage.js
import { useState } from 'react';

export const useLocalStorage = (key, initialValue) => {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  const setValue = (value) => {
    try {
      setStoredValue(value);
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error(error);
    }
  };

  return [storedValue, setValue];
};
```

### Проста валідація форми
```javascript
// utils/validators.js
export const validators = {
  email: (value) => {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!value) return 'Email є обов\'язковим';
    if (!emailRegex.test(value)) return 'Невірний формат email';
    return null;
  },

  password: (value) => {
    if (!value) return 'Пароль є обов\'язковим';
    if (value.length < 6) return 'Пароль має бути мінімум 6 символів';
    return null;
  },

  amount: (value) => {
    if (!value) return 'Сума є обов\'язковою';
    if (isNaN(value)) return 'Сума має бути числом';
    if (value <= 0) return 'Сума має бути більше 0';
    return null;
  }
};

// Використання
const emailError = validators.email(email);
if (emailError) {
  setError(emailError);
}
```

### Форматування даних
```javascript
// utils/formatters.js
export const formatters = {
  currency: (amount) => {
    return new Intl.NumberFormat('uk-UA', {
      style: 'currency',
      currency: 'UAH'
    }).format(amount);
  },

  date: (date) => {
    return new Date(date).toLocaleDateString('uk-UA');
  },

  percentage: (value) => {
    return `${(value * 100).toFixed(2)}%`;
  }
};

// Використання
<span>{formatters.currency(1500)}</span> // ₴1,500.00
```

## ⚠️ Часті помилки початківців

### ❌ НЕПРАВИЛЬНО:
```javascript
// 1. Прямі зміни стану
const [user, setUser] = useState({ name: 'John', age: 25 });
user.age = 26; // ❌ Не змінюйте стан напряму!
setUser(user); // ❌ React не побачить зміни

// 2. Неправильне використання useEffect
useEffect(() => {
  fetchData(); // ❌ Без залежностей - буде викликатись на кожен рендер
});

// 3. Забуття про key в списках
items.map(item => <div>{item.name}</div>) // ❌ Немає key

// 4. Використання index як key
items.map((item, index) => <div key={index}>{item.name}</div>) // ❌ Погана практика
```

### ✅ ПРАВИЛЬНО:
```javascript
// 1. Правильна зміна стану
setUser({ ...user, age: 26 }); // ✅ Створюємо новий об'єкт

// 2. Правильне використання useEffect
useEffect(() => {
  fetchData();
}, []); // ✅ Порожній масив = виконається тільки раз

// 3. Правильне використання key
items.map(item => <div key={item.id}>{item.name}</div>) // ✅ Унікальний key

// 4. Обробка помилок
try {
  const data = await fetchData();
  setData(data);
} catch (error) {
  setError(error.message); // ✅ Завжди обробляйте помилки
}
```

## 🎓 Корисні поради

### 1. Почніть з простого
- Спочатку зробіть щоб працювало, потім покращуйте
- Не намагайтесь відразу зробити все ідеально

### 2. Використовуйте готові рішення
- Material-UI для швидкого UI
- React Hook Form для форм
- SWR або React Query для API

### 3. Тестуйте на ходу
- Використовуйте console.log() для дебагу
- React DevTools для перегляду стану
- Network tab для перевірки API запитів

### 4. Структура компонента
```javascript
// Рекомендована структура
const MyComponent = () => {
  // 1. Стан
  const [data, setData] = useState(null);

  // 2. Хуки
  const { user } = useAuth();

  // 3. Ефекти
  useEffect(() => {
    // ...
  }, []);

  // 4. Handlers
  const handleClick = () => {
    // ...
  };

  // 5. Render helpers
  const renderList = () => {
    // ...
  };

  // 6. Main render
  return <div>...</div>;
};
```

## 📚 Наступні кроки

Після завершення MVP:
1. Додайте більше функціональності (категорії, фільтри)
2. Покращіть UI/UX (анімації, transitions)
3. Додайте тести
4. Оптимізуйте продуктивність
5. Додайте PWA функціональність

## 🔗 Корисні ресурси для навчання

### Безкоштовні курси:
- [React Official Tutorial](https://react.dev/learn)
- [FreeCodeCamp React Course](https://www.freecodecamp.org/learn/front-end-development-libraries/react/)
- [Scrimba React Course](https://scrimba.com/learn/learnreact)

### YouTube канали:
- Web Dev Simplified
- Traversy Media
- The Net Ninja

### Документація:
- [React Docs](https://react.dev/)
- [MDN Web Docs](https://developer.mozilla.org/)
- [JavaScript.info](https://javascript.info/)

Успіхів у розробці! 🚀