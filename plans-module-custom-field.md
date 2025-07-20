Kế hoạch chi tiết để build module Custom Fields cho Akaunting:

## **1. PHÂN TÍCH YÊU CẦU**

### **Core Features:**

- Tạo/sửa/xóa custom fields
- 8 loại field: Text, TextArea, Checkbox, Toggle, Date, DateTime, Time, Select
- Gán fields vào các entities: Invoice, Customer, Item, Account, Transfer, Vendor, Bill, Payment
- Validation rules
- Positioning (before/after existing fields)
- Icon và CSS class support

### **Advanced Features:**

- Multi-company support
- Import/Export custom field data
- Reporting integration
- API endpoints
- Field dependencies/conditional logic

## **2. KIẾN TRÚC DATABASE**

```sql
-- Bảng lưu định nghĩa custom fields
CREATE TABLE custom_fields (
    id INT PRIMARY KEY AUTO_INCREMENT,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) NOT NULL UNIQUE,
    type ENUM('text', 'textarea', 'checkbox', 'toggle', 'date', 'datetime', 'time', 'select') NOT NULL,
    location VARCHAR(100) NOT NULL, -- invoice, customer, item, etc.
    validation TEXT,
    default_value TEXT,
    options TEXT, -- JSON cho select options
    position_field VARCHAR(100), -- field để position
    position_type ENUM('before', 'after') DEFAULT 'after',
    icon VARCHAR(50),
    css_class VARCHAR(255),
    is_required BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INT DEFAULT 0,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    INDEX idx_company_location (company_id, location)
);

-- Bảng lưu giá trị custom fields
CREATE TABLE custom_field_values (
    id INT PRIMARY KEY AUTO_INCREMENT,
    custom_field_id INT NOT NULL,
    entity_type VARCHAR(100) NOT NULL, -- App\Models\Sale\Invoice
    entity_id INT NOT NULL,
    value TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    FOREIGN KEY (custom_field_id) REFERENCES custom_fields(id) ON DELETE CASCADE,
    UNIQUE KEY unique_field_entity (custom_field_id, entity_type, entity_id)
);
```

## **3. CẤU TRÚC MODULE**

```
modules/CustomFields/
├── composer.json
├── module.json
├── Config/
│   └── config.php
├── Database/
│   ├── Migrations/
│   │   ├── 2025_01_01_000001_create_custom_fields_table.php
│   │   └── 2025_01_01_000002_create_custom_field_values_table.php
│   └── Seeders/
├── Entities/
│   ├── CustomField.php
│   └── CustomFieldValue.php
├── Http/
│   ├── Controllers/
│   │   ├── CustomFieldController.php
│   │   └── Api/
│   │       └── CustomFieldController.php
│   ├── Requests/
│   │   ├── CustomFieldRequest.php
│   │   └── CustomFieldValueRequest.php
│   └── Middleware/
├── Providers/
│   ├── CustomFieldServiceProvider.php
│   └── EventServiceProvider.php
├── Resources/
│   ├── views/
│   │   ├── fields/
│   │   │   ├── create.blade.php
│   │   │   ├── edit.blade.php
│   │   │   ├── index.blade.php
│   │   │   └── show.blade.php
│   │   ├── partials/
│   │   │   ├── field-types/
│   │   │   │   ├── text.blade.php
│   │   │   │   ├── textarea.blade.php
│   │   │   │   ├── checkbox.blade.php
│   │   │   │   ├── select.blade.php
│   │   │   │   └── date.blade.php
│   │   │   └── field-renderer.blade.php
│   │   └── layouts/
│   ├── assets/
│   │   ├── js/
│   │   │   ├── custom-fields.js
│   │   │   └── field-builder.js
│   │   └── css/
│   │       └── custom-fields.css
│   └── lang/
│       ├── en/
│       └── vi/
├── Routes/
│   ├── web.php
│   └── api.php
├── Services/
│   ├── CustomFieldService.php
│   ├── CustomFieldRenderer.php
│   └── CustomFieldValidator.php
├── Traits/
│   └── HasCustomFields.php
├── Events/
│   ├── CustomFieldCreated.php
│   └── CustomFieldUpdated.php
├── Listeners/
│   ├── InjectCustomFields.php
│   └── SaveCustomFieldValues.php
└── Tests/
    ├── Feature/
    └── Unit/
```

## **4. IMPLEMENTATION PLAN**

### **Phase 1: Core Foundation (Tuần 1-2)**

```php
// 1. Tạo module structure
php artisan module:make CustomFields

// 2. Database setup
- Tạo migrations
- Tạo models với relationships
- Seed data mẫu

// 3. Basic CRUD cho Custom Fields
- Controller và Routes
- Views cơ bản
- Form validation
```

### **Phase 2: Field Types & Rendering (Tuần 3-4)**

```php
// 1. CustomFieldRenderer Service
class CustomFieldRenderer
{
    public function renderField($field, $value = null, $entity = null)
    {
        return view("customfields::partials.field-types.{$field->type}", [
            'field' => $field,
            'value' => $value,
            'entity' => $entity
        ])->render();
    }

    public function renderFormFields($location, $entity = null)
    {
        $fields = CustomField::active()
            ->forLocation($location)
            ->ordered()
            ->get();

        return $fields->map(function($field) use ($entity) {
            return $this->renderField($field, $this->getValue($field, $entity), $entity);
        })->implode('');
    }
}

// 2. HasCustomFields Trait
trait HasCustomFields
{
    public function customFieldValues()
    {
        return $this->morphMany(CustomFieldValue::class, 'entity');
    }

    public function getCustomField($code)
    {
        return $this->customFieldValues()
            ->whereHas('customField', function($q) use ($code) {
                $q->where('code', $code);
            })->first()?->value;
    }

    public function setCustomField($code, $value)
    {
        // Implementation
    }
}
```

### **Phase 3: Integration với Core Models (Tuần 5-6)**

```php
// 1. Event Listeners để inject fields
class InjectCustomFields
{
    public function handle($event)
    {
        // Inject vào form views
        if ($event instanceof ViewComposer) {
            $this->injectFieldsIntoView($event);
        }
    }
}

// 2. Hook vào existing forms
// Sử dụng Akaunting's Hook system
Hook::addAction('invoice.form.after', function($invoice) {
    echo app(CustomFieldRenderer::class)->renderFormFields('invoice', $invoice);
});

// 3. Modify existing models
// Thêm HasCustomFields trait vào Invoice, Customer, etc.
```

### **Phase 4: Advanced Features (Tuần 7-8)**

```php
// 1. Validation Service
class CustomFieldValidator
{
    public function validateEntity($entity, $data)
    {
        $fields = CustomField::forLocation($entity->getTable())->get();

        foreach ($fields as $field) {
            $this->validateField($field, $data[$field->code] ?? null);
        }
    }
}

// 2. API Integration
class Api\CustomFieldController extends Controller
{
    public function index(Request $request)
    {
        return CustomField::query()
            ->when($request->location, fn($q, $location) => $q->where('location', $location))
            ->paginate();
    }
}

// 3. Import/Export functionality
class CustomFieldImporter
{
    public function import($file, $mapping)
    {
        // Import custom field data
    }
}
```

## **5. TECHNICAL CONSIDERATIONS**

### **Frontend Integration:**

```javascript
// custom-fields.js
class CustomFieldBuilder {
    constructor() {
        this.initFieldBuilder();
        this.bindEvents();
    }

    initFieldBuilder() {
        // Dynamic field type switching
        $("#field-type").on("change", (e) => {
            this.showFieldTypeOptions(e.target.value);
        });
    }

    showFieldTypeOptions(type) {
        // Show/hide options based on field type
        $(".field-options").hide();
        $(`.field-options[data-type="${type}"]`).show();
    }
}
```

### **Performance Optimization:**

```php
// Eager loading relationships
$invoices = Invoice::with('customFieldValues.customField')->get();

// Caching field definitions
Cache::remember("custom_fields_{$location}_{$companyId}", 3600, function() {
    return CustomField::active()->forLocation($location)->get();
});
```

## **6. TESTING STRATEGY**

```php
// Feature Tests
class CustomFieldTest extends TestCase
{
    /** @test */
    public function can_create_custom_field()
    {
        $response = $this->post('/custom-fields', [
            'name' => 'Tax ID',
            'code' => 'tax_id',
            'type' => 'text',
            'location' => 'customer',
            'is_required' => true
        ]);

        $response->assertStatus(201);
        $this->assertDatabaseHas('custom_fields', ['code' => 'tax_id']);
    }

    /** @test */
    public function custom_field_appears_in_customer_form()
    {
        $field = CustomField::factory()->create(['location' => 'customer']);

        $response = $this->get('/customers/create');

        $response->assertSee($field->name);
    }
}
```

## **7. TIMELINE & MILESTONES**

| Tuần | Milestone         | Deliverables                         |
| ---- | ----------------- | ------------------------------------ |
| 1-2  | Foundation        | Database, Models, Basic CRUD         |
| 3-4  | Field Rendering   | All field types, Form integration    |
| 5-6  | Core Integration  | Hook vào existing forms, Save values |
| 7-8  | Advanced Features | API, Import/Export, Validation       |
| 9    | Testing & Polish  | Unit tests, Bug fixes, Documentation |
| 10   | Deployment        | Production ready, Performance tuning |

## **8. NEXT STEPS**

1. **Setup development environment** với docker-compose đã tạo
2. **Tạo module skeleton** và database structure
3. **Implement basic CRUD** cho custom fields
4. **Tạo field renderer service** và test với 1-2 field types
5. **Integration testing** với existing forms
