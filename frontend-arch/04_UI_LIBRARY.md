<!-- Author: Ratheesh G Kumar, Software Engineer, Team CurrencyXnge_fintech SaaS Product -->
# 🎨 UI Component Library & Design System

We use a **Atomic Design** approach implemented with **TailwindCSS** and **Headless UI** for accessibility.

## 1. Design Tokens (Tailwind Config)

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          500: '#2563eb', // Brand Blue
          600: '#1d4ed8',
        },
        accent: '#06b6d4', // Cyan
        danger: '#ef4444',
        success: '#22c55e',
        background: '#0f172a', // Slate 900
        surface: 'rgba(30, 41, 59, 0.7)', // Glassmorphism
      },
      fontFamily: {
        sans: ['Inter', 'sans-serif'],
      },
      backdropBlur: {
        xs: '2px',
      }
    }
  }
}
```

---

## 2. Core Components

### 2.1. `CurrencyInput`
A specialized input that handles formatting, flags, and drop-downs.

*   **Props**: `value`, `currency`, `onAmountChange`, `onCurrencyChange`
*   **Behavior**:
    *   Prevents non-numeric input.
    *   Auto-formats thousands separator (1,000.00).
    *   Shows flag icon based on Currency Code (using `circle-flags` library).

### 2.2. `LiveTicker`
A scrolling marquee or static card showing live rates.

*   **Logic**:
    *   Green arrow (▲) if `newRate > oldRate`.
    *   Red arrow (▼) if `newRate < oldRate`.
    *   Flash animation on update.

### 2.3. `StatusBadge`
Uniform status indicators across Admin and Customer apps.

*   **Pending**: Yellow/Orange bg.
*   **Completed**: Green bg.
*   **Failed**: Red bg with tooltip explaining error.

---

## 3. Responsive Layouts

### Desktop (Admin/Teller)
*   **Sidebar Navigation**: Collapsible for maximum screen real estate.
*   **Data Grids**: Dense tables for transactions (AG Grid or TanStack Table).
*   **Modals**: Large centered dialogs for complex actions.

### Mobile (Customer)
*   **Bottom Navigation**: Thumb-friendly main menu (Home, Wallet, Remit, Profile).
*   **Cards**: Info presented in vertical stacks.
*   **sheets**: Slide-up bottom sheets for filters / forms instead of full modals.
