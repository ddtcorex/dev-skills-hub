# Security Best Practices Reference

This document covers security patterns and vulnerabilities in Magento 2.

## XSS Prevention

### Template Escaping

```php
<?php
/** @var \Magento\Framework\Escaper $escaper */
?>

<!-- HTML Content -->
<?= $escaper->escapeHtml($productName) ?>

<!-- HTML Attributes -->
<div class="<?= $escaper->escapeHtmlAttr($cssClass) ?>"
     data-id="<?= $escaper->escapeHtmlAttr($id) ?>">

<!-- URLs (ensure safe protocol) -->
<a href="<?= $escaper->escapeUrl($url) ?>">Link</a>

<!-- JavaScript Strings -->
<script>
    var config = <?= /* @noEscape */ json_encode($config) ?>;
    var productName = '<?= $escaper->escapeJs($productName) ?>';
</script>

<!-- CSS (escape for style attributes) -->
<div style="color: <?= $escaper->escapeCss($color) ?>">

<!-- Quotes -->
<input value="<?= $escaper->escapeQuote($value) ?>">
```

### Block Method Escaping

```php
// escapeHtml is default for getData()
$this->getData('product_name'); // auto-escaped if using getHtml()

// Explicit escaping methods
$block->escapeHtml($text);
$block->escapeJs($text);
$block->escapeUrl($url);
$block->escapeQuote($text);
```

### HTML Purifier (For Rich Content)

```php
// For user-generated HTML content
use Magento\Cms\Model\Template\FilterProvider;

public function __construct(
    private readonly FilterProvider $filterProvider
) {}

public function getFilteredContent(string $html): string
{
    return $this->filterProvider->getPageFilter()->filter($html);
}
```

## CSRF Protection

### Form Key in Forms

```php
<form action="<?= $escaper->escapeUrl($block->getSubmitUrl()) ?>" method="post">
    <?= $block->getBlockHtml('formkey') ?>
    <!-- or -->
    <input name="form_key" type="hidden" value="<?= $block->getFormKey() ?>"/>
</form>
```

### AJAX Requests

```php
// JavaScript - include form key
define([
    'jquery',
    'Magento_Customer/js/customer-data'
], function ($, customerData) {
    'use strict';

    return function (url, data) {
        var formKey = $.cookie('form_key');
        return $.ajax({
            url: url,
            type: 'POST',
            data: {
                ...data,
                form_key: formKey
            }
        });
    };
});
```

### Adminhtml Controllers

```php
// In admin controllers, form key validation is automatic
// But for custom actions, add explicit check
use Magento\Framework\Controller\ResultFactory;

public function execute()
{
    if (!$this->_formKeyValidator->validate($this->getRequest())) {
        $result = ['success' => false, 'error' => 'Invalid form key'];
        return $this->resultFactory->create(ResultFactory::TYPE_JSON)->setData($result);
    }
    // Continue processing
}
```

## SQL Injection Prevention

### Wrong - Direct SQL

```php
// WRONG - SQL Injection vulnerability
$id = $this->getRequest()->getParam('id');
$collection = $this->_resource->getConnection()
    ->fetchAll("SELECT * FROM products WHERE id = $id");
```

### Correct - Parameterized Queries

```php
// CORRECT - Using Magento's resource
$collection = $this->productCollection->create()
    ->addFieldToFilter('entity_id', $id);

// OR using resource connection with binding
$connection = $this->resource->getConnection();
$select = $connection->select()
    ->from($this->resource->getTableName('products'))
    ->where('entity_id = :id');
$bind = ['id' => (int)$id];
$product = $connection->fetchRow($select, $bind);
```

## Direct Object Instantiation

### Wrong

```php
// WRONG - Service Locator anti-pattern
use Magento\Framework\App\ObjectManager;
$objectManager = ObjectManager::getInstance();
$product = $objectManager->create(\Magento\Catalog\Model\Product::class);
```

### Correct

```php
// CORRECT - Use Dependency Injection
public function __construct(
    private readonly \Magento\Catalog\Api\ProductRepositoryInterface $productRepository,
    private readonly \Magento\Catalog\Model\ProductFactory $productFactory
) {}

public function createProduct(): \Magento\Catalog\Api\Data\ProductInterface
{
    // For reading
    $product = $this->productRepository->getById($id);

    // For creation
    $product = $this->productFactory->create();
}
```

## File Inclusion

### Wrong

```php
// WRONG - Path traversal vulnerability
$file = $this->getRequest()->getParam('file');
include '/path/to/files/' . $file;
```

### Correct

```php
// CORRECT - Validate and sanitize
$file = $this->getRequest()->getParam('file');
$allowedFiles = ['config.json', 'metadata.xml', 'data.csv'];
if (!in_array($file, $allowedFiles, true)) {
    throw new \Magento\Framework\Exception\LocalizedException(
        __('Invalid file specified')
    );
}
$safePath = $this->directoryList->getPath('var') . DIRECTORY_SEPARATOR . $file;
```

## Password Hashing

### Wrong

```php
// WRONG - Plain text comparison
if ($password === $storedHash) { }
```

### Correct

```php
// CORRECT - Use Magento's encryption
use Magento\Framework\Encryption\EncryptorInterface;

public function __construct(
    private readonly EncryptorInterface $encryptor
) {}

public function verify(string $password, string $hash): bool
{
    return $this->encryptor->verifyHash($password, $hash);
}

public function hash(string $password): string
{
    return $this->encryptor->getHash($password, 10);
}
```

## Session and Cookie Security

```php
// Secure cookie settings
use Magento\Framework\Stdlib\Cookie\CookieMetadataFactory;

public function setSecureCookie(string $value): void
{
    $metadata = $this->cookieMetadataFactory->createPublicCookieMetadata()
        ->setPath('/')
        ->setDuration(86400)
        ->setSecure(true)
        ->setHttpOnly(true)
        ->setSameSite('Strict');

    $this->cookieManager->setPublicCookie('my_cookie', $value, $metadata);
}
```

## Security Checklist

- [ ] All user input escaped before output
- [ ] Form keys in all POST forms
- [ ] No direct SQL queries (use Models/Collections)
- [ ] No ObjectManager instantiation in production code
- [ ] Passwords hashed (never stored plain text)
- [ ] File uploads validated and sanitized
- [ ] Rate limiting on sensitive endpoints
- [ ] Admin URL not predictable (`/admin` changed)
- [ ] Two-factor authentication enforced for admin
- [ ] CSP headers configured in `csp_whitelist.xml`