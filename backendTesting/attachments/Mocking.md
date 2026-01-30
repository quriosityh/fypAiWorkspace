---
link: https://jestjs.io/docs/mock-functions
---
# Mocking in Jest: Complete Guide

## What is Mocking?

Mocking is the practice of replacing real dependencies (functions, modules, APIs) with fake implementations during testing. This lets you:

- Test code in isolation
- Control external behavior (APIs, databases)
- Verify function calls without executing real logic
- Speed up tests by avoiding slow operations

---

## Core Mocking Concepts

### 1. **Mock Functions (`jest.fn()`)**

Create fake functions that track how they're called.

```javascript
// Basic mock function
const mockCallback = jest.fn();

// Call it
mockCallback('hello', 42);

// Verify it was called
expect(mockCallback).toHaveBeenCalled();
expect(mockCallback).toHaveBeenCalledWith('hello', 42);
expect(mockCallback).toHaveBeenCalledTimes(1);
```

**With Return Values:**

```javascript
// Mock with return value
const mockFn = jest.fn().mockReturnValue(42);
console.log(mockFn()); // 42

// Mock with different returns per call
const mockMultiple = jest.fn()
  .mockReturnValueOnce('first')
  .mockReturnValueOnce('second')
  .mockReturnValue('default');

console.log(mockMultiple()); // 'first'
console.log(mockMultiple()); // 'second'
console.log(mockMultiple()); // 'default'
console.log(mockMultiple()); // 'default'
```

**With Implementations:**

```javascript
// Mock with custom logic
const mockImplementation = jest.fn((x, y) => x + y);
expect(mockImplementation(2, 3)).toBe(5);

// Mock async functions
const mockAsync = jest.fn().mockResolvedValue('success');
await expect(mockAsync()).resolves.toBe('success');

const mockError = jest.fn().mockRejectedValue(new Error('fail'));
await expect(mockError()).rejects.toThrow('fail');
```

---

### 2. **Mocking Modules (`jest.mock()`)**

Replace entire modules with mocks.

**Example Module to Mock:**

```javascript
// utils/api.js
export const fetchUser = async (id) => {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
};

export const deleteUser = async (id) => {
  await fetch(`/api/users/${id}`, { method: 'DELETE' });
};
```

**Test with Module Mock:**

```javascript
// __tests__/userService.test.js
import { fetchUser, deleteUser } from '../utils/api';

// Mock the entire module
jest.mock('../utils/api');

describe('User Service', () => {
  beforeEach(() => {
    // Clear mock data between tests
    jest.clearAllMocks();
  });

  test('should fetch user data', async () => {
    // Define mock behavior
    fetchUser.mockResolvedValue({ id: 1, name: 'John' });

    const user = await fetchUser(1);
    
    expect(fetchUser).toHaveBeenCalledWith(1);
    expect(user).toEqual({ id: 1, name: 'John' });
  });

  test('should handle errors', async () => {
    fetchUser.mockRejectedValue(new Error('Network error'));

    await expect(fetchUser(1)).rejects.toThrow('Network error');
  });
});
```

---

### 3. **Partial Module Mocking**

Mock only specific exports, keep others real.

```javascript
// Mock only fetchUser, keep deleteUser real
jest.mock('../utils/api', () => ({
  ...jest.requireActual('../utils/api'), // Keep original exports
  fetchUser: jest.fn(), // Override this one
}));
```

---

### 4. **Mocking Classes**

```javascript
// services/Database.js
export class Database {
  connect() {
    // Real connection logic
  }
  
  query(sql) {
    // Real query logic
  }
}
```

**Mock the Class:**

```javascript
import { Database } from '../services/Database';

jest.mock('../services/Database');

test('uses database', () => {
  // Mock instance methods
  const mockQuery = jest.fn().mockResolvedValue([{ id: 1 }]);
  Database.mockImplementation(() => ({
    connect: jest.fn(),
    query: mockQuery,
  }));

  const db = new Database();
  db.query('SELECT * FROM users');

  expect(mockQuery).toHaveBeenCalledWith('SELECT * FROM users');
});
```

---

## Real-World Examples

### Example 1: Testing API Calls

```javascript
// userService.js
import axios from 'axios';

export async function getUserProfile(userId) {
  try {
    const response = await axios.get(`/api/users/${userId}`);
    return response.data;
  } catch (error) {
    throw new Error('Failed to fetch user');
  }
}
```

**Test:**

```javascript
// userService.test.js
import axios from 'axios';
import { getUserProfile } from './userService';

jest.mock('axios'); // Mock axios module

describe('getUserProfile', () => {
  test('fetches user data successfully', async () => {
    const mockUser = { id: 1, name: 'Alice' };
    axios.get.mockResolvedValue({ data: mockUser });

    const result = await getUserProfile(1);

    expect(axios.get).toHaveBeenCalledWith('/api/users/1');
    expect(result).toEqual(mockUser);
  });

  test('handles fetch errors', async () => {
    axios.get.mockRejectedValue(new Error('Network Error'));

    await expect(getUserProfile(1)).rejects.toThrow('Failed to fetch user');
  });
});
```

---

### Example 2: Mocking Timers

```javascript
// scheduler.js
export function scheduleTask(callback, delay) {
  setTimeout(callback, delay);
}
```

**Test with Fake Timers:**

```javascript
import { scheduleTask } from './scheduler';

jest.useFakeTimers(); // Use Jest's fake timers

test('executes callback after delay', () => {
  const callback = jest.fn();
  
  scheduleTask(callback, 1000);
  
  expect(callback).not.toHaveBeenCalled();
  
  // Fast-forward time
  jest.advanceTimersByTime(1000);
  
  expect(callback).toHaveBeenCalledTimes(1);
});

afterAll(() => {
  jest.useRealTimers(); // Restore real timers
});
```

---

### Example 3: Spying on Methods

```javascript
// calculator.js
export const calculator = {
  add: (a, b) => a + b,
  multiply: (a, b) => a * b,
};
```

**Test with Spies:**

```javascript
import { calculator } from './calculator';

test('tracks method calls', () => {
  // Spy on method without changing behavior
  const addSpy = jest.spyOn(calculator, 'add');
  
  const result = calculator.add(2, 3);
  
  expect(result).toBe(5); // Real implementation still works
  expect(addSpy).toHaveBeenCalledWith(2, 3);
  
  addSpy.mockRestore(); // Restore original implementation
});

test('mocks method implementation', () => {
  const multiplySpy = jest.spyOn(calculator, 'multiply')
    .mockReturnValue(100);
  
  expect(calculator.multiply(2, 3)).toBe(100); // Uses mock
  
  multiplySpy.mockRestore();
  expect(calculator.multiply(2, 3)).toBe(6); // Original restored
});
```

---

## Mock Assertions

```javascript
const mock = jest.fn();

// Check if called
expect(mock).toHaveBeenCalled();
expect(mock).not.toHaveBeenCalled();

// Check call count
expect(mock).toHaveBeenCalledTimes(3);

// Check arguments
expect(mock).toHaveBeenCalledWith('arg1', 'arg2');
expect(mock).toHaveBeenLastCalledWith('arg1', 'arg2');
expect(mock).toHaveBeenNthCalledWith(2, 'arg1', 'arg2'); // Second call

// Inspect calls
expect(mock.mock.calls).toEqual([
  ['first', 'call'],
  ['second', 'call'],
]);

expect(mock.mock.results[0].value).toBe('return value');
```

---

## Best Practices

### ✅ DO:

- Clear mocks between tests: `jest.clearAllMocks()` in `beforeEach()`
- Mock external dependencies (APIs, databases, file system)
- Use descriptive mock names: `mockFetchUser`, `mockDatabase`
- Test both success and failure scenarios
- Verify mocks were called correctly

### ❌ DON'T:

- Mock everything (test real logic when possible)
- Forget to restore spies: `spy.mockRestore()`
- Leave fake timers enabled: `jest.useRealTimers()`
- Over-specify mock implementations (keep tests flexible)

---

## Setup/Teardown Pattern

```javascript
describe('Test Suite', () => {
  beforeEach(() => {
    // Reset mocks before each test
    jest.clearAllMocks();
  });

  afterEach(() => {
    // Restore original implementations
    jest.restoreAllMocks();
  });

  afterAll(() => {
    // Clean up global mocks
    jest.useRealTimers();
  });

  test('example test', () => {
    // Your test here
  });
});
```

---

## Quick Reference

|**Task**|**Method**|
|---|---|
|Create mock function|`jest.fn()`|
|Mock module|`jest.mock('module')`|
|Spy on method|`jest.spyOn(object, 'method')`|
|Set return value|`.mockReturnValue(value)`|
|Set async return|`.mockResolvedValue(value)`|
|Set implementation|`.mockImplementation(fn)`|
|Use fake timers|`jest.useFakeTimers()`|
|Advance timers|`jest.advanceTimersByTime(ms)`|
|Clear mock data|`jest.clearAllMocks()`|
|Restore original|`jest.restoreAllMocks()`|

---

## Next Steps

Practice mocking with a real project! Try:

1. Mock an API call in your codebase
2. Write tests for async functions
3. Use fake timers for setTimeout/setInterval tests
4. Spy on methods to verify internal calls

Would you like me to help you apply mocking to a specific scenario from your project?