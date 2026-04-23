# Complexity — Function Complexity / Separation of Concerns

## Why It Matters

Complex functions are hard to read, hard to test, and a great place for bugs to hide. When a function does multiple things, changes can cause unintended side effects. Several simple functions are better than one complex function.

## Principles

- A function should do one thing only (Single Responsibility).
- Maintain a consistent level of abstraction within a function. Don't mix high-level logic with low-level details.
- When nesting gets deep, split the function or use early returns.
- When there are too many parameters, group them into an object or split the function.

## Checklist

- [ ] Excessive function length (varies by language, but review if over 40 lines)
- [ ] Nesting depth of 3 or more levels (if inside for inside if)
- [ ] A single function performing multiple independent tasks
- [ ] Functions with 4 or more parameters
- [ ] Boolean parameters that branch function behavior (flag arguments)
- [ ] Complex conditionals (combining 3 or more conditions)
- [ ] A single file/class bearing too many responsibilities
- [ ] Nested try-catch (try inside try) for compensation/rollback logic
- [ ] A single API handler handling authentication, validation, business logic, and side effects all at once

## Good Examples / Bad Examples

Bad example — function with mixed responsibilities:
```
function processOrder(order) {
  // Validation
  if (!order.items || order.items.length === 0) {
    throw new Error("Empty order");
  }
  if (!order.address) {
    throw new Error("No address");
  }

  // Price calculation
  let total = 0;
  for (const item of order.items) {
    let price = item.price * item.quantity;
    if (item.discount) {
      price = price * (1 - item.discount);
    }
    total += price;
  }

  // Save
  db.save({ ...order, total });

  // Send email
  sendEmail(order.email, `Order complete: $${total}`);
}
```

Good example — separated responsibilities:
```
function processOrder(order) {
  validateOrder(order);
  const total = calculateTotal(order.items);
  db.save({ ...order, total });
  notifyOrderComplete(order.email, total);
}
```

Bad example — deep nesting:
```
function findEligible(users) {
  const result = [];
  for (const user of users) {
    if (user.active) {
      if (user.age >= 18) {
        if (user.verified) {
          result.push(user);
        }
      }
    }
  }
  return result;
}
```

Good example — early return / filtering:
```
function findEligible(users) {
  return users.filter(user =>
    user.active && user.age >= 18 && user.verified
  );
}
```

## Anti-patterns

- **God Function** — A single function of hundreds of lines handling everything. Find boundaries and split it.
- **Flag Argument** — `process(data, true, false)`. Instead of branching with boolean parameters, separate into distinct functions.
- **Arrow Anti-Pattern** — Nested conditionals/loops forming an arrow shape (→). Flatten with guard clauses and early returns.
- **Over-splitting** — Extracting 2-line logic into a separate function is also problematic. If the abstraction cost exceeds the inline code cost, don't split.
- **Nested Try-Catch Compensation** — Nesting another try inside a try for rollback on business logic failure. Transaction-level support or separate function extraction is more appropriate.
