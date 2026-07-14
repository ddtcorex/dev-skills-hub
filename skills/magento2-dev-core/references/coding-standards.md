# Coding Standards Reference

This document provides detailed coding standards for Magento 2 development.

## PHPCS Rules (Magento2 Standard)

### File Structure

```php
<?php
/**
 * Vendor Name
 * Copyright (c) [YEAR] [Vendor Name]
 * See LICENSE for license details.
 */

declare(strict_types=1);
```

### Class Structure

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\Controller\Result\JsonFactory;
use Magento\Framework\Controller\Result\Json as JsonResult;

/**
 * Index controller for module
 */
class Index implements HttpGetActionInterface
{
    /** @var JsonFactory */
    private JsonFactory $resultJsonFactory;

    /**
     * @param JsonFactory $resultJsonFactory
     */
    public function __construct(JsonFactory $resultJsonFactory)
    {
        $this->resultJsonFactory = $resultJsonFactory;
    }

    /**
     * Execute controller
     *
     * @return JsonResult
     */
    public function execute(): JsonResult
    {
        return $this->resultJsonFactory->create()->setData(['success' => true]);
    }
}
```

### Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Classes | PascalCase | `ProductRepository` |
| Interfaces | PascalCase + Interface suffix | `ProductInterface` |
| Methods | camelCase | `getProductById()` |
| Properties | snake_case | `$product_id` |
| Constants | UPPER_SNAKE_CASE | `CACHE_TAG` |
| Private props | snake_case with underscore | `$_cacheIdPrefix` |
| Tables | snake_case | `sales_order_item` |
| Columns | snake_case | `customer_id` |

### Type Hints

Always use strict type hints:

```php
// String types
public function setName(string $name): self;

// Return types
public function getId(): int;

// Nullable types
public function getOptional(): ?string;

// Union types (PHP 8+)
public function process(int|string $input): int|string;

// Void return
public function save(ProductInterface $product): void;
```

### PHPCS Configuration

```xml
<!-- phpcs.xml -->
<?xml version="1.0"?>
<ruleset name="VendorModule">
    <description>Vendor Module coding standard</description>
    <file>app/code/Vendor/Module</file>
    <exclude-pattern>*/Test/*</exclude-pattern>
    <arg name="standard">Magento2</arg>
    <arg name="encoding">utf-8</arg>
    <arg name="tab-width">4</arg>
</ruleset>
```

## PHPStan Configuration

```neon
# phpstan.neon
parameters:
    level: 6
    paths:
        - app/code/Vendor/Module
    excludePaths:
        - app/code/Vendor/Module/Test
    checkGenericClassInNonGenericObjectType: false
    reportUnmatchedIgnoredErrors: false
    ignoreErrors:
        - '#Call to an undefined method Mage_.*#'
```

## Common PHPCS Violations

| Error | Correct | Wrong |
|-------|---------|-------|
| Missing docblock | `/** @var Product */` | `/** @var */` |
| Unused use | Remove or use | `use Unused\Class` |
| Line too long | Split line | Continuation on column 200 |
| Blank line after import | `use X;↵↵class` | `use X;↵class` |
| Missing license header | Add header | No header |

## Automated Commands

```bash
# Run PHPCS
vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module

# Fix auto-fixable issues
vendor/bin/phpcbf --standard=Magento2 app/code/Vendor/Module

# Run PHPStan
vendor/bin/phpstan analyse app/code/Vendor/Module -c phpstan.neon

# Combined check
vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module && \
vendor/bin/phpstan analyse app/code/Vendor/Module -c phpstan.neon
```