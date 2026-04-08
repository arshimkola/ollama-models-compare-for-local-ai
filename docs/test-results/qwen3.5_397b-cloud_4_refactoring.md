Here is the refactored code, followed by a detailed explanation of the changes made.

### Refactored Code

```javascript
/**
 * Processes an array of user data, filtering for active adults,
 * calculating engagement scores, and sorting by score.
 * @param {Array} users - The input array of user objects.
 * @returns {Array} Processed and sorted user data.
 */
export const processData = (users = []) => {
  if (!Array.isArray(users)) return [];

  return users
    .filter(isEligibleUser)
    .map(transformUser)
    .sort((a, b) => b.score - a.score);
};

// --- Configuration Constants ---
const USER_TYPE = 'user';
const STATUS_ACTIVE = 'active';
const MIN_AGE = 18;

const AGE_CATEGORIES = [
  { max: 25, label: 'young' },
  { max: 40, label: 'mid' },
  { max: 60, label: 'senior' },
  { max: Infinity, label: 'elder' },
];

const SPENDING_THRESHOLDS = [
  { min: 1000, score: 3 },
  { min: 500, score: 2 },
  { min: 100, score: 1 },
  { min: 0, score: 0 },
];

const REFERRAL_THRESHOLD = 5;
const REFERRAL_BONUS = 2;

// --- Helper Functions ---

const isEligibleUser = (user) =>
  user?.type === USER_TYPE &&
  user?.age >= MIN_AGE &&
  user?.status === STATUS_ACTIVE;

const getAgeCategory = (age) => {
  const category = AGE_CATEGORIES.find((c) => age < c.max);
  return category ? category.label : 'elder';
};

const calculateSpendingScore = (totalSpent) => {
  const tier = SPENDING_THRESHOLDS.find((t) => totalSpent >= t.min);
  return tier ? tier.score : 0;
};

const calculateTotalSpent = (purchases) => {
  if (!Array.isArray(purchases)) return 0;
  return purchases.reduce((sum, purchase) => sum + (purchase?.amount || 0), 0);
};

const transformUser = (user) => {
  const totalSpent = calculateTotalSpent(user.purchases);
  const baseScore = calculateSpendingScore(totalSpent);
  const referralBonus = user.referrals > REFERRAL_THRESHOLD ? REFERRAL_BONUS : 0;

  return {
    fullName: `${user.firstName} ${user.lastName}`,
    email: user.email?.toLowerCase().trim() ?? 'N/A',
    age: user.age,
    category: getAgeCategory(user.age),
    score: baseScore + referralBonus,
    totalSpent,
  };
};
```

### Key Decisions & Explanations

#### 1. Functional Pipeline (`filter`, `map`, `sort`)
*   **Original:** Used a `for` loop with nested `if` statements and manual array pushing.
*   **Refactor:** Chained array methods.
*   **Why:** This creates a declarative data flow. It is easier to read ("Filter eligible users, then transform them, then sort them") and reduces the cognitive load of tracking loop indices and mutable state (`results`, `i`, `j`, `total`).

#### 2. Extraction of Helper Functions
*   **Original:** All logic was inside one large function.
*   **Refactor:** Logic is split into `isEligibleUser`, `getAgeCategory`, `calculateSpendingScore`, and `transformUser`.
*   **Why:** This follows the **Single Responsibility Principle**. If the business logic for "age categories" changes, you only update `getAgeCategory`. It also makes the main `processData` function self-documenting.

#### 3. Removal of Magic Numbers
*   **Original:** Numbers like `18`, `1000`, `5`, `2` were scattered throughout the logic.
*   **Refactor:** Defined as constants (`MIN_AGE`, `SPENDING_THRESHOLDS`, `REFERRAL_BONUS`).
*   **Why:** If requirements change (e.g., adulthood age becomes 21), you update it in one place. It also gives context to what the numbers represent.

#### 4. Modern Syntax & Safety
*   **Variables:** Replaced `var` with `const` (for values that don't change) and arrow functions.
*   **Equality:** Replaced `==` with `===` to prevent type coercion bugs.
*   **String Interpolation:** Used template literals (`` `${first} ${last}` ``) instead of concatenation.
*   **Null Safety:** Used Optional Chaining (`user?.email`) and Nullish Coalescing (`?? 'N/A'`). This prevents crashes if `user` or `user.email` is undefined/null, which was a risk in the original code.

#### 5. Data Consistency
*   **Original:** `totalSpent` was only added to the object if purchases existed.
*   **Refactor:** `totalSpent` is always included (defaulting to `0`).
*   **Why:** Returning objects with inconsistent shapes (sometimes having a property, sometimes not) is a common source of bugs in downstream code. It is safer to always return the same structure.

#### 6. Input Validation
*   **Original:** Assumed `data` was a valid array.
*   **Refactor:** Added `users = []` default parameter and `Array.isArray` check.
*   **Why:** Functions should be defensive. If `processData` is called with `null` or `undefined`, it should fail gracefully rather than throwing a cryptic runtime error.

#### 7. Configuration Data Structures
*   **Original:** Used nested ternary operators for age and score logic (`age < 25 ? ... : age < 40 ? ...`).
*   **Refactor:** Used configuration arrays (`AGE_CATEGORIES`, `SPENDING_THRESHOLDS`) with `.find()`.
*   **Why:** Nested ternaries are hard to read and maintain. Configuration arrays make the rules data-driven and easier to extend (e.g., adding a new spending tier doesn't require changing logic code).
