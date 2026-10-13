# QuickBite — MEMBER 1: Customer & Account Management — COMPLETE CODE

*Backend (Java / Spring Boot) + Frontend (HTML + CSS) of this module in one document.*

---

## PART A — BACKEND (Java)

### `src/main/java/com/example/fooddelivery/model/Customer.java`
```java
package com.example.fooddelivery.model;

import com.example.fooddelivery.common.Person;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

/**
 * MEMBER 1 - CUSTOMER MODULE
 * OOP: INHERITANCE (BaseEntity -> Person -> Customer), POLYMORPHISM (getRole),
 *      ENCAPSULATION (private fields + getters/setters).
 */
@Entity
@Table(name = "customers")
public class Customer extends Person {

    @Column(nullable = false)
    private String password;

    private int loyaltyPoints = 0;

    private boolean active = true;

    public Customer() {
    }

    /** Constructor OVERLOADING (OOP): a quick way to create a customer in code/tests. */
    public Customer(String name, String email, String password) {
        setName(name);
        setEmail(email);
        this.password = password;
    }

    public Customer(String name, String email, String password, String phone, String address) {
        this(name, email, password);
        setPhone(phone);
        setAddress(address);
    }

    /** POLYMORPHISM: different implementation from Driver. */
    @Override
    public String getRole() {
        return "CUSTOMER";
    }

    @Override
    public String getDisplayName() {
        return getId() == null ? getName() : getName() + " (Customer #" + getId() + ")";
    }

    /** Extra loyalty point for every completed order. */
    public void addLoyaltyPoints(int points) {
        this.loyaltyPoints += points;
    }

    public String getInitials() {
        if (getName() == null || getName().isBlank()) {
            return "?";
        }
        String[] parts = getName().trim().split("\\s+");
        if (parts.length == 1) {
            return parts[0].substring(0, 1).toUpperCase();
        }
        return (parts[0].substring(0, 1) + parts[parts.length - 1].substring(0, 1)).toUpperCase();
    }

    // ---------- Getters / Setters ----------

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public int getLoyaltyPoints() {
        return loyaltyPoints;
    }

    public void setLoyaltyPoints(int loyaltyPoints) {
        this.loyaltyPoints = loyaltyPoints;
    }

    public boolean isActive() {
        return active;
    }

    public void setActive(boolean active) {
        this.active = active;
    }
}

```

### `src/main/java/com/example/fooddelivery/repository/CustomerRepository.java`
```java
package com.example.fooddelivery.repository;

import com.example.fooddelivery.model.Customer;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

/** MEMBER 1 - data access for customers (CRUD comes free from JpaRepository). */
@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    Optional<Customer> findByEmail(String email);

    boolean existsByEmail(String email);

    List<Customer> findByNameContainingIgnoreCase(String name);

    List<Customer> findByActiveTrue();
}

```

### `src/main/java/com/example/fooddelivery/service/CustomerService.java`
```java
package com.example.fooddelivery.service;

import com.example.fooddelivery.model.Customer;

import java.util.List;
import java.util.Optional;

/**
 * OOP CONCEPT: ABSTRACTION
 * The controller only talks to this INTERFACE - it never knows (or cares)
 * whether the data comes from a database, a file or a test fake.
 * The implementation is in CustomerServiceImpl.
 */
public interface CustomerService {

    List<Customer> findAll();

    Customer findById(Long id);

    Customer save(Customer customer);

    Customer register(Customer customer);

    void deleteById(Long id);

    Optional<Customer> authenticate(String email, String password);

    List<Customer> search(String keyword);

    void toggleActive(Long id);

    long count();
}

```

### `src/main/java/com/example/fooddelivery/service/impl/CustomerServiceImpl.java`
```java
package com.example.fooddelivery.service.impl;

import com.example.fooddelivery.common.ResourceNotFoundException;
import com.example.fooddelivery.model.Customer;
import com.example.fooddelivery.repository.CustomerRepository;
import com.example.fooddelivery.service.CustomerService;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

/**
 * MEMBER 1 - business logic for the Customer module.
 * Notice "implements CustomerService" -> the controller depends on the
 * interface (ABSTRACTION), not on this class.
 */
@Service
public class CustomerServiceImpl implements CustomerService {

    private final CustomerRepository customerRepository;

    /** CONSTRUCTOR INJECTION (Spring gives us the repository automatically). */
    public CustomerServiceImpl(CustomerRepository customerRepository) {
        this.customerRepository = customerRepository;
    }

    @Override
    public List<Customer> findAll() {
        return customerRepository.findAll();
    }

    @Override
    public Customer findById(Long id) {
        return customerRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.of("Customer", id));
    }

    @Override
    public Customer save(Customer customer) {
        return customerRepository.save(customer);
    }

    /** CREATE with business rules: email must be unique. */
    @Override
    public Customer register(Customer customer) {
        if (customerRepository.existsByEmail(customer.getEmail())) {
            throw new IllegalStateException("A customer with the e-mail " + customer.getEmail() + " already exists.");
        }
        customer.setActive(true);
        customer.setLoyaltyPoints(0);
        return customerRepository.save(customer);
    }

    @Override
    public void deleteById(Long id) {
        if (!customerRepository.existsById(id)) {
            throw ResourceNotFoundException.of("Customer", id);
        }
        customerRepository.deleteById(id);
    }

    /** Simple login check (demonstration project - not production security). */
    @Override
    public Optional<Customer> authenticate(String email, String password) {
        return customerRepository.findByEmail(email)
                .filter(c -> c.getPassword().equals(password) && c.isActive());
    }

    @Override
    public List<Customer> search(String keyword) {
        if (keyword == null || keyword.isBlank()) {
            return findAll();
        }
        return customerRepository.findByNameContainingIgnoreCase(keyword.trim());
    }

    @Override
    public void toggleActive(Long id) {
        Customer customer = findById(id);
        customer.setActive(!customer.isActive());
        customerRepository.save(customer);
    }

    @Override
    public long count() {
        return customerRepository.count();
    }
}

```

### `src/main/java/com/example/fooddelivery/controller/CustomerController.java`
```java
package com.example.fooddelivery.controller;

import com.example.fooddelivery.common.SessionHelper;
import com.example.fooddelivery.model.Customer;
import com.example.fooddelivery.service.CustomerService;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

/**
 * MEMBER 1 - CUSTOMER MODULE
 * Full CRUD:
 *   CREATE -> GET  /customers/new   + POST /customers
 *   READ   -> GET  /customers       + GET  /customers/{id}
 *   UPDATE -> GET  /customers/{id}/edit + POST /customers/{id}
 *   DELETE -> POST /customers/{id}/delete
 */
@Controller
@RequestMapping("/customers")
public class CustomerController {

    private final CustomerService customerService;

    public CustomerController(CustomerService customerService) {
        this.customerService = customerService;
    }

    // ---------------- READ (list + search) ----------------
    @GetMapping
    public String list(@RequestParam(required = false) String keyword, Model model) {
        model.addAttribute("customers", customerService.search(keyword));
        model.addAttribute("keyword", keyword);
        return "customers/list";
    }

    // ---------------- CREATE ----------------
    @GetMapping("/new")
    public String createForm(Model model) {
        model.addAttribute("customer", new Customer());
        return "customers/form";
    }

    @PostMapping
    public String create(@ModelAttribute("customer") Customer customer,
                         RedirectAttributes redirectAttributes) {
        if (customer.getName() == null || customer.getName().isBlank()
                || customer.getEmail() == null || customer.getEmail().isBlank()
                || customer.getPassword() == null || customer.getPassword().isBlank()) {
            redirectAttributes.addFlashAttribute("error", "Name, e-mail and password are required.");
            return "redirect:/customers/new";
        }
        try {
            Customer saved = customerService.register(customer);
            redirectAttributes.addFlashAttribute("success", "Customer \"" + saved.getName() + "\" registered successfully.");
        } catch (IllegalStateException ex) {
            redirectAttributes.addFlashAttribute("error", ex.getMessage());
            return "redirect:/customers/new";
        }
        return "redirect:/customers";
    }

    // ---------------- READ (single) ----------------
    @GetMapping("/{id}")
    public String view(@PathVariable Long id, Model model) {
        model.addAttribute("customer", customerService.findById(id));
        return "customers/view";
    }

    // ---------------- UPDATE ----------------
    @GetMapping("/{id}/edit")
    public String editForm(@PathVariable Long id, Model model) {
        model.addAttribute("customer", customerService.findById(id));
        return "customers/form";
    }

    @PostMapping("/{id}")
    public String update(@PathVariable Long id,
                         @ModelAttribute("customer") Customer customer,
                         RedirectAttributes redirectAttributes) {
        Customer existing = customerService.findById(id);
        existing.setName(customer.getName());
        existing.setEmail(customer.getEmail());
        existing.setPhone(customer.getPhone());
        existing.setAddress(customer.getAddress());
        if (customer.getPassword() != null && !customer.getPassword().isBlank()) {
            existing.setPassword(customer.getPassword());
        }
        customerService.save(existing);
        redirectAttributes.addFlashAttribute("success", "Customer updated successfully.");
        return "redirect:/customers/" + id;
    }

    // ---------------- DELETE ----------------
    @PostMapping("/{id}/delete")
    public String delete(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        customerService.deleteById(id);
        redirectAttributes.addFlashAttribute("success", "Customer deleted.");
        return "redirect:/customers";
    }

    // ---------------- EXTRA: activate / deactivate ----------------
    @PostMapping("/{id}/toggle")
    public String toggleActive(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        customerService.toggleActive(id);
        redirectAttributes.addFlashAttribute("success", "Customer status changed.");
        return "redirect:/customers";
    }

    /** Demo convenience: sign in as this customer without typing the password. */
    @PostMapping("/{id}/login-as")
    public String loginAs(@PathVariable Long id, HttpSession session, RedirectAttributes redirectAttributes) {
        Customer customer = customerService.findById(id);
        SessionHelper.login(session, customer.getId(), customer.getName());
        redirectAttributes.addFlashAttribute("success", "You are now signed in as " + customer.getName() + ".");
        return "redirect:/";
    }
}

```

### `src/main/java/com/example/fooddelivery/controller/AuthController.java`
```java
package com.example.fooddelivery.controller;

import com.example.fooddelivery.common.SessionHelper;
import com.example.fooddelivery.model.Customer;
import com.example.fooddelivery.service.CustomerService;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.util.Optional;

/**
 * MEMBER 1 - CUSTOMER & ACCOUNT MODULE (sign in / sign out)
 * ---------------------------------------------------------
 * GET  /login   -> sign in form
 * POST /login   -> checks e-mail + password against the database (session based)
 * POST /logout  -> clears the session
 *
 * The dashboard itself (GET /) lives in HomeController and is shared by the team.
 */
@Controller
public class AuthController {

    private final CustomerService customerService;

    public AuthController(CustomerService customerService) {
        this.customerService = customerService;
    }

    @GetMapping("/login")
    public String loginForm() {
        return "customers/login";
    }

    @PostMapping("/login")
    public String login(@RequestParam String email,
                        @RequestParam String password,
                        HttpSession session,
                        RedirectAttributes redirectAttributes) {
        Optional<Customer> customer = customerService.authenticate(email, password);
        if (customer.isEmpty()) {
            redirectAttributes.addFlashAttribute("error", "Wrong e-mail or password (or the account is disabled).");
            return "redirect:/login";
        }
        SessionHelper.login(session, customer.get().getId(), customer.get().getName());
        redirectAttributes.addFlashAttribute("success", "Welcome back, " + customer.get().getName() + "!");
        return "redirect:/";
    }

    @PostMapping("/logout")
    public String logout(HttpSession session, RedirectAttributes redirectAttributes) {
        SessionHelper.logout(session);
        redirectAttributes.addFlashAttribute("success", "You have signed out.");
        return "redirect:/";
    }
}

```


## PART B — FRONTEND (HTML templates)

### `src/main/resources/templates/customers/list.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>QuickBite - Customers (Member 1)</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1>Customers</h1>
            <p class="muted">Member 1 &mdash; full CRUD: register, view, edit and delete customers.</p>
        </div>
        <a th:href="@{/customers/new}" class="btn">+ Add customer</a>
    </div>

    <form class="filter-form" th:action="@{/customers}" method="get">
        <div class="form-group">
            <label for="keyword">Search by name</label>
            <input type="text" id="keyword" name="keyword" th:value="${keyword}" placeholder="e.g. Kasun">
        </div>
        <button type="submit" class="btn btn-slate">Search</button>
        <a th:href="@{/customers}" class="btn btn-outline">Reset</a>
    </form>

    <div class="card empty" th:if="${#lists.isEmpty(customers)}">
        <span class="big">&#128100;</span>
        No customers found. Use <strong>Add customer</strong> to create the first one.
    </div>

    <div class="table-wrap" th:unless="${#lists.isEmpty(customers)}">
        <table class="table">
            <thead>
            <tr>
                <th>ID</th>
                <th>Name</th>
                <th>E-mail</th>
                <th>Phone</th>
                <th>Loyalty points</th>
                <th>Status</th>
                <th>Actions</th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="customer : ${customers}">
                <td th:text="${customer.id}">1</td>
                <td>
                    <span class="avatar" th:text="${customer.initials}" style="margin-right:8px;">KP</span>
                    <strong th:text="${customer.name}">Kasun Perera</strong>
                </td>
                <td th:text="${customer.email}">kasun@example.com</td>
                <td th:text="${customer.phone} ?: '-'">0771234567</td>
                <td th:text="${customer.loyaltyPoints}">0</td>
                <td>
                    <span class="badge" th:classappend="${customer.active} ? 'status-delivered' : 'status-cancelled'"
                          th:text="${customer.active} ? 'Active' : 'Disabled'">Active</span>
                </td>
                <td>
                    <div class="actions">
                        <a th:href="@{/customers/{id}(id=${customer.id})}" class="btn btn-sm btn-outline">View</a>
                        <a th:href="@{/customers/{id}/edit(id=${customer.id})}" class="btn btn-sm btn-slate">Edit</a>
                        <form th:action="@{/customers/{id}/login-as(id=${customer.id})}" method="post" class="inline-form">
                            <button type="submit" class="btn btn-sm btn-orange">Sign in as</button>
                        </form>
                        <form th:action="@{/customers/{id}/toggle(id=${customer.id})}" method="post" class="inline-form">
                            <button type="submit" class="btn btn-sm btn-amber"
                                    th:text="${customer.active} ? 'Disable' : 'Enable'">Disable</button>
                        </form>
                        <form th:action="@{/customers/{id}/delete(id=${customer.id})}" method="post" class="inline-form"
                              >
                            <button type="submit" class="btn btn-sm btn-danger">Delete</button>
                        </form>
                    </div>
                </td>
            </tr>
            </tbody>
        </table>
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```

### `src/main/resources/templates/customers/form.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title th:text="${customer.id == null} ? 'QuickBite - New customer' : 'QuickBite - Edit customer'">Customer form</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1 th:text="${customer.id == null} ? 'Register a new customer' : 'Edit customer'">Customer</h1>
            <p class="muted">Member 1 &mdash; CREATE / UPDATE form</p>
        </div>
        <a th:href="@{/customers}" class="btn btn-outline">&larr; Back to list</a>
    </div>

    <div class="card">
        <form th:action="${customer.id == null} ? @{/customers} : @{/customers/{id}(id=${customer.id})}" method="post">
            <div class="form-grid">
                <div class="form-group">
                    <label for="name">Full name *</label>
                    <input type="text" id="name" name="name" th:value="${customer.name}" required placeholder="Kasun Perera">
                </div>
                <div class="form-group">
                    <label for="email">E-mail *</label>
                    <input type="email" id="email" name="email" th:value="${customer.email}" required
                           placeholder="kasun@example.com">
                </div>
                <div class="form-group">
                    <label for="phone">Phone</label>
                    <input type="text" id="phone" name="phone" th:value="${customer.phone}" placeholder="0771234567">
                </div>
                <div class="form-group">
                    <label for="password">Password <span th:if="${customer.id != null}" class="muted">(leave blank to keep current)</span>
                        <span th:if="${customer.id == null}">*</span></label>
                    <input type="password" id="password" name="password" th:required="${customer.id == null}"
                           placeholder="••••">
                </div>
                <div class="form-group full">
                    <label for="address">Delivery address</label>
                    <textarea id="address" name="address" th:text="${customer.address}"
                              placeholder="No 12, Temple Road, Negombo"></textarea>
                </div>
            </div>
            <div class="form-actions">
                <button type="submit" class="btn" th:text="${customer.id == null} ? 'Register customer' : 'Save changes'">Save</button>
                <a th:href="@{/customers}" class="btn btn-outline">Cancel</a>
            </div>
        </form>
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```

### `src/main/resources/templates/customers/view.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title th:text="'QuickBite - ' + ${customer.name}">Customer</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1 th:text="${customer.name}">Customer</h1>
            <p class="muted">Member 1 &mdash; customer profile (READ)</p>
        </div>
        <div class="btn-row">
            <a th:href="@{/customers}" class="btn btn-outline">&larr; All customers</a>
            <a th:href="@{/customers/{id}/edit(id=${customer.id})}" class="btn btn-slate">Edit</a>
            <a th:href="@{/orders/my}" class="btn">My orders</a>
        </div>
    </div>

    <div class="card">
        <div style="display:flex; gap:16px; align-items:center; margin-bottom:14px;">
            <span class="avatar" th:text="${customer.initials}" style="width:64px;height:64px;font-size:1.4rem;">KP</span>
            <div>
                <h2 style="margin:0" th:text="${customer.name}">Kasun Perera</h2>
                <span class="chip" th:text="${customer.role}">CUSTOMER</span>
                <span class="badge" th:classappend="${customer.active} ? 'status-delivered' : 'status-cancelled'"
                      th:text="${customer.active} ? 'Active' : 'Disabled'">Active</span>
            </div>
        </div>

        <dl class="kv">
            <dt>Customer ID</dt>
            <dd th:text="${customer.id}">1</dd>
            <dt>E-mail</dt>
            <dd th:text="${customer.email}">kasun@example.com</dd>
            <dt>Phone</dt>
            <dd th:text="${customer.phone} ?: '-'">-</dd>
            <dt>Delivery address</dt>
            <dd th:text="${customer.address} ?: '-'">-</dd>
            <dt>Loyalty points</dt>
            <dd th:text="${customer.loyaltyPoints} + ' points (10 points per delivered order)'">0</dd>
            <dt>Display name</dt>
            <dd th:text="${customer.displayName}">-</dd>
        </dl>

        <div class="btn-row mt">
            <form th:action="@{/customers/{id}/login-as(id=${customer.id})}" method="post" class="inline-form">
                <button type="submit" class="btn btn-orange">Sign in as this customer</button>
            </form>
            <form th:action="@{/customers/{id}/delete(id=${customer.id})}" method="post" class="inline-form"
                  >
                <button type="submit" class="btn btn-danger">Delete customer</button>
            </form>
        </div>
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```

### `src/main/resources/templates/customers/login.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>QuickBite - Sign in</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="card" style="max-width:520px;margin:20px auto;">
        <h1>Sign in</h1>
        <p class="muted">Sign in to add food to your cart, place orders and write reviews.</p>

        <form th:action="@{/login}" method="post">
            <div class="form-group mb">
                <label for="email">E-mail</label>
                <input type="email" id="email" name="email" required placeholder="kasun@example.com">
            </div>
            <div class="form-group mb">
                <label for="password">Password</label>
                <input type="password" id="password" name="password" required placeholder="••••">
            </div>
            <button type="submit" class="btn" style="width:100%">Sign in</button>
        </form>

        <div class="alert alert-info mt">
            <strong>Demo accounts</strong> (password <code>1234</code> for all):<br>
            kasun@example.com &middot; nimali@example.com &middot; arun@example.com<br>
            <span class="muted">Or open the Customers page and click &ldquo;Sign in as&rdquo;.</span>
        </div>
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```


## PART C — CSS (shared stylesheet)

All pages share `src/main/resources/static/css/style.css`. For your pages the most relevant classes are `.avatar`, `.kv`, `.user-chip`, `.alert-*`. Full file below:

```css
/* ==========================================================================
   QUICKBITE - Online Food Delivery System
   Single stylesheet used by every page (pure CSS, no frameworks, no JS).
   ========================================================================== */

:root {
    --green: #16a34a;
    --green-dark: #15803d;
    --green-light: #dcfce7;
    --orange: #f97316;
    --orange-light: #ffedd5;
    --red: #dc2626;
    --red-light: #fee2e2;
    --amber: #d97706;
    --amber-light: #fef3c7;
    --blue: #2563eb;
    --blue-light: #dbeafe;
    --slate: #0f172a;
    --slate-soft: #1e293b;
    --muted: #64748b;
    --border: #e2e8f0;
    --bg: #f1f5f9;
    --card: #ffffff;
    --text: #1e293b;
    --radius: 14px;
    --shadow: 0 1px 3px rgba(15, 23, 42, .08), 0 4px 14px rgba(15, 23, 42, .06);
}

* { box-sizing: border-box; }

body {
    margin: 0;
    font-family: "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
    background: var(--bg);
    color: var(--text);
    line-height: 1.55;
}

.container { max-width: 1150px; margin: 0 auto; padding: 0 20px; }
main.container { padding-top: 26px; padding-bottom: 40px; min-height: 60vh; }

h1, h2, h3, h4 { color: var(--slate); line-height: 1.25; }
h1 { font-size: 1.75rem; margin: 0 0 6px; }
h2 { font-size: 1.3rem; margin: 0 0 14px; }
h3 { font-size: 1.05rem; margin: 0 0 8px; }
a { color: var(--green-dark); text-decoration: none; }
a:hover { text-decoration: underline; }
.muted { color: var(--muted); font-size: .9rem; }
.center { text-align: center; }
.mt { margin-top: 18px; }
.mb { margin-bottom: 18px; }

/* ------------------------------- navbar --------------------------------- */
.navbar {
    background: linear-gradient(90deg, var(--slate) 0%, var(--slate-soft) 100%);
    color: #fff;
    box-shadow: var(--shadow);
}
.nav-inner {
    display: flex;
    align-items: center;
    gap: 18px;
    flex-wrap: wrap;
    padding: 12px 20px;
}
.brand {
    font-size: 1.35rem;
    font-weight: 800;
    color: #fff;
    letter-spacing: .3px;
}
.brand:hover { text-decoration: none; }
.brand span { color: var(--orange); }
.nav-links { display: flex; gap: 4px; flex-wrap: wrap; flex: 1; }
.nav-links a {
    color: #e2e8f0;
    padding: 6px 11px;
    border-radius: 8px;
    font-size: .92rem;
}
.nav-links a:hover { background: rgba(255, 255, 255, .12); text-decoration: none; color: #fff; }
.nav-actions { display: flex; align-items: center; gap: 10px; }
.user-chip { display: flex; align-items: center; gap: 8px; color: #e2e8f0; font-size: .9rem; }
.user-chip form { margin: 0; }
.cart-link { color: #fff; border-color: #475569; }
.cart-link:hover { background: rgba(255,255,255,.1); }
.badge-count {
    background: var(--orange);
    color: #fff;
    border-radius: 999px;
    padding: 1px 8px;
    font-size: .78rem;
    margin-left: 4px;
}

/* ------------------------------- alerts --------------------------------- */
.alert {
    padding: 11px 16px;
    border-radius: 10px;
    margin: 12px 0 0;
    font-size: .93rem;
    border: 1px solid transparent;
}
.alert-success { background: var(--green-light); color: #14532d; border-color: #86efac; }
.alert-error { background: var(--red-light); color: #7f1d1d; border-color: #fca5a5; }
.alert-info { background: var(--blue-light); color: #1e3a8a; border-color: #93c5fd; }

/* ------------------------------- buttons -------------------------------- */
.btn {
    display: inline-block;
    background: var(--green);
    color: #fff !important;
    border: 1px solid var(--green);
    padding: 9px 16px;
    border-radius: 10px;
    font-size: .92rem;
    font-weight: 600;
    cursor: pointer;
    text-decoration: none !important;
    transition: background .15s ease, transform .05s ease;
}
.btn:hover { background: var(--green-dark); }
.btn:active { transform: translateY(1px); }
.btn-sm { padding: 5px 11px; font-size: .82rem; border-radius: 8px; }
.btn-outline { background: transparent; color: var(--green-dark) !important; }
.btn-outline:hover { background: var(--green-light); }
.btn-orange { background: var(--orange); border-color: var(--orange); }
.btn-orange:hover { background: #ea580c; }
.btn-danger { background: var(--red); border-color: var(--red); }
.btn-danger:hover { background: #b91c1c; }
.btn-amber { background: var(--amber); border-color: var(--amber); }
.btn-amber:hover { background: #b45309; }
.btn-slate { background: var(--slate-soft); border-color: var(--slate-soft); }
.btn-slate:hover { background: #0f172a; }
.btn-row { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; }
.inline { display: inline; }
.inline-form { display: inline; margin: 0; }

/* -------------------------------- cards --------------------------------- */
.card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    padding: 20px;
    margin-bottom: 20px;
}
.card.tight { padding: 14px 16px; }
.card-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(265px, 1fr));
    gap: 18px;
    margin-bottom: 22px;
}
.page-head {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    gap: 14px;
    flex-wrap: wrap;
    margin-bottom: 16px;
}
.page-head p { margin: 0; }

/* ------------------------------- hero ----------------------------------- */
.hero {
    background: linear-gradient(120deg, var(--slate) 0%, #14532d 60%, var(--green-dark) 100%);
    color: #fff;
    border-radius: var(--radius);
    padding: 34px 30px;
    margin-bottom: 22px;
    box-shadow: var(--shadow);
}
.hero h1 { color: #fff; font-size: 2rem; margin-bottom: 8px; }
.hero p { color: #dcfce7; margin: 0 0 18px; max-width: 640px; }

/* ------------------------------ stat cards ------------------------------ */
.stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(170px, 1fr));
    gap: 14px;
    margin-bottom: 24px;
}
.stat-card {
    background: var(--card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    box-shadow: var(--shadow);
    padding: 16px 18px;
}
.stat-card .stat-value { font-size: 1.6rem; font-weight: 800; color: var(--slate); }
.stat-card .stat-label { color: var(--muted); font-size: .85rem; }
.stat-card.accent-green { border-left: 4px solid var(--green); }
.stat-card.accent-orange { border-left: 4px solid var(--orange); }
.stat-card.accent-blue { border-left: 4px solid var(--blue); }
.stat-card.accent-amber { border-left: 4px solid var(--amber); }

/* -------------------------------- tables -------------------------------- */
.table-wrap { overflow-x: auto; }
table.table {
    width: 100%;
    border-collapse: collapse;
    background: var(--card);
    border-radius: 12px;
    overflow: hidden;
    box-shadow: var(--shadow);
    font-size: .92rem;
}
.table th {
    background: var(--slate-soft);
    color: #e2e8f0;
    text-align: left;
    padding: 11px 14px;
    font-size: .82rem;
    text-transform: uppercase;
    letter-spacing: .4px;
}
.table td { padding: 11px 14px; border-bottom: 1px solid var(--border); vertical-align: middle; }
.table tr:last-child td { border-bottom: none; }
.table tbody tr:hover { background: #f8fafc; }
.table .actions { white-space: nowrap; display: flex; gap: 6px; flex-wrap: wrap; }

/* -------------------------------- badges -------------------------------- */
.badge {
    display: inline-block;
    padding: 3px 10px;
    border-radius: 999px;
    font-size: .76rem;
    font-weight: 700;
    border: 1px solid transparent;
}
.status-placed { background: var(--blue-light); color: #1e40af; border-color: #93c5fd; }
.status-confirmed { background: #e0e7ff; color: #3730a3; border-color: #a5b4fc; }
.status-preparing { background: var(--amber-light); color: #92400e; border-color: #fcd34d; }
.status-out-for-delivery { background: var(--orange-light); color: #9a3412; border-color: #fdba74; }
.status-delivered { background: var(--green-light); color: #166534; border-color: #86efac; }
.status-cancelled { background: var(--red-light); color: #991b1b; border-color: #fca5a5; }
.status-pending { background: var(--amber-light); color: #92400e; border-color: #fcd34d; }
.status-paid { background: var(--green-light); color: #166534; border-color: #86efac; }
.status-failed { background: var(--red-light); color: #991b1b; border-color: #fca5a5; }
.status-refunded { background: #e2e8f0; color: #334155; border-color: #cbd5e1; }
.status-available { background: var(--green-light); color: #166534; border-color: #86efac; }
.status-on-delivery { background: var(--orange-light); color: #9a3412; border-color: #fdba74; }
.status-offline { background: #e2e8f0; color: #334155; border-color: #cbd5e1; }
.chip {
    display: inline-block;
    background: var(--bg);
    border: 1px solid var(--border);
    border-radius: 999px;
    padding: 2px 10px;
    font-size: .76rem;
    color: var(--muted);
}
.chip-veg { background: var(--green-light); color: #166534; border-color: #86efac; }

/* ------------------------------ menu cards ------------------------------ */
.dish-card { display: flex; flex-direction: column; }
.dish-card .dish-head { display: flex; justify-content: space-between; gap: 10px; align-items: flex-start; }
.dish-card .price { font-weight: 800; color: var(--green-dark); font-size: 1.05rem; white-space: nowrap; }
.dish-card p { color: var(--muted); font-size: .88rem; margin: 6px 0 10px; flex: 1; }
.dish-card .dish-actions { display: flex; gap: 8px; align-items: center; }
.qty-input {
    width: 62px;
    padding: 7px 8px;
    border: 1px solid var(--border);
    border-radius: 8px;
    font-size: .9rem;
}
.stars { color: #f59e0b; letter-spacing: 2px; font-size: 1rem; }

/* -------------------------------- forms --------------------------------- */
.form-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 14px 18px; }
.form-grid .full { grid-column: 1 / -1; }
@media (max-width: 700px) { .form-grid { grid-template-columns: 1fr; } }
.form-group label { display: block; font-weight: 600; font-size: .88rem; margin-bottom: 5px; color: var(--slate); }
.form-group input,
.form-group select,
.form-group textarea {
    width: 100%;
    padding: 9px 12px;
    border: 1px solid var(--border);
    border-radius: 10px;
    font-size: .93rem;
    font-family: inherit;
    background: #fff;
    color: var(--text);
}
.form-group input:focus,
.form-group select:focus,
.form-group textarea:focus {
    outline: 2px solid var(--green);
    outline-offset: 1px;
    border-color: var(--green);
}
.form-group textarea { resize: vertical; min-height: 84px; }
.check-row { display: flex; align-items: center; gap: 8px; margin-top: 26px; }
.check-row input { width: auto; }
.form-actions { margin-top: 18px; display: flex; gap: 10px; flex-wrap: wrap; }
.filter-form {
    display: flex;
    gap: 10px;
    align-items: flex-end;
    flex-wrap: wrap;
    margin-bottom: 16px;
}
.filter-form .form-group { margin: 0; min-width: 200px; }

/* ------------------------------- totals --------------------------------- */
.totals { max-width: 420px; margin-left: auto; }
.totals .row { display: flex; justify-content: space-between; padding: 7px 0; border-bottom: 1px dashed var(--border); }
.totals .row:last-child { border-bottom: none; font-weight: 800; font-size: 1.1rem; color: var(--slate); }
.totals .grand { color: var(--green-dark); }

/* ------------------------------ empty state ----------------------------- */
.empty {
    text-align: center;
    padding: 44px 20px;
    color: var(--muted);
    background: var(--card);
    border: 1px dashed var(--border);
    border-radius: var(--radius);
}
.empty .big { font-size: 2.4rem; display: block; margin-bottom: 8px; }

/* -------------------------------- footer -------------------------------- */
.footer {
    background: var(--slate);
    color: #94a3b8;
    padding: 22px 0;
    margin-top: 30px;
    font-size: .88rem;
}
.footer p { margin: 4px 0; }
.footer .muted { color: #64748b; }

/* ------------------------------ misc bits ------------------------------- */
.avatar {
    width: 44px;
    height: 44px;
    border-radius: 50%;
    background: var(--green-light);
    color: var(--green-dark);
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-weight: 800;
    font-size: 1rem;
}
.kv { display: grid; grid-template-columns: 170px 1fr; gap: 6px 14px; font-size: .93rem; }
.kv dt { color: var(--muted); }
.kv dd { margin: 0; font-weight: 600; }
.section-title {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin: 26px 0 12px;
}
.section-title h2 { margin: 0; }
.restaurant-head {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    flex-wrap: wrap;
    border-bottom: 2px solid var(--border);
    padding-bottom: 8px;
    margin-bottom: 14px;
}

```
