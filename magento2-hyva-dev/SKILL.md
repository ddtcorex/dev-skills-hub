---
name: magento2-hyva-dev
description: |
  Expert Hyvä theme development for Magento 2. Use when:
  - "Create Hyvä theme", "setup child theme", "build Alpine.js component"
  - "Make this CSP-compliant", "add Tailwind CSS classes"
  - "Convert Luma to Hyvä", "migrate Knockout to Alpine"
  - "Hyvä checkout", "Hyvä React components"
  - "Tailwind configuration", "CSP nonce registration"

  This is a SPECIALIZED skill for Hyvä-specific patterns. DEPENDENT on magento2-dev-core
  for PHP/backend patterns.
compatibility: claude, codex, opencode, copilot
depends: [magento2-dev-core]
metadata:
  audience: frontend developers
  workflow: magento
  references:
    - https://github.com/hyva-themes/hyva-ai-tools
    - https://docs.hyva.io/
---

# Magento 2 Hyvä Developer

Hyvä is a modern Magento 2 frontend framework with dramatically simplified JavaScript and CSS. This skill covers Hyvä-specific patterns.

## Hyvä vs Luma Comparison

| Aspect | Luma | Hyvä |
|--------|------|------|
| JavaScript | ~200 resources (RequireJS/Knockout) | 2 resources (Alpine.js) |
| CSS | LESS-based | Tailwind CSS |
| Bundle Size | 500KB+ | <50KB |
| Core Web Vitals | Challenging | Optimized |
| Learning Curve | Steep | Gentle |
| Maintenance | Complex | Simple |

## Theme Structure

```
app/design/frontend/Vendor/Theme/
├── registration.php
├── theme.xml
├── composer.json
├── package.json
├── tailwind.config.js
├── package.json
├── web/
│   ├── tailwind/
│   │   ├── base/           # Preflight, resets
│   │   ├── components/     # Reusable components
│   │   │   ├── buttons.css
│   │   │   ├── forms.css
│   │   │   └── messages.css
│   │   ├── utilities/     # Custom utilities
│   │   └── theme/         # Page-specific
│   └── js/
│       └── alpinejs/      # Alpine components
├── layout/
│   └── default.xml
└── templates/
    └── ...
```

## CSP (Content Security Policy) Compliance

### Critical: PCI-DSS 4.0 (Required since April 2025)

Payment pages MUST NOT use:
- `unsafe-eval` CSP directive
- `unsafe-inline` CSP directive

### CSP Nonce Registration

**Every inline script MUST register with CSP:**

```php
<?php
/** @var Hyva\Theme\ViewModel\HyvaCsp $hyvaCsp */
use Hyva\Theme\ViewModel\HyvaCsp;
?>

<!-- Register script before using nonce -->
<?php $hyvaCsp->registerInlineScript() ?>
<script nonce="<?= $cspNonce ?>">
    // CSP-compliant code
</script>
```

### CSP-Compatible Alpine.js Patterns

**WRONG (CSP violations):**
```html
<!-- These patterns break CSP -->
<button @click="count++">Add</button>
<span x-show="!loading">Ready</span>
<input :value="name" @input="name = $event.target.value">
```

**CORRECT (CSP-compliant):**
```html
<!-- Use methods for mutations -->
<button @click="increment">Add</button>
<span x-show="isNotLoading">Ready</span>
<input :value="name" @input="updateName">
```

```javascript
function initComponent() {
    return {
        count: 0,
        loading: true,
        name: '',

        increment() {
            this.count++;
        },

        isNotLoading() {
            return !this.loading;
        },

        updateName(event) {
            this.name = event.target.value;
        }
    }
}
window.addEventListener('alpine:init', () => {
    Alpine.data('initComponent', initComponent);
}, {once: true})
```

### Registering Alpine Components

```php
<?php
/** @var Hyva\Theme\ViewModel\HyvaCsp $hyvaCsp */
?>

<script>
function initProductSlider() {
    return {
        products: [],
        currentIndex: 0,

        init() {
            // Initialization
        },

        next() {
            this.currentIndex = (this.currentIndex + 1) % this.products.length;
        },

        prev() {
            this.currentIndex = (this.currentIndex - 1 + this.products.length) % this.products.length;
        }
    }
}
window.addEventListener('alpine:init', () => Alpine.data('initProductSlider', initProductSlider), {once: true})
</script>
<?php $hyvaCsp->registerInlineScript() ?>
```

## Alpine.js Component Structure

### Basic Component

```javascript
// web/js/alpinejs/Example.js
function initExample() {
    return {
        // Observable state
        isOpen: false,
        items: [],
        selectedId: null,

        // Computed (reactive)
        get hasItems() {
            return this.items.length > 0;
        },

        // Methods
        toggle() {
            this.isOpen = !this.isOpen;
        },

        select(id) {
            this.selectedId = id;
        },

        // Lifecycle
        init() {
            // Called when component initializes
            this.loadData();
        },

        loadData() {
            fetch('/api/data')
                .then(res => res.json())
                .then(data => this.items = data);
        }
    }
}
window.addEventListener('alpine:init', () => Alpine.data('initExample', initExample), {once: true})
```

### Template Usage

```html
<div x-data="initExample">
    <button @click="toggle">Toggle</button>

    <div x-show="isOpen">
        <template x-for="item in items" :key="item.id">
            <div @click="select(item.id)" :class="{ 'selected': selectedId === item.id }">
                <span x-text="item.name"></span>
            </div>
        </template>
    </div>
</div>

<script>
    function initExample() {
        // ... component logic
    }
    window.addEventListener('alpine:init', () => Alpine.data('initExample', initExample), {once: true})
</script>
<?php $hyvaCsp->registerInlineScript() ?>
```

### Passing Data from PHP

```php
<?php
/** @var \Magento\Framework\Escaper $escaper */
/** @var \Hyva\Theme\ViewModel\HyvaCsp $hyvaCsp */
$productsJson = json_encode($products, JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT);
?>

<div x-data="initProductList"
     data-products="<?= $escaper->escapeHtmlAttr($productsJson) ?>">
</div>

<script>
function initProductList() {
    return {
        products: [],

        init() {
            this.products = JSON.parse(this.$root.dataset.products || '[]');
        }
    }
}
window.addEventListener('alpine:init', () => Alpine.data('initProductList', initProductList), {once: true})
</script>
<?php $hyvaCsp->registerInlineScript() ?>
```

## Hyvä Utilities

Hyvä provides global utilities via the `hyva` object:

### Form Handling
```javascript
// Get form key
hyva.getFormKey()

// Submit form via POST
hyva.postForm({
    action: '/checkout',
    data: { product_id: 123, qty: 1 }
})

// Alternative with fetch
async function submitForm(url, data) {
    const formKey = hyva.getFormKey();
    const response = await fetch(url, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-Requested-With': 'XMLHttpRequest'
        },
        body: JSON.stringify({ ...data, form_key: formKey })
    });
    return response.json();
}
```

### Cookies
```javascript
hyva.getCookie('customer_segment')
hyva.setCookie('recent_viewed', productId, 30)
```

### Formatting
```javascript
hyva.formatPrice(price, showSign)
hyva.str('Hello {0}', name)
hyva.safeParseNumber(value)
```

### DOM Manipulation
```javascript
hyva.replaceDomElement('#target', '<span>New content</span>')
hyva.trapFocus(modalElement)
```

### Events
```javascript
// After Alpine initialization
hyva.alpineInitialized(function() {
    console.log('Alpine ready');
})
```

## Tailwind CSS

### Directory Structure
```
web/tailwind/
├── base/
│   └── _styles.pcss         # Preflight, typography
├── components/
│   ├── _buttons.pcss
│   ├── _forms.pcss
│   └── _messages.pcss
├── utilities/
│   └── _custom-utilities.pcss
├── theme/
│   ├── _header.pcss
│   └── _footer.pcss
└── app.css                  # Main entry
```

### Build Commands
```bash
# Development with watch
npm run watch

# Production build
npm run build

# PurgeCSS config (auto-included)
# Tailwind automatically removes unused classes
```

### Common Classes

```html
<!-- Buttons -->
<button class="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700">
    Add to Cart
</button>

<!-- Forms -->
<input type="text" 
       class="w-full border border-gray-300 rounded px-3 py-2 focus:border-blue-500 focus:outline-none">

<!-- Cards -->
<div class="bg-white rounded-lg shadow-md p-4">
    <!-- Content -->
</div>

<!-- Grid -->
<div class="grid grid-cols-1 md:grid-cols-3 gap-4">
    <!-- Items -->
</div>
```

### Responsive Design
```html
<!-- Mobile first -->
<div class="w-full md:w-1/2 lg:w-1/3">
    <!-- Shows 100% on mobile, 50% on tablet, 33% on desktop -->
</div>
```

## Layout XML

### Hyvä-Specific Handles

```xml
<!-- Add Hyvä modal support -->
<update handle="hyva_modal"/>

<!-- Register custom Alpine component -->
<head>
    <script src="Hyva_Theme::js/alpinejs/my-component.js"/>
</head>
```

### Override Template
```xml
<referenceBlock name="product.info" template="MyCompany_MyTheme::product/view.phtml"/>
```

## Migration from Luma

### Step 1: Analyze Dependencies
```bash
# List jQuery dependencies
grep -r "require.*jquery" app/design/frontend/Vendor/Theme/web/js/

# Check Knockout bindings
grep -r "data-bind=" app/design/frontend/Vendor/Theme/templates/
```

### Step 2: Replace JavaScript
```javascript
// Luma Knockout
define(['ko'], function(ko) {
    return {
        items: ko.observableArray([]),
        addItem: function(item) {
            this.items.push(item);
        }
    };
});

// Hyvä Alpine
function initComponent() {
    return {
        items: [],
        addItem(item) {
            this.items.push(item);
        }
    }
}
```

### Step 3: Replace LESS with Tailwind
```less
// Luma LESS
.product-card {
    .lib-card();
    .lib-respond-to(@mobile, { width: 100%; });
}

// Hyvä Tailwind
<div class="bg-white rounded-lg shadow-md p-4 w-full md:w-1/2">
```

### Step 4: Update Templates
```php
// Luma (Knockout)
<span data-bind="text: product.name"></span>

// Hyvä (Alpine)
<span x-text="product.name"></span>
```

## Official Hyvä AI Tools

Hyvä provides official AI skills for various assistants:

| Tool | Purpose | Install |
|------|---------|---------|
| hyva-alpine-component | CSP-compatible Alpine components | `.opencode/skills/` |
| hyva-child-theme | Theme creation | `.opencode/skills/` |
| hyva-cms-component | CMS blocks | `.opencode/skills/` |
| hyva-ui-component | UI component installation | `.opencode/skills/` |

```bash
# Install Hyvä AI tools
curl -fsSL https://raw.githubusercontent.com/hyva-themes/hyva-ai-tools/main/install.sh | sh -s opencode
```

## Verification

```bash
# Build CSS
cd web/tailwind && npm run build

# Clear cache
bin/magento cache:clean layout block_html full_page

# Static content deploy
bin/magento setup:static-content:deploy -f

# Test in browser
# Check console for CSP errors
# Test with CSP headers enabled
```

## Usage

**Trigger phrases:**
- "Create Hyvä component"
- "Build Alpine.js for Hyvä"
- "Make CSP-compliant"
- "Convert to Hyvä"
- "Tailwind styling"
- "Hyvä checkout customization"