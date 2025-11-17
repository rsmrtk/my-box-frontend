# 🎨 Frontend Development Guide

## 📚 Зміст
1. [Структура проекту](#структура-проекту)
2. [React Patterns](#react-patterns)
3. [Компоненти](#компоненти)
4. [State Management](#state-management)
5. [API Integration](#api-integration)
6. [Styling](#styling)
7. [Testing](#testing)
8. [Best Practices](#best-practices)

## 📁 Структура проекту

```
frontend/
├── src/
│   ├── components/           # Переіспользуємі компоненти
│   │   ├── common/          # Базові компоненти
│   │   │   ├── Button/
│   │   │   │   ├── Button.jsx
│   │   │   │   ├── Button.module.css
│   │   │   │   ├── Button.test.js
│   │   │   │   └── index.js
│   │   │   ├── Input/
│   │   │   ├── Card/
│   │   │   └── Modal/
│   │   │
│   │   ├── charts/          # Графіки та візуалізація
│   │   │   ├── LineChart/
│   │   │   ├── PieChart/
│   │   │   └── BarChart/
│   │   │
│   │   └── layout/          # Layout компоненти
│   │       ├── Header/
│   │       ├── Sidebar/
│   │       └── Footer/
│   │
│   ├── pages/               # Сторінки додатку
│   │   ├── Dashboard/
│   │   │   ├── Dashboard.jsx
│   │   │   ├── Dashboard.module.css
│   │   │   └── components/    # Компоненти специфічні для сторінки
│   │   ├── Transactions/
│   │   ├── Reports/
│   │   └── Settings/
│   │
│   ├── features/            # Feature-based modules
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── services/
│   │   │   └── store/
│   │   │
│   │   └── transactions/
│   │       ├── components/
│   │       ├── hooks/
│   │       └── services/
│   │
│   ├── hooks/               # Custom React hooks
│   ├── services/            # API services
│   ├── store/               # Global state
│   ├── utils/               # Утиліти
│   ├── styles/              # Глобальні стилі
│   └── constants/           # Константи
```

## 🎯 React Patterns

### 1. Функціональні компоненти з Hooks

```jsx
// ✅ ДОБРЕ: Використовуйте функціональні компоненти
import React, { useState, useEffect } from 'react';
import styles from './TransactionCard.module.css';

const TransactionCard = ({ transaction, onUpdate }) => {
  const [isEditing, setIsEditing] = useState(false);
  const [amount, setAmount] = useState(transaction.amount);

  useEffect(() => {
    // Side effects тут
  }, [transaction.id]);

  return (
    <div className={styles.card}>
      <h3>{transaction.title}</h3>
      <p className={styles.amount}>${amount}</p>
      {/* ... */}
    </div>
  );
};

export default TransactionCard;
```

### 2. Custom Hooks для бізнес-логіки

```jsx
// hooks/useTransactions.js
import { useState, useEffect } from 'react';
import { transactionService } from '../services/transactionService';

export const useTransactions = (userId) => {
  const [transactions, setTransactions] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchTransactions = async () => {
      try {
        setLoading(true);
        const data = await transactionService.getAll(userId);
        setTransactions(data);
      } catch (err) {
        setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchTransactions();
  }, [userId]);

  const addTransaction = async (transaction) => {
    try {
      const newTransaction = await transactionService.create(transaction);
      setTransactions([...transactions, newTransaction]);
      return newTransaction;
    } catch (err) {
      setError(err.message);
      throw err;
    }
  };

  return {
    transactions,
    loading,
    error,
    addTransaction,
    refetch: () => fetchTransactions()
  };
};
```

### 3. Container/Presentational Pattern

```jsx
// containers/DashboardContainer.jsx (Smart Component)
import React from 'react';
import { useTransactions } from '../../hooks/useTransactions';
import { useAuth } from '../../hooks/useAuth';
import DashboardView from './DashboardView';

const DashboardContainer = () => {
  const { user } = useAuth();
  const { transactions, loading, error } = useTransactions(user.id);

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;

  return <DashboardView transactions={transactions} user={user} />;
};

// components/DashboardView.jsx (Dumb Component)
const DashboardView = ({ transactions, user }) => {
  return (
    <div className={styles.dashboard}>
      <h1>Welcome, {user.name}!</h1>
      <TransactionList transactions={transactions} />
    </div>
  );
};
```

### 4. Compound Components Pattern

```jsx
// components/Card/index.jsx
const Card = ({ children, className }) => (
  <div className={`${styles.card} ${className}`}>{children}</div>
);

Card.Header = ({ children }) => (
  <div className={styles.header}>{children}</div>
);

Card.Body = ({ children }) => (
  <div className={styles.body}>{children}</div>
);

Card.Footer = ({ children }) => (
  <div className={styles.footer}>{children}</div>
);

// Використання:
<Card>
  <Card.Header>Transaction Details</Card.Header>
  <Card.Body>
    <p>Amount: $100</p>
  </Card.Body>
  <Card.Footer>
    <Button>Edit</Button>
  </Card.Footer>
</Card>
```

## 🧩 Компоненти

### Правила створення компонентів

1. **Single Responsibility**: Один компонент - одна відповідальність
2. **DRY**: Не повторюйте код, винесіть в окремий компонент
3. **Composition over Inheritance**: Використовуйте композицію

### Приклад компонента

```jsx
// components/common/Button/Button.jsx
import React from 'react';
import PropTypes from 'prop-types';
import styles from './Button.module.css';

const Button = ({
  children,
  variant = 'primary',
  size = 'medium',
  disabled = false,
  onClick,
  type = 'button',
  className = '',
  ...rest
}) => {
  const buttonClasses = [
    styles.button,
    styles[variant],
    styles[size],
    disabled && styles.disabled,
    className
  ].filter(Boolean).join(' ');

  return (
    <button
      className={buttonClasses}
      disabled={disabled}
      onClick={onClick}
      type={type}
      {...rest}
    >
      {children}
    </button>
  );
};

Button.propTypes = {
  children: PropTypes.node.isRequired,
  variant: PropTypes.oneOf(['primary', 'secondary', 'danger']),
  size: PropTypes.oneOf(['small', 'medium', 'large']),
  disabled: PropTypes.bool,
  onClick: PropTypes.func,
  type: PropTypes.oneOf(['button', 'submit', 'reset']),
  className: PropTypes.string
};

export default Button;
```

## 📊 State Management

### Context API для глобального стану

```jsx
// store/AuthContext.jsx
import React, { createContext, useContext, useReducer } from 'react';

const AuthContext = createContext();

const authReducer = (state, action) => {
  switch (action.type) {
    case 'LOGIN':
      return { ...state, user: action.payload, isAuthenticated: true };
    case 'LOGOUT':
      return { user: null, isAuthenticated: false };
    default:
      return state;
  }
};

export const AuthProvider = ({ children }) => {
  const [state, dispatch] = useReducer(authReducer, {
    user: null,
    isAuthenticated: false
  });

  const login = async (credentials) => {
    const user = await authService.login(credentials);
    dispatch({ type: 'LOGIN', payload: user });
  };

  const logout = () => {
    authService.logout();
    dispatch({ type: 'LOGOUT' });
  };

  return (
    <AuthContext.Provider value={{ ...state, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

### Zustand для складнішого стану

```jsx
// store/transactionStore.js
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

const useTransactionStore = create(
  devtools(
    persist(
      (set, get) => ({
        transactions: [],
        filters: {
          category: null,
          dateRange: null
        },

        // Actions
        setTransactions: (transactions) =>
          set({ transactions }),

        addTransaction: (transaction) =>
          set((state) => ({
            transactions: [...state.transactions, transaction]
          })),

        updateTransaction: (id, updates) =>
          set((state) => ({
            transactions: state.transactions.map(t =>
              t.id === id ? { ...t, ...updates } : t
            )
          })),

        deleteTransaction: (id) =>
          set((state) => ({
            transactions: state.transactions.filter(t => t.id !== id)
          })),

        setFilter: (filterType, value) =>
          set((state) => ({
            filters: { ...state.filters, [filterType]: value }
          })),

        // Computed values
        getFilteredTransactions: () => {
          const { transactions, filters } = get();
          return transactions.filter(t => {
            if (filters.category && t.category !== filters.category) {
              return false;
            }
            // More filter logic
            return true;
          });
        }
      }),
      {
        name: 'transaction-storage'
      }
    )
  )
);

export default useTransactionStore;
```

## 🔌 API Integration

### Service Layer Pattern

```jsx
// services/api.js
const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8080/api';

class ApiService {
  constructor() {
    this.baseURL = API_BASE_URL;
  }

  async request(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const config = {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...this.getAuthHeader(),
        ...options.headers,
      },
    };

    try {
      const response = await fetch(url, config);

      if (!response.ok) {
        throw new Error(`API Error: ${response.status}`);
      }

      return await response.json();
    } catch (error) {
      console.error('API Request failed:', error);
      throw error;
    }
  }

  getAuthHeader() {
    const token = localStorage.getItem('authToken');
    return token ? { Authorization: `Bearer ${token}` } : {};
  }

  get(endpoint) {
    return this.request(endpoint, { method: 'GET' });
  }

  post(endpoint, data) {
    return this.request(endpoint, {
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  put(endpoint, data) {
    return this.request(endpoint, {
      method: 'PUT',
      body: JSON.stringify(data),
    });
  }

  delete(endpoint) {
    return this.request(endpoint, { method: 'DELETE' });
  }
}

export default new ApiService();
```

```jsx
// services/transactionService.js
import api from './api';

class TransactionService {
  async getAll(userId) {
    return api.get(`/users/${userId}/transactions`);
  }

  async getById(id) {
    return api.get(`/transactions/${id}`);
  }

  async create(transaction) {
    return api.post('/transactions', transaction);
  }

  async update(id, updates) {
    return api.put(`/transactions/${id}`, updates);
  }

  async delete(id) {
    return api.delete(`/transactions/${id}`);
  }

  async getStatistics(userId, period) {
    return api.get(`/users/${userId}/statistics?period=${period}`);
  }
}

export default new TransactionService();
```

## 🎨 Styling

### CSS Modules

```css
/* Button.module.css */
.button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s ease;
}

.primary {
  background-color: var(--color-primary);
  color: white;
}

.primary:hover {
  background-color: var(--color-primary-dark);
}

.secondary {
  background-color: transparent;
  color: var(--color-primary);
  border: 1px solid var(--color-primary);
}

.small {
  padding: 4px 8px;
  font-size: 12px;
}

.large {
  padding: 12px 24px;
  font-size: 16px;
}

.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

### Global Variables

```css
/* styles/variables.css */
:root {
  /* Colors */
  --color-primary: #4F46E5;
  --color-primary-dark: #4338CA;
  --color-secondary: #06B6D4;
  --color-success: #10B981;
  --color-warning: #F59E0B;
  --color-danger: #EF4444;

  /* Typography */
  --font-family: 'Inter', sans-serif;
  --font-size-xs: 12px;
  --font-size-sm: 14px;
  --font-size-base: 16px;
  --font-size-lg: 18px;
  --font-size-xl: 20px;

  /* Spacing */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;

  /* Shadows */
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);
}
```

## 🧪 Testing

### Component Testing

```jsx
// Button.test.js
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import '@testing-library/jest-dom';
import Button from './Button';

describe('Button Component', () => {
  test('renders button with text', () => {
    render(<Button>Click me</Button>);
    const button = screen.getByText('Click me');
    expect(button).toBeInTheDocument();
  });

  test('calls onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    fireEvent.click(screen.getByText('Click me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  test('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);
    const button = screen.getByText('Click me');
    expect(button).toBeDisabled();
  });

  test('applies correct variant class', () => {
    const { rerender } = render(<Button variant="primary">Click me</Button>);
    expect(screen.getByText('Click me')).toHaveClass('primary');

    rerender(<Button variant="secondary">Click me</Button>);
    expect(screen.getByText('Click me')).toHaveClass('secondary');
  });
});
```

### Hook Testing

```jsx
// useTransactions.test.js
import { renderHook, waitFor } from '@testing-library/react';
import { useTransactions } from './useTransactions';
import * as transactionService from '../services/transactionService';

jest.mock('../services/transactionService');

describe('useTransactions Hook', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('fetches transactions on mount', async () => {
    const mockTransactions = [
      { id: 1, amount: 100 },
      { id: 2, amount: 200 }
    ];

    transactionService.getAll.mockResolvedValue(mockTransactions);

    const { result } = renderHook(() => useTransactions('user123'));

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.transactions).toEqual(mockTransactions);
    expect(result.current.error).toBeNull();
  });

  test('handles error when fetching fails', async () => {
    const errorMessage = 'Network error';
    transactionService.getAll.mockRejectedValue(new Error(errorMessage));

    const { result } = renderHook(() => useTransactions('user123'));

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });

    expect(result.current.transactions).toEqual([]);
    expect(result.current.error).toBe(errorMessage);
  });
});
```

## ✅ Best Practices

### 1. Component Guidelines

```jsx
// ✅ ДОБРЕ: Чітко розділені concerns
const TransactionList = ({ transactions, onDelete }) => {
  return (
    <ul>
      {transactions.map(transaction => (
        <TransactionItem
          key={transaction.id}
          transaction={transaction}
          onDelete={onDelete}
        />
      ))}
    </ul>
  );
};

// ❌ ПОГАНО: Змішана логіка
const TransactionList = () => {
  const [transactions, setTransactions] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    // Fetching logic here...
  }, []);

  // Delete logic here...

  return (
    // UI logic here...
  );
};
```

### 2. Performance Optimization

```jsx
// Використовуйте React.memo для expensive components
const ExpensiveChart = React.memo(({ data }) => {
  return <ComplexChart data={data} />;
}, (prevProps, nextProps) => {
  // Custom comparison
  return prevProps.data.id === nextProps.data.id;
});

// Використовуйте useMemo для expensive calculations
const Dashboard = ({ transactions }) => {
  const statistics = useMemo(() => {
    return calculateStatistics(transactions);
  }, [transactions]);

  return <StatisticsDisplay stats={statistics} />;
};

// Використовуйте useCallback для stable references
const TransactionForm = ({ onSubmit }) => {
  const [formData, setFormData] = useState({});

  const handleSubmit = useCallback((e) => {
    e.preventDefault();
    onSubmit(formData);
  }, [formData, onSubmit]);

  return <form onSubmit={handleSubmit}>{/* ... */}</form>;
};
```

### 3. Error Boundaries

```jsx
// components/ErrorBoundary.jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error, errorInfo);
    // Send to error reporting service
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>Oops! Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}

// Використання
<ErrorBoundary>
  <Dashboard />
</ErrorBoundary>
```

### 4. Code Splitting

```jsx
// Lazy loading routes
import React, { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';

const Dashboard = lazy(() => import('./pages/Dashboard'));
const Transactions = lazy(() => import('./pages/Transactions'));
const Reports = lazy(() => import('./pages/Reports'));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/transactions" element={<Transactions />} />
          <Route path="/reports" element={<Reports />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### 5. Form Handling

```jsx
// hooks/useForm.js
export const useForm = (initialValues, validate) => {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});

  const handleChange = (e) => {
    const { name, value } = e.target;
    setValues({ ...values, [name]: value });

    // Clear error when user types
    if (errors[name]) {
      setErrors({ ...errors, [name]: '' });
    }
  };

  const handleBlur = (e) => {
    const { name } = e.target;
    setTouched({ ...touched, [name]: true });

    // Validate on blur
    if (validate) {
      const validationErrors = validate(values);
      setErrors({ ...errors, [name]: validationErrors[name] });
    }
  };

  const handleSubmit = (onSubmit) => (e) => {
    e.preventDefault();

    if (validate) {
      const validationErrors = validate(values);
      setErrors(validationErrors);

      if (Object.keys(validationErrors).length === 0) {
        onSubmit(values);
      }
    } else {
      onSubmit(values);
    }
  };

  const reset = () => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  };

  return {
    values,
    errors,
    touched,
    handleChange,
    handleBlur,
    handleSubmit,
    reset
  };
};
```

## 📝 Checklist для Code Review

- [ ] Компонент має одну відповідальність
- [ ] Props типізовані з PropTypes або TypeScript
- [ ] Немає прямих DOM маніпуляцій
- [ ] Side effects винесені в useEffect
- [ ] Використано правильні React hooks
- [ ] Немає memory leaks (cleanup в useEffect)
- [ ] Оптимізовано re-renders (memo, useMemo, useCallback)
- [ ] Error handling на місці
- [ ] Є unit tests
- [ ] Код читабельний та maintainable
- [ ] Стилі модульні та reusable
- [ ] Accessibility (ARIA attributes, semantic HTML)

## 🚀 Початок роботи

1. **Встановіть залежності:**
```bash
cd frontend
npm install
```

2. **Запустіть development server:**
```bash
npm run dev
```

3. **Запустіть тести:**
```bash
npm test
```

4. **Build для production:**
```bash
npm run build
```

Запити 

1. Користувач додає дохід:
   - Натискає "Додати дохід"
   - Заповнює форму
   - Frontend → POST /api/incomes
   - Backend створює і повертає новий income
2. Відображення після додавання:
   - Frontend отримує відповідь від POST
   - Або робить GET /api/incomes щоб оновити весь список
   - Показує оновлені дані користувачу
3. Користувач відкриває додаток:
   - Frontend → GET /api/incomes
   - Показує всі доходи в таблиці/списку
4. Користувач клікає на конкретний дохід:
   - Frontend → GET /api/incomes/{id}
   - Показує детальну інформацію

Коротко:
- POST = Створити новий дохід
- GET = Отримати/показати доходи
- PUT = Оновити існуючий дохід
- DELETE = Видалити дохід

## 📚 Корисні ресурси

- [React Documentation](https://react.dev/)
- [React Patterns](https://reactpatterns.com/)
- [Testing Library](https://testing-library.com/)
- [React Hook Form](https://react-hook-form.com/)
- [Zustand](https://github.com/pmndrs/zustand)
- [SWR for Data Fetching](https://swr.vercel.app/)