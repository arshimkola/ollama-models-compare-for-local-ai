Here is the refactored code, followed by an explanation of the key decisions made to improve maintainability and readability.

### Refactored Code

```javascript
// Constants for configuration thresholds
const SCORE_THRESHOLDS = {
  HIGH: { limit: 1000, points: 3 },
  MEDIUM: { limit: 500, points: 2 },
  LOW: { limit: 100, points: 1 },
  DEFAULT: { points: 0 },
};

const REFERRAL_BONUS = 2;
const REFERRAL_THRESHOLD = 5;

/**
 * Processes raw user data to calculate scores and categorize users.
 * @param {Array} data - Raw array of user objects.
 * @returns {Array} - Sorted array of processed user objects.
 */
const processData = (data = []) => {
  return data
    .filter(isValidUser) // 1. Filter out invalid/inactive users first
    .map(transformUser)  // 2. Transform remaining users
    .sort((a, b) => b.score - a.score); // 3. Sort by score descending
};

// --- Helper Functions ---

/** Checks if a user meets the criteria for processing */
const isValidUser = (user) =>
  user.type === "user" && user.age >= 18 && user.status === "active";

/** Maps a raw user object to the desired output format */
const transformUser = (user) => {
  const totalSpent = calculateTotalSpent(user.purchases);
  const score = calculateUserScore(user, totalSpent);

  return {
    fullName: `${user.firstName} ${user.lastName}`,
    email: normalizeEmail(user.email),
    age: user.age,
    category: getAgeCategory(user.age),
    score,
    totalSpent,
  };
};

/** Calculates total purchase amount */
const calculateTotalSpent = (purchases = []) =>
  purchases.reduce((sum, purchase) => sum + (purchase.amount || 0), 0);

/** Normalizes email or returns default */
const normalizeEmail = (email) =>
  email ? email.toLowerCase().trim() : "N/A";

/** Determines age category based on numeric value */
const getAgeCategory = (age) => {
  if (age < 25) return "young";
  if (age < 40) return "mid";
  if (age < 60) return "senior";
  return "elder";
};

/** Calculates final score based on spending and referrals */
const calculateUserScore = (user, totalSpent) => {
  const spendScore = getSpendScore(totalSpent);
  const referralBonus = (user.referrals || 0) > REFERRAL_THRESHOLD ? REFERRAL_BONUS : 0;
  return spendScore + referralBonus;
};

/** Determines score based on total spent */
const getSpendScore = (total) => {
  if (total > SCORE_THRESHOLDS.HIGH.limit) return SCORE_THRESHOLDS.HIGH.points;
  if (total > SCORE_THRESHOLDS.MEDIUM.limit) return SCORE_THRESHOLDS.MEDIUM.points;
  if (total > SCORE_THRESHOLDS.LOW.limit) return SCORE_THRESHOLDS.LOW.points;
  return SCORE_THRESHOLDS.DEFAULT.points;
};

export default processData;
```

---

### Key Decisions & Improvements

#### 1. Adoption of Functional Programming (Declarative over Imperative)
The original code used a single large `for` loop with nested `if` statements, often referred to as "Arrow Code" or the "Arrow Anti-pattern."
*   **Refactoring:** I replaced the loop with a chain of array methods: `.filter()`, `.map()`, and `.sort()`.
*   **Benefit:** This separates the logic into three distinct phases: *Selection*, *Transformation*, and *Ordering*. It makes the data flow obvious at a glance (`processData` reads like a pipeline) rather than requiring the reader to mentally execute a loop.

#### 2. Single Responsibility Principle (SRP)
The original function did everything: validation, calculation, string manipulation, object creation, and sorting.
*   **Refactoring:** I extracted logic into small, dedicated helper functions (`isValidUser`, `calculateTotalSpent`, `getAgeCategory`, etc.).
*   **Benefit:** Each function does exactly one thing. This makes the code significantly easier to test (you can unit test `getAgeCategory` independently of the database connection) and easier to debug. If the logic for age categories changes, you only update `getAgeCategory` without touching the main processing loop.

#### 3. Flattening Nested Conditionals
The original code had three levels of nesting inside the loop (`if type`, `if age`, `if status`).
*   **Refactoring:** I moved the checks into `isValidUser` and used a guard clause approach (via `.filter`).
*   **Benefit:** This reduces cognitive complexity. We reject invalid data immediately rather than indenting logic deeper and deeper.

#### 4. Modern JavaScript Syntax (ES6+)
*   **`const`/`let`**: Replaced `var` to prevent hoisting issues and signal intent (immutability where possible).
*   **Arrow Functions**: Concise syntax for helper functions.
*   **Template Literals**: Used `${user.firstName} ${user.lastName}` instead of string concatenation for better readability.
*   **Default Parameters**: Used `(data = [])` in the main function to ensure safety if `null` or `undefined` is passed.
*   **Optional Chaining & Fallbacks**: In `normalizeEmail` and `calculateTotalSpent`, I used default values (e.g., `purchases = []`) and logical checks to handle missing data gracefully, preventing "cannot read property of undefined" errors.

#### 5. Avoiding Mutation
The original code created an object `item`, calculated `total`, assigned properties to `item`, conditionally added properties to `item`, and then pushed it.
*   **Refactoring:** The `transformUser` function builds and returns a single object in one pass. `calculateTotalSpent` uses `.reduce`, which is a safer, immutable pattern for accumulation than a mutable `total` variable.
*   **Benefit:** This prevents side effects. The data flows in one direction, making it easier to reason about state changes.

#### 6. Magic Numbers to Constants
The original code contained "magic numbers" like `1000`, `5`, and `2` inside the logic.
*   **Refactoring:** Extracted these to `SCORE_THRESHOLDS` and `REFERRAL_BONUS` constants.
*   **Benefit:** If business requirements change (e.g., the referral bonus becomes 3), you change the config constant, not the calculation logic. It also serves as self-documenting code.

#### 7. Immutable Object Construction
The original code conditionally added `totalSpent` to the object only if purchases existed. This creates an inconsistent object shape (some objects have `totalSpent`, others don't).
*   **Refactoring:** The refactored code always returns `totalSpent` (defaulting to 0).
*   **Benefit:** Consistent object shapes reduce bugs downstream. Consumers of this data array know that `totalSpent` will always exist.
