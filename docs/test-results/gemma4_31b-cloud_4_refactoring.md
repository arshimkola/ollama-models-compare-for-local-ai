Here is the refactored code. I have broken the logic into smaller, single-responsibility functions to improve readability and testability.

### The Refactored Code

```javascript
/**
 * Constants for business logic to avoid "magic numbers"
 */
const AGE_CATEGORIES = [
  { limit: 25, label: 'young' },
  { limit: 40, label: 'mid' },
  { limit: 60, label: 'senior' },
  { limit: Infinity, label: 'elder' },
];

const SCORE_THRESHOLDS = [
  { limit: 1000, points: 3 },
  { limit: 500, points: 2 },
  { limit: 100, points: 1 },
  { limit: 0, points: 0 },
];

const getAgeCategory = (age) => 
  AGE_CATEGORIES.find(cat => age < cat.limit).label;

const getPurchaseScore = (total) => 
  SCORE_THRESHOLDS.find(thresh => total > thresh.limit).points;

const calculateTotalSpent = (purchases = []) => 
  purchases.reduce((sum, item) => sum + item.amount, 0);

const transformUser = (user) => {
  const totalSpent = calculateTotalSpent(user.purchases);
  let score = getPurchaseScore(totalSpent);

  if (user.referrals > 5) {
    score += 2;
  }

  return {
    fullName: `${user.firstName} ${user.lastName}`,
    email: user.email?.toLowerCase().trim() ?? "N/A",
    age: user.age,
    category: getAgeCategory(user.age),
    score,
    ...(totalSpent > 0 && { totalSpent }), // Only add totalSpent if they actually bought something
  };
};

/**
 * Processes raw data to filter active adults and rank them by score.
 */
export function processData(data) {
  if (!Array.isArray(data)) return [];

  return data
    .filter(item => 
      item.type === "user" && 
      item.age >= 18 && 
      item.status === "active"
    )
    .map(transformUser)
    .sort((a, b) => b.score - a.score);
}
```

---

### Key Decisions & Improvements

#### 1. Eliminated "Arrow Code" (Deep Nesting)
The original code had four levels of nested `if` statements. I replaced this with a `.filter()` chain. This creates a "linear" flow: **Filter $\rightarrow$ Transform $\rightarrow$ Sort**. It is much easier to scan and understand.

#### 2. Declarative vs. Imperative Logic
*   **Arrays instead of `for` loops:** I replaced manual `for` loops with `.filter()`, `.map()`, and `.reduce()`. This removes the need to manage counters (`i`, `j`) and temporary arrays (`results = []`).
*   **Lookup Tables:** Instead of complex nested ternary operators for age and scores, I used configuration arrays (`AGE_CATEGORIES`). This makes it trivial to change the business rules (e.g., changing "senior" to start at 55) without touching the logic.

#### 3. Modern JavaScript Features
*   **Template Literals:** Used `` `${}` `` instead of `+` for string concatenation.
*   **Optional Chaining (`?.`):** Replaced the manual check for `data[i].email` with `user.email?.toLowerCase()`, which safely handles `null` or `undefined`.
*   **Nullish Coalescing (`??`):** Used to provide the "N/A" fallback.
*   **Destructuring & Arrow Functions:** Used `const` and arrow functions to ensure block-scoping and more concise syntax.

#### 4. Single Responsibility Principle (SRP)
The original function did everything: filtering, calculating totals, determining scores, formatting strings, and sorting. I split these into helper functions:
*   `calculateTotalSpent`: Only cares about summing numbers.
*   `transformUser`: Only cares about mapping a raw object to a clean "User" object.
*   `processData`: Orchestrates the high-level flow.

#### 5. Time and Space Complexity
*   **Time Complexity:** Remains $O(N \log N)$ due to the sort at the end, which is optimal for this requirement.
*   **Space Complexity:** $O(N)$ to store the results.

### Summary of Changes
| Old Approach | New Approach | Benefit |
| :--- | :--- | :--- |
| `var` | `const` / `let` | Prevents variable hoisting bugs |
| Nested `if` | `.filter()` | Improved readability (Reduced cognitive load) |
| Ternary chains | `.find()` on config array | Easier to maintain/update rules |
| `for` loop | `.reduce()` | More concise, less error-prone |
| Manual string concat | Template Literals | Cleaner syntax |
