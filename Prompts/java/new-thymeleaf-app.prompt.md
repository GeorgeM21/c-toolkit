---
agent: 'Feature-Scaffolder'
description: 'Scaffold a new Spring MVC + Thymeleaf web application with component-based templates'
# Note: Uses Feature-Scaffolder agent if available in .github/agents/, otherwise falls back to default Copilot agent
---

# Scaffold a New Spring MVC + Thymeleaf Web Application

## Task

Create a complete **Spring MVC + Thymeleaf** server-rendered web application with proper template organization, form handling, validation, and optional HTMX integration for dynamic partial updates. This is the Java equivalent of a Blazor Server application -- server-side rendering with rich interactivity.

---

## Configuration

| Parameter          | Options                                           | Default                  |
|--------------------|---------------------------------------------------|--------------------------|
| Project Name       | *user-provided*                                   | `thymeleaf-app`          |
| Java Version       | 17, 21                                            | 21                       |
| CSS Framework      | Bootstrap, Tailwind, None                         | Bootstrap                |
| Template Engine    | Thymeleaf                                         | Thymeleaf                |
| Authentication     | None, Spring Security Form Login, OAuth2          | None                     |
| Include HTMX       | Yes / No                                          | Yes (for dynamic updates)|
| Include Docker     | Yes / No                                          | Yes                      |

---

## Project Structure

```
{{projectName}}/
├── pom.xml (or build.gradle)
├── Dockerfile
├── docker-compose.yml
├── src/
│   ├── main/
│   │   ├── java/com/example/{{projectName}}/
│   │   │   ├── Application.java
│   │   │   ├── config/
│   │   │   │   ├── WebMvcConfig.java
│   │   │   │   └── SecurityConfig.java          # if authentication selected
│   │   │   ├── controller/
│   │   │   │   ├── HomeController.java
│   │   │   │   └── ProductController.java
│   │   │   ├── service/
│   │   │   │   └── ProductService.java
│   │   │   ├── model/
│   │   │   │   ├── Product.java
│   │   │   │   └── ProductForm.java
│   │   │   └── repository/
│   │   │       └── ProductRepository.java
│   │   └── resources/
│   │       ├── templates/
│   │       │   ├── layout/
│   │       │   │   └── default.html             # base layout
│   │       │   ├── fragments/
│   │       │   │   ├── header.html
│   │       │   │   ├── nav.html
│   │       │   │   └── footer.html
│   │       │   ├── products/
│   │       │   │   ├── list.html
│   │       │   │   ├── detail.html
│   │       │   │   ├── form.html
│   │       │   │   └── _card.html               # reusable fragment
│   │       │   ├── index.html
│   │       │   └── error.html
│   │       ├── static/
│   │       │   ├── css/
│   │       │   │   └── app.css
│   │       │   └── js/
│   │       │       └── app.js
│   │       ├── application.yml
│   │       └── messages.properties
│   └── test/
│       └── java/com/example/{{projectName}}/
│           ├── controller/
│           │   └── ProductControllerTest.java
│           └── ApplicationTests.java
```

---

## Required Dependencies

- `spring-boot-starter-parent` 3.3.0
- `spring-boot-starter-web`
- `spring-boot-starter-thymeleaf`
- `thymeleaf-layout-dialect` (nz.net.ultraq.thymeleaf)
- `spring-boot-starter-validation`
- `spring-boot-starter-data-jpa`
- `h2` (runtime)
- `bootstrap` 5.3.3 via WebJars + `webjars-locator-core`
- `htmx.org` 1.9.12 via WebJars (npm)
- `spring-boot-starter-security` + `thymeleaf-extras-springsecurity6` (optional, if auth selected)
- `spring-boot-devtools` (runtime, optional)
- `spring-boot-starter-test` (test)

---

## Core Implementation

### 1. Application Configuration

```yaml
# application.yml
spring:
  application:
    name: {{projectName}}
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  h2:
    console:
      enabled: true
      path: /h2-console
  thymeleaf:
    cache: false          # disable cache during development
    prefix: classpath:/templates/
    suffix: .html

server:
  port: 8080
  error:
    whitelabel:
      enabled: false      # use custom error.html instead
```

```properties
# messages.properties (i18n)
app.name={{projectName}}
product.name=Name
product.description=Description
product.price=Price
product.category=Category
product.save.success=Product saved successfully!
product.delete.success=Product deleted successfully!
validation.required=This field is required
```

---

### 2. Domain Entity

```java
package com.example.thymeleafapp.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "product")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 255)
    private String name;

    @Column(length = 1000)
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(length = 100)
    private String category;

    @Column(nullable = false)
    private boolean inStock = true;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();

    @Column(nullable = false)
    private LocalDateTime updatedAt = LocalDateTime.now();

    @PreUpdate
    public void preUpdate() { this.updatedAt = LocalDateTime.now(); }

    // Constructors
    public Product() {}

    public Product(String name, String description, BigDecimal price,
                   String category, boolean inStock) {
        this.name = name; this.description = description;
        this.price = price; this.category = category; this.inStock = inStock;
    }

    // Getters and setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    public String getCategory() { return category; }
    public void setCategory(String category) { this.category = category; }
    public boolean isInStock() { return inStock; }
    public void setInStock(boolean inStock) { this.inStock = inStock; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### 3. Form Backing Object (DTO with Validation)

```java
package com.example.thymeleafapp.model;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import java.math.BigDecimal;

public class ProductForm {

    private Long id;

    @NotBlank(message = "Name is required")
    @Size(max = 255, message = "Name must not exceed 255 characters")
    private String name;

    @Size(max = 1000) private String description;
    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01") private BigDecimal price;
    @Size(max = 100) private String category;
    private boolean inStock = true;

    public ProductForm() {}

    public static ProductForm fromEntity(Product product) {
        ProductForm form = new ProductForm();
        form.setId(product.getId()); form.setName(product.getName());
        form.setDescription(product.getDescription()); form.setPrice(product.getPrice());
        form.setCategory(product.getCategory()); form.setInStock(product.isInStock());
        return form;
    }

    public Product toEntity() {
        return new Product(name, description, price, category, inStock);
    }

    public void applyTo(Product product) {
        product.setName(this.name); product.setDescription(this.description);
        product.setPrice(this.price); product.setCategory(this.category);
        product.setInStock(this.inStock);
    }

    // Getters and setters (generate all)
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    public String getCategory() { return category; }
    public void setCategory(String category) { this.category = category; }
    public boolean isInStock() { return inStock; }
    public void setInStock(boolean inStock) { this.inStock = inStock; }
}
```

---

### 4. Repository

```java
package com.example.thymeleafapp.repository;

import com.example.thymeleafapp.model.Product;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByCategory(String category, Pageable pageable);
    Page<Product> findByNameContainingIgnoreCase(String name, Pageable pageable);
}
```

### 5. Service Layer

```java
package com.example.thymeleafapp.service;

import com.example.thymeleafapp.model.Product;
import com.example.thymeleafapp.model.ProductForm;
import com.example.thymeleafapp.repository.ProductRepository;
import org.springframework.data.domain.*;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.Optional;

@Service
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository repository;
    public ProductService(ProductRepository repository) { this.repository = repository; }

    public Page<Product> findAll(int page, int size) {
        return repository.findAll(PageRequest.of(page, size, Sort.by("createdAt").descending()));
    }
    public Page<Product> search(String query, int page, int size) {
        return repository.findByNameContainingIgnoreCase(query, PageRequest.of(page, size, Sort.by("name").ascending()));
    }
    public Optional<Product> findById(Long id) { return repository.findById(id); }

    @Transactional
    public Product create(ProductForm form) { return repository.save(form.toEntity()); }

    @Transactional
    public Product update(Long id, ProductForm form) {
        Product product = repository.findById(id).orElseThrow(() -> new ProductNotFoundException(id));
        form.applyTo(product);
        return repository.save(product);
    }

    @Transactional
    public void delete(Long id) {
        if (!repository.existsById(id)) throw new ProductNotFoundException(id);
        repository.deleteById(id);
    }

    public static class ProductNotFoundException extends RuntimeException {
        public ProductNotFoundException(Long id) { super("Product not found with id: " + id); }
    }
}
```

---

### 6. Controllers

#### HomeController

```java
@Controller
public class HomeController {
    @GetMapping("/")
    public String index(Model model) { model.addAttribute("pageTitle", "Home"); return "index"; }
}
```

#### ProductController

```java
package com.example.thymeleafapp.controller;

import com.example.thymeleafapp.model.Product;
import com.example.thymeleafapp.model.ProductForm;
import com.example.thymeleafapp.service.ProductService;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/products")
public class ProductController {

    private final ProductService productService;
    public ProductController(ProductService productService) { this.productService = productService; }

    @GetMapping
    public String list(@RequestParam(defaultValue = "0") int page,
                       @RequestParam(defaultValue = "10") int size,
                       @RequestParam(required = false) String search, Model model) {
        Page<Product> products = (search != null && !search.isBlank())
            ? productService.search(search, page, size) : productService.findAll(page, size);
        if (search != null) model.addAttribute("search", search);
        model.addAttribute("products", products);
        model.addAttribute("pageTitle", "Products");
        return "products/list";
    }

    @GetMapping(headers = "HX-Request")
    public String listPartial(@RequestParam(defaultValue = "0") int page,
                              @RequestParam(defaultValue = "10") int size,
                              @RequestParam(required = false) String search, Model model) {
        Page<Product> products = (search != null && !search.isBlank())
            ? productService.search(search, page, size) : productService.findAll(page, size);
        model.addAttribute("products", products);
        return "products/list :: product-table";
    }

    @GetMapping("/{id}")
    public String detail(@PathVariable Long id, Model model) {
        Product product = productService.findById(id)
            .orElseThrow(() -> new ProductService.ProductNotFoundException(id));
        model.addAttribute("product", product);
        model.addAttribute("pageTitle", product.getName());
        return "products/detail";
    }

    @GetMapping("/new")
    public String createForm(Model model) {
        model.addAttribute("productForm", new ProductForm());
        model.addAttribute("pageTitle", "New Product");
        model.addAttribute("isEdit", false);
        return "products/form";
    }

    @PostMapping
    public String create(@Valid @ModelAttribute("productForm") ProductForm form,
                         BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {
        if (bindingResult.hasErrors()) {
            model.addAttribute("pageTitle", "New Product"); model.addAttribute("isEdit", false);
            return "products/form";
        }
        Product created = productService.create(form);
        redirectAttributes.addFlashAttribute("successMessage",
            "Product '" + created.getName() + "' created successfully!");
        return "redirect:/products/" + created.getId();
    }

    @GetMapping("/{id}/edit")
    public String editForm(@PathVariable Long id, Model model) {
        Product product = productService.findById(id)
            .orElseThrow(() -> new ProductService.ProductNotFoundException(id));
        model.addAttribute("productForm", ProductForm.fromEntity(product));
        model.addAttribute("pageTitle", "Edit " + product.getName());
        model.addAttribute("isEdit", true);
        return "products/form";
    }

    @PostMapping("/{id}")
    public String update(@PathVariable Long id, @Valid @ModelAttribute("productForm") ProductForm form,
                         BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {
        if (bindingResult.hasErrors()) {
            model.addAttribute("pageTitle", "Edit Product"); model.addAttribute("isEdit", true);
            return "products/form";
        }
        Product updated = productService.update(id, form);
        redirectAttributes.addFlashAttribute("successMessage",
            "Product '" + updated.getName() + "' updated successfully!");
        return "redirect:/products/" + updated.getId();
    }

    @PostMapping("/{id}/delete")
    public String delete(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        productService.delete(id);
        redirectAttributes.addFlashAttribute("successMessage", "Product deleted successfully!");
        return "redirect:/products";
    }

    @DeleteMapping(value = "/{id}", headers = "HX-Request")
    @ResponseBody
    public String deleteHtmx(@PathVariable Long id) {
        productService.delete(id);
        return "";
    }

    @ExceptionHandler(ProductService.ProductNotFoundException.class)
    public String handleNotFound(ProductService.ProductNotFoundException ex, Model model) {
        model.addAttribute("errorMessage", ex.getMessage());
        model.addAttribute("pageTitle", "Not Found");
        return "error";
    }
}
```

---

### 7. Thymeleaf Templates

#### Base Layout (`templates/layout/default.html`)

This template demonstrates the **Thymeleaf Layout Dialect** decorator pattern -- all content pages decorate this layout.

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title th:text="${pageTitle} ? ${pageTitle} + ' | {{projectName}}' : '{{projectName}}'">
        {{projectName}}
    </title>
    <link rel="stylesheet" th:href="@{/webjars/bootstrap/css/bootstrap.min.css}">
    <link rel="stylesheet" th:href="@{/css/app.css}">
    <script th:src="@{/webjars/htmx.org/dist/htmx.min.js}" defer></script>
</head>
<body>
    <div th:replace="~{fragments/nav :: nav}"></div>

    <!-- Flash messages -->
    <div class="container mt-3">
        <div th:if="${successMessage}" class="alert alert-success alert-dismissible fade show" role="alert">
            <span th:text="${successMessage}"></span>
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
        </div>
        <div th:if="${errorMessage}" class="alert alert-danger alert-dismissible fade show" role="alert">
            <span th:text="${errorMessage}"></span>
            <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
        </div>
    </div>

    <main class="container py-4" layout:fragment="content">
        <!-- Child template content goes here -->
    </main>

    <div th:replace="~{fragments/footer :: footer}"></div>
    <script th:src="@{/webjars/bootstrap/js/bootstrap.bundle.min.js}"></script>
    <script th:src="@{/js/app.js}"></script>
</body>
</html>
```

#### Product List Page (`templates/products/list.html`)

This is the main content page example showing HTMX search, table rendering, pagination, and inline delete.

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
      layout:decorate="~{layout/default}">
<body>
<main layout:fragment="content">
    <div th:replace="~{fragments/header :: header('Products', 'Manage your product catalogue')}"></div>

    <div class="d-flex justify-content-between align-items-center mb-3">
        <form class="d-flex" style="max-width: 400px;">
            <input type="search" name="search" class="form-control me-2"
                   placeholder="Search products..." th:value="${search}"
                   hx-get="/products" hx-target="#product-table"
                   hx-trigger="input changed delay:300ms, search" hx-swap="innerHTML">
        </form>
        <a th:href="@{/products/new}" class="btn btn-success">+ New Product</a>
    </div>

    <div id="product-table" th:fragment="product-table">
        <div th:if="${products.isEmpty()}" class="alert alert-info">No products found.</div>
        <table th:if="${!products.isEmpty()}" class="table table-striped table-hover">
            <thead class="table-dark">
                <tr>
                    <th>Name</th><th>Category</th>
                    <th class="text-end">Price</th><th class="text-center">In Stock</th>
                    <th class="text-end">Actions</th>
                </tr>
            </thead>
            <tbody>
                <tr th:each="product : ${products.content}" th:id="'row-' + ${product.id}">
                    <td><a th:href="@{/products/{id}(id=${product.id})}" th:text="${product.name}">Name</a></td>
                    <td th:text="${product.category} ?: '-'">Category</td>
                    <td class="text-end" th:text="${#numbers.formatDecimal(product.price, 1, 2)}">0.00</td>
                    <td class="text-center">
                        <span class="badge" th:classappend="${product.inStock ? 'bg-success' : 'bg-secondary'}"
                              th:text="${product.inStock ? 'Yes' : 'No'}">Yes</span>
                    </td>
                    <td class="text-end">
                        <a th:href="@{/products/{id}/edit(id=${product.id})}" class="btn btn-sm btn-outline-primary">Edit</a>
                        <button class="btn btn-sm btn-outline-danger"
                                th:attr="hx-delete=@{/products/{id}(id=${product.id})}"
                                hx-target="closest tr" hx-swap="outerHTML swap:500ms"
                                hx-confirm="Are you sure you want to delete this product?">Delete</button>
                    </td>
                </tr>
            </tbody>
        </table>

        <!-- Pagination -->
        <nav th:if="${products.totalPages > 1}" aria-label="Product pagination">
            <ul class="pagination justify-content-center">
                <li class="page-item" th:classappend="${products.first ? 'disabled' : ''}">
                    <a class="page-link" th:href="@{/products(page=${products.number - 1}, size=${products.size}, search=${search})}">Previous</a>
                </li>
                <li th:each="i : ${#numbers.sequence(0, products.totalPages - 1)}" class="page-item"
                    th:classappend="${i == products.number ? 'active' : ''}">
                    <a class="page-link" th:href="@{/products(page=${i}, size=${products.size}, search=${search})}" th:text="${i + 1}">1</a>
                </li>
                <li class="page-item" th:classappend="${products.last ? 'disabled' : ''}">
                    <a class="page-link" th:href="@{/products(page=${products.number + 1}, size=${products.size}, search=${search})}">Next</a>
                </li>
            </ul>
        </nav>
    </div>
</main>
</body>
</html>
```

> **Note:** Generate additional content pages (`detail.html`, `form.html`, `_card.html`, `index.html`, `error.html`) and fragment files (`nav.html`, `header.html`, `footer.html`) following the same Thymeleaf Layout Dialect decorator pattern. The form page should use `th:field` binding, inline `th:errors` validation display, and the `isEdit` flag to toggle create/update behavior.

---

### 8. Static Assets

Generate `static/css/app.css` extending Bootstrap with HTMX-specific styles (`.htmx-indicator`, `.htmx-swapping` fade-out for deleted rows, card hover effects) and `static/js/app.js` with HTMX CSRF token injection and auto-dismiss alerts.

---

### 9. WebMvc Configuration

```java
package com.example.thymeleafapp.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.*;

@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**").addResourceLocations("classpath:/static/");
    }

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/about").setViewName("about");
    }
}
```

### 10. Spring Security Configuration (Optional)

Generate this only when the user selects **Spring Security Form Login** or **OAuth2**. Include `SecurityFilterChain` with form login, URL-based authorization (admin-only for create/edit/delete), and in-memory demo users.

---

### 11. Docker Support

For Dockerfile and docker-compose.yml, use the `/add-dockerfile` prompt or generate:
- Dockerfile: `eclipse-temurin:21-jre-alpine`, copy JAR, expose 8080
- docker-compose: app service + PostgreSQL 16 with healthcheck and volume

---

### 12. Controller Test

```java
package com.example.thymeleafapp.controller;

import com.example.thymeleafapp.model.Product;
import com.example.thymeleafapp.service.ProductService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.bean.MockBean;
import org.springframework.data.domain.PageImpl;
import org.springframework.test.web.servlet.MockMvc;
import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired private MockMvc mockMvc;
    @MockBean private ProductService productService;

    @Test
    void shouldShowProductList() throws Exception {
        Product product = new Product("Test", "Desc", new BigDecimal("9.99"), "Cat", true);
        product.setId(1L);
        when(productService.findAll(anyInt(), anyInt()))
            .thenReturn(new PageImpl<>(List.of(product)));
        mockMvc.perform(get("/products"))
            .andExpect(status().isOk())
            .andExpect(view().name("products/list"))
            .andExpect(model().attributeExists("products"));
    }

    @Test
    void shouldRejectInvalidForm() throws Exception {
        mockMvc.perform(post("/products").param("name", "").param("price", ""))
            .andExpect(status().isOk())
            .andExpect(view().name("products/form"))
            .andExpect(model().hasErrors());
    }

    @Test
    void shouldCreateProduct() throws Exception {
        Product created = new Product("New Product", "Desc", new BigDecimal("19.99"), "Cat", true);
        created.setId(1L);
        when(productService.create(any())).thenReturn(created);
        mockMvc.perform(post("/products")
                .param("name", "New Product").param("price", "19.99").param("inStock", "true"))
            .andExpect(status().is3xxRedirection())
            .andExpect(redirectedUrl("/products/1"))
            .andExpect(flash().attributeExists("successMessage"));
    }
}
```

---

## Best Practices

- **Use Thymeleaf Layout Dialect** for DRY page layouts -- every page decorates a single base layout.
- **Keep templates small** -- extract reusable parts into fragments (`th:fragment` / `th:replace`).
- **Use `th:field`** for form binding -- it handles `name`, `id`, and `value` automatically.
- **Display validation errors inline** with `th:errors` and `is-invalid` CSS class.
- **Use `RedirectAttributes.addFlashAttribute()`** for Post-Redirect-Get success/error messages.
- **Use HTMX for dynamic updates** -- search-as-you-type, inline delete, partial refreshes without a JS framework.
- **Detect HTMX requests** via the `HX-Request` header to return fragments instead of full pages.
- **Separate form DTOs from entities** -- `ProductForm` carries validation; the JPA entity stays clean.
