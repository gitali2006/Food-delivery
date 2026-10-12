# QuickBite — MEMBER 3: Menu & Food Item Management — COMPLETE CODE

*Backend (Java / Spring Boot) + Frontend (HTML + CSS) of this module in one document.*

---

## PART A — BACKEND (Java)

### `src/main/java/com/example/fooddelivery/model/MenuItem.java`
```java
package com.example.fooddelivery.model;

import com.example.fooddelivery.common.BaseEntity;
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;

import java.math.BigDecimal;

/**
 * MEMBER 3 - MENU MODULE
 * OOP: INHERITANCE from BaseEntity, ENCAPSULATION (price can only change
 *      through methods), ASSOCIATION with Restaurant.
 */
@Entity
@Table(name = "menu_items")
public class MenuItem extends BaseEntity {

    @Column(nullable = false)
    private String name;

    @Column(length = 500)
    private String description;

    private String category;      // Starters, Main, Dessert, Drinks

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price = BigDecimal.ZERO;

    private boolean available = true;

    private boolean vegetarian = false;

    /** Many menu items belong to one restaurant. */
    @ManyToOne
    @JoinColumn(name = "restaurant_id", nullable = false)
    private Restaurant restaurant;

    public MenuItem() {
    }

    public MenuItem(String name, String category, BigDecimal price, Restaurant restaurant) {
        this.name = name;
        this.category = category;
        this.price = price;
        this.restaurant = restaurant;
    }

    @Override
    public String getDisplayName() {
        return name;
    }

    /** BUSINESS LOGIC inside the object (ENCAPSULATION): price of N portions. */
    public BigDecimal totalFor(int quantity) {
        if (quantity <= 0) {
            return BigDecimal.ZERO;
        }
        return price.multiply(BigDecimal.valueOf(quantity));
    }

    public String getAvailabilityLabel() {
        return available ? "Available" : "Sold out";
    }

    // ---------- Getters / Setters ----------

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    public String getCategory() {
        return category;
    }

    public void setCategory(String category) {
        this.category = category;
    }

    public BigDecimal getPrice() {
        return price;
    }

    public void setPrice(BigDecimal price) {
        this.price = price;
    }

    public boolean isAvailable() {
        return available;
    }

    public void setAvailable(boolean available) {
        this.available = available;
    }

    public boolean isVegetarian() {
        return vegetarian;
    }

    public void setVegetarian(boolean vegetarian) {
        this.vegetarian = vegetarian;
    }

    public Restaurant getRestaurant() {
        return restaurant;
    }

    public void setRestaurant(Restaurant restaurant) {
        this.restaurant = restaurant;
    }
}

```

### `src/main/java/com/example/fooddelivery/repository/MenuItemRepository.java`
```java
package com.example.fooddelivery.repository;

import com.example.fooddelivery.model.MenuItem;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

/** MEMBER 3 - data access for menu items. */
@Repository
public interface MenuItemRepository extends JpaRepository<MenuItem, Long> {

    List<MenuItem> findByRestaurantId(Long restaurantId);

    List<MenuItem> findByRestaurantIdAndAvailableTrue(Long restaurantId);

    List<MenuItem> findByNameContainingIgnoreCase(String name);

    List<MenuItem> findByCategoryIgnoreCase(String category);

    long countByRestaurantId(Long restaurantId);
}

```

### `src/main/java/com/example/fooddelivery/service/MenuItemService.java`
```java
package com.example.fooddelivery.service;

import com.example.fooddelivery.model.MenuItem;
import com.example.fooddelivery.model.Restaurant;

import java.util.List;

public interface MenuItemService {

    List<MenuItem> findAll();

    List<MenuItem> findByRestaurant(Long restaurantId);

    List<MenuItem> findAvailableByRestaurant(Long restaurantId);

    List<MenuItem> search(String keyword);

    List<MenuItem> findByCategory(String category);

    MenuItem findById(Long id);

    MenuItem save(MenuItem menuItem);

    void deleteById(Long id);

    void toggleAvailability(Long id);

    long count();

    /** Every dish grouped by its restaurant (used by the customer menu page). */
    java.util.Map<Restaurant, List<MenuItem>> groupByRestaurant();
}

```

### `src/main/java/com/example/fooddelivery/service/impl/MenuItemServiceImpl.java`
```java
package com.example.fooddelivery.service.impl;

import com.example.fooddelivery.common.ResourceNotFoundException;
import com.example.fooddelivery.model.MenuItem;
import com.example.fooddelivery.model.Restaurant;
import com.example.fooddelivery.repository.MenuItemRepository;
import com.example.fooddelivery.repository.RestaurantRepository;
import com.example.fooddelivery.service.MenuItemService;
import org.springframework.stereotype.Service;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

/** MEMBER 3 - business logic for the Menu module. */
@Service
public class MenuItemServiceImpl implements MenuItemService {

    private final MenuItemRepository menuItemRepository;
    private final RestaurantRepository restaurantRepository;

    public MenuItemServiceImpl(MenuItemRepository menuItemRepository,
                               RestaurantRepository restaurantRepository) {
        this.menuItemRepository = menuItemRepository;
        this.restaurantRepository = restaurantRepository;
    }

    @Override
    public List<MenuItem> findAll() {
        return menuItemRepository.findAll();
    }

    @Override
    public List<MenuItem> findByRestaurant(Long restaurantId) {
        return menuItemRepository.findByRestaurantId(restaurantId);
    }

    @Override
    public List<MenuItem> findAvailableByRestaurant(Long restaurantId) {
        return menuItemRepository.findByRestaurantIdAndAvailableTrue(restaurantId);
    }

    @Override
    public List<MenuItem> search(String keyword) {
        if (keyword == null || keyword.isBlank()) {
            return findAll();
        }
        return menuItemRepository.findByNameContainingIgnoreCase(keyword.trim());
    }

    @Override
    public List<MenuItem> findByCategory(String category) {
        if (category == null || category.isBlank()) {
            return findAll();
        }
        return menuItemRepository.findByCategoryIgnoreCase(category.trim());
    }

    @Override
    public MenuItem findById(Long id) {
        return menuItemRepository.findById(id)
                .orElseThrow(() -> ResourceNotFoundException.of("Menu item", id));
    }

    @Override
    public MenuItem save(MenuItem menuItem) {
        return menuItemRepository.save(menuItem);
    }

    @Override
    public void deleteById(Long id) {
        if (!menuItemRepository.existsById(id)) {
            throw ResourceNotFoundException.of("Menu item", id);
        }
        menuItemRepository.deleteById(id);
    }

    @Override
    public void toggleAvailability(Long id) {
        MenuItem item = findById(id);
        item.setAvailable(!item.isAvailable());
        menuItemRepository.save(item);
    }

    @Override
    public long count() {
        return menuItemRepository.count();
    }

    /** Groups menu items so the menu page can show one section per restaurant. */
    @Override
    public Map<Restaurant, List<MenuItem>> groupByRestaurant() {
        Map<Restaurant, List<MenuItem>> grouped = new LinkedHashMap<>();
        for (Restaurant restaurant : restaurantRepository.findAll()) {
            List<MenuItem> items = menuItemRepository.findByRestaurantId(restaurant.getId());
            if (!items.isEmpty()) {
                grouped.put(restaurant, items);
            }
        }
        return grouped;
    }
}

```

### `src/main/java/com/example/fooddelivery/controller/MenuItemController.java`
```java
package com.example.fooddelivery.controller;

import com.example.fooddelivery.model.MenuItem;
import com.example.fooddelivery.service.MenuItemService;
import com.example.fooddelivery.service.RestaurantService;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

import java.math.BigDecimal;

/**
 * MEMBER 3 - MENU MODULE
 * Public menu:   GET /menu          (customers browse dishes grouped by restaurant)
 * CREATE  -> GET /menu/new      + POST /menu
 * READ    -> GET /menu/manage   (+ search / filter by restaurant)
 * UPDATE  -> GET /menu/{id}/edit + POST /menu/{id}
 * DELETE  -> POST /menu/{id}/delete
 */
@Controller
@RequestMapping("/menu")
public class MenuItemController {

    private final MenuItemService menuItemService;
    private final RestaurantService restaurantService;

    public MenuItemController(MenuItemService menuItemService, RestaurantService restaurantService) {
        this.menuItemService = menuItemService;
        this.restaurantService = restaurantService;
    }

    /** Public menu page for customers. */
    @GetMapping
    public String publicMenu(Model model) {
        model.addAttribute("menuByRestaurant", menuItemService.groupByRestaurant());
        return "menu/menu";
    }

    /** Management table used by the staff. */
    @GetMapping("/manage")
    public String manage(@RequestParam(required = false) String keyword,
                         @RequestParam(required = false) Long restaurantId,
                         Model model) {
        if (restaurantId != null) {
            model.addAttribute("menuItems", menuItemService.findByRestaurant(restaurantId));
        } else {
            model.addAttribute("menuItems", menuItemService.search(keyword));
        }
        model.addAttribute("keyword", keyword);
        model.addAttribute("restaurantId", restaurantId);
        model.addAttribute("restaurants", restaurantService.findAll());
        return "menu/manage";
    }

    @GetMapping("/new")
    public String createForm(Model model) {
        model.addAttribute("menuItem", new MenuItem());
        model.addAttribute("restaurants", restaurantService.findAll());
        return "menu/form";
    }

    @PostMapping
    public String create(@ModelAttribute("menuItem") MenuItem menuItem,
                         @RequestParam(required = false) Long restaurantId,
                         RedirectAttributes redirectAttributes) {
        if (menuItem.getName() == null || menuItem.getName().isBlank() || restaurantId == null
                || menuItem.getPrice() == null) {
            redirectAttributes.addFlashAttribute("error", "Name, price and restaurant are required.");
            return "redirect:/menu/new";
        }
        if (menuItem.getPrice().compareTo(BigDecimal.ZERO) < 0) {
            redirectAttributes.addFlashAttribute("error", "Price cannot be negative.");
            return "redirect:/menu/new";
        }
        menuItem.setRestaurant(restaurantService.findById(restaurantId));
        menuItemService.save(menuItem);
        redirectAttributes.addFlashAttribute("success", "\"" + menuItem.getName() + "\" was added to the menu.");
        return "redirect:/menu/manage";
    }

    @GetMapping("/{id}/edit")
    public String editForm(@PathVariable Long id, Model model) {
        model.addAttribute("menuItem", menuItemService.findById(id));
        model.addAttribute("restaurants", restaurantService.findAll());
        return "menu/form";
    }

    @PostMapping("/{id}")
    public String update(@PathVariable Long id,
                         @ModelAttribute("menuItem") MenuItem menuItem,
                         @RequestParam(required = false) Long restaurantId,
                         RedirectAttributes redirectAttributes) {
        MenuItem existing = menuItemService.findById(id);
        existing.setName(menuItem.getName());
        existing.setDescription(menuItem.getDescription());
        existing.setCategory(menuItem.getCategory());
        existing.setPrice(menuItem.getPrice());
        existing.setVegetarian(menuItem.isVegetarian());
        existing.setAvailable(menuItem.isAvailable());
        if (restaurantId != null) {
            existing.setRestaurant(restaurantService.findById(restaurantId));
        }
        menuItemService.save(existing);
        redirectAttributes.addFlashAttribute("success", "Menu item updated.");
        return "redirect:/menu/manage";
    }

    @PostMapping("/{id}/delete")
    public String delete(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        try {
            menuItemService.deleteById(id);
            redirectAttributes.addFlashAttribute("success", "Menu item deleted.");
        } catch (Exception ex) {
            redirectAttributes.addFlashAttribute("error",
                    "This dish is used in existing orders, so it cannot be deleted. Mark it as 'sold out' instead.");
        }
        return "redirect:/menu/manage";
    }

    @PostMapping("/{id}/toggle")
    public String toggleAvailability(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        menuItemService.toggleAvailability(id);
        redirectAttributes.addFlashAttribute("success", "Availability changed.");
        return "redirect:/menu/manage";
    }
}

```


## PART B — FRONTEND (HTML templates)

### `src/main/resources/templates/menu/menu.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>QuickBite - Menu (Member 3)</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1>Our menu</h1>
            <p class="muted">Member 3 &mdash; every dish grouped under its restaurant. Add anything to your cart.</p>
        </div>
        <div class="btn-row">
            <a th:href="@{/menu/manage}" class="btn btn-outline">Manage menu (staff)</a>
            <a th:href="@{/cart}" class="btn">Go to cart</a>
        </div>
    </div>

    <div class="card empty" th:if="${#maps.isEmpty(menuByRestaurant)}">
        <span class="big">&#127858;</span>
        The menu is empty. Staff can add dishes from <a th:href="@{/menu/manage}">Manage menu</a>.
    </div>

    <section th:each="entry : ${menuByRestaurant}">
        <div class="restaurant-head">
            <div>
                <h2 style="margin:0" th:text="${entry.key.name}">Rice & Curry House</h2>
                <p class="muted" style="margin:2px 0 0;">
                    <span class="chip" th:text="${entry.key.cuisineType}">Sri Lankan</span>
                    <span class="stars" th:text="${entry.key.ratingStars}">*****</span>
                    <span class="badge" th:classappend="${entry.key.open} ? 'status-delivered' : 'status-cancelled'"
                          th:text="${entry.key.statusLabel}">Open</span>
                </p>
            </div>
            <a th:href="@{/restaurants/{id}(id=${entry.key.id})}" class="btn btn-sm btn-outline">Restaurant info</a>
        </div>

        <div class="card-grid">
            <div class="card dish-card" th:each="item : ${entry.value}">
                <div class="dish-head">
                    <h3 th:text="${item.name}">Chicken Kottu</h3>
                    <span class="price" th:text="'Rs. ' + ${#numbers.formatDecimal(item.price, 1, 'COMMA', 2, 'POINT')}">Rs. 0</span>
                </div>
                <p th:text="${item.description} ?: ''">Description</p>
                <p>
                    <span class="chip" th:text="${item.category} ?: 'Uncategorised'">Main</span>
                    <span class="chip chip-veg" th:if="${item.vegetarian}">&#127793; Veg</span>
                    <span class="badge" th:classappend="${item.available} ? 'status-delivered' : 'status-cancelled'"
                          th:text="${item.availabilityLabel}">Available</span>
                </p>
                <form class="dish-actions" th:action="@{/cart/add}" method="post">
                    <input type="hidden" name="menuItemId" th:value="${item.id}">
                    <input class="qty-input" type="number" name="quantity" value="1" min="1" th:disabled="${!item.available}">
                    <button type="submit" class="btn btn-sm btn-orange" th:disabled="${!item.available}">
                        <span th:text="${item.available} ? 'Add to cart' : 'Sold out'">Add to cart</span>
                    </button>
                </form>
            </div>
        </div>
    </section>

    <div class="alert alert-info" th:if="${currentCustomerName == null}" style="margin-bottom:20px;">
        You are not signed in &mdash; adding to the cart will ask you to <a th:href="@{/login}">sign in</a> first.
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```

### `src/main/resources/templates/menu/manage.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>QuickBite - Manage menu (Member 3)</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1>Manage menu</h1>
            <p class="muted">Member 3 &mdash; full CRUD for dishes (search, filter, edit, sold-out toggle, delete).</p>
        </div>
        <a th:href="@{/menu/new}" class="btn">+ Add dish</a>
    </div>

    <form class="filter-form" th:action="@{/menu/manage}" method="get">
        <div class="form-group">
            <label for="keyword">Search dish</label>
            <input type="text" id="keyword" name="keyword" th:value="${keyword}" placeholder="e.g. Kottu">
        </div>
        <div class="form-group">
            <label for="restaurantId">Restaurant</label>
            <select id="restaurantId" name="restaurantId">
                <option value="">All restaurants</option>
                <option th:each="r : ${restaurants}" th:value="${r.id}" th:text="${r.name}"
                        th:selected="${restaurantId != null and restaurantId == r.id}">Restaurant</option>
            </select>
        </div>
        <button type="submit" class="btn btn-slate">Filter</button>
        <a th:href="@{/menu/manage}" class="btn btn-outline">Reset</a>
    </form>

    <div class="card empty" th:if="${#lists.isEmpty(menuItems)}">
        <span class="big">&#127858;</span>
        No dishes found.
    </div>

    <div class="table-wrap" th:unless="${#lists.isEmpty(menuItems)}">
        <table class="table">
            <thead>
            <tr>
                <th>ID</th>
                <th>Dish</th>
                <th>Restaurant</th>
                <th>Category</th>
                <th>Price (Rs.)</th>
                <th>Type</th>
                <th>Status</th>
                <th>Actions</th>
            </tr>
            </thead>
            <tbody>
            <tr th:each="item : ${menuItems}">
                <td th:text="${item.id}">1</td>
                <td>
                    <strong th:text="${item.name}">Chicken Kottu</strong>
                    <div class="muted" th:text="${item.description} ?: ''" style="font-size:.82rem;">Description</div>
                </td>
                <td th:text="${item.restaurant.name}">Rice & Curry House</td>
                <td><span class="chip" th:text="${item.category} ?: '-'">Main</span></td>
                <td th:text="${#numbers.formatDecimal(item.price, 1, 'COMMA', 2, 'POINT')}">0.00</td>
                <td>
                    <span class="chip chip-veg" th:if="${item.vegetarian}">Veg</span>
                    <span class="chip" th:unless="${item.vegetarian}">Non-veg</span>
                </td>
                <td>
                    <span class="badge" th:classappend="${item.available} ? 'status-delivered' : 'status-cancelled'"
                          th:text="${item.availabilityLabel}">Available</span>
                </td>
                <td>
                    <div class="actions">
                        <a th:href="@{/menu/{id}/edit(id=${item.id})}" class="btn btn-sm btn-slate">Edit</a>
                        <form th:action="@{/menu/{id}/toggle(id=${item.id})}" method="post" class="inline-form">
                            <button type="submit" class="btn btn-sm btn-amber"
                                    th:text="${item.available} ? 'Sold out' : 'Available'">Sold out</button>
                        </form>
                        <form th:action="@{/menu/{id}/delete(id=${item.id})}" method="post" class="inline-form"
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

### `src/main/resources/templates/menu/form.html`
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title th:text="${menuItem.id == null} ? 'QuickBite - New dish' : 'QuickBite - Edit dish'">Menu item form</title>
    <link rel="stylesheet" th:href="@{/css/style.css}">
</head>
<body>
<div th:replace="~{fragments/layout :: nav}"></div>

<main class="container">
    <div class="page-head">
        <div>
            <h1 th:text="${menuItem.id == null} ? 'Add a dish' : 'Edit dish'">Dish</h1>
            <p class="muted">Member 3 &mdash; CREATE / UPDATE form</p>
        </div>
        <a th:href="@{/menu/manage}" class="btn btn-outline">&larr; Back to menu</a>
    </div>

    <div class="card">
        <form th:action="${menuItem.id == null} ? @{/menu} : @{/menu/{id}(id=${menuItem.id})}" method="post">
            <div class="form-grid">
                <div class="form-group">
                    <label for="name">Dish name *</label>
                    <input type="text" id="name" name="name" th:value="${menuItem.name}" required placeholder="Chicken Kottu">
                </div>
                <div class="form-group">
                    <label for="price">Price in Rs. *</label>
                    <input type="number" id="price" name="price" step="0.01" min="0" required
                           th:value="${menuItem.price}" placeholder="1200.00">
                </div>
                <div class="form-group">
                    <label for="category">Category</label>
                    <select id="category" name="category">
                        <option value="">-- select --</option>
                        <option value="Starters" th:selected="${menuItem.category == 'Starters'}">Starters</option>
                        <option value="Main" th:selected="${menuItem.category == 'Main'}">Main</option>
                        <option value="Dessert" th:selected="${menuItem.category == 'Dessert'}">Dessert</option>
                        <option value="Drinks" th:selected="${menuItem.category == 'Drinks'}">Drinks</option>
                    </select>
                </div>
                <div class="form-group">
                    <label for="restaurantId">Restaurant *</label>
                    <select id="restaurantId" name="restaurantId" required>
                        <option value="">-- select restaurant --</option>
                        <option th:each="r : ${restaurants}" th:value="${r.id}" th:text="${r.name}"
                                th:selected="${menuItem.restaurant != null and menuItem.restaurant.id == r.id}">Restaurant</option>
                    </select>
                </div>
                <div class="form-group full">
                    <label for="description">Description</label>
                    <textarea id="description" name="description" th:text="${menuItem.description}"
                              placeholder="What is inside this dish?"></textarea>
                </div>
                <div class="form-group check-row">
                    <input type="checkbox" id="vegetarian" name="vegetarian" th:checked="${menuItem.vegetarian}">
                    <label for="vegetarian" style="margin:0">Vegetarian</label>
                </div>
                <div class="form-group check-row">
                    <input type="checkbox" id="available" name="available"
                           th:checked="${menuItem.id == null or menuItem.available}">
                    <label for="available" style="margin:0">Available for ordering</label>
                </div>
            </div>
            <div class="form-actions">
                <button type="submit" class="btn" th:text="${menuItem.id == null} ? 'Add dish' : 'Save changes'">Save</button>
                <a th:href="@{/menu/manage}" class="btn btn-outline">Cancel</a>
            </div>
        </form>
    </div>
</main>

<div th:replace="~{fragments/layout :: footer}"></div>
</body>
</html>

```


## PART C — CSS (shared stylesheet)

All pages share `src/main/resources/static/css/style.css`. For your pages the most relevant classes are `.dish-card`, `.price`, `.chip`, `.chip-veg`, `.qty-input`. Full file below:

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
